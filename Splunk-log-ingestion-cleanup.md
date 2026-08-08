### Step 7: Tidy up log ingestion
Sample log from /var/log/pfsense.log

Command:

```
sudo grep -i suricata /var/log/pfsense.log | tail -1
```

```
2026-08-07T14:29:11+00:00 pfsense-test.hometest.arpa suricata[52824]: {"timestamp":"2026-08-07T14:29:11.064430+1000","flow_id":646996726108717,"in_iface":"vtnet1","event_type":"http","src_ip":"192.168.1.202","src_port":8333,"dest_ip":"10.20.10.201","dest_port":8000,"proto":"TCP","pkt_src":"wire/pcap","tx_id":105,"http":{"hostname":"192.168.1.215","http_port":8000,"url":"/en-US/splunkd/__raw/services/server/health/splunkd?output_mode=json","http_user_agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36","http_content_type":"application/json","http_method":"GET","protocol":"HTTP/1.1","status":200,"length":417,"request_headers":[{"name":"Connection","value":"keep-alive"},{"name":"X-Requested-With","value":"XMLHttpRequest"},{"name":"Content-Type","value":"application/x-www-form-urlencoded"},{"name":"Accept","value":"*/*"},{"name":"Accept-Encoding","value":"gzip, deflate"},{"name":"Accept-Language","value":"en-US,en;q=0.7"},{"name":"Cookie","value":"wz-api=default; splunkweb_csrf
```
Right now Splunk is receiving the entire syslog line as _raw, and you're using:

| spath input=_raw

```
index=* sourcetype=syslog source=pfsense suricata| spath input=_raw
```

That's fine for testing, but your Suricata JSON is embedded inside the syslog header:
```
Aug 7 14:29:11 pfsense-test.hometest.arpa suricata[52824]: {"timestamp":...}
                                                               ↑
                                                        JSON starts here
```

So the next step is properly parsing the Suricata EVE JSON in Splunk.

Let's test whether Splunk can extract the JSON cleanly by explicitly extracting everything after the first `{`.

Try:
```
index=* sourcetype=syslog source=pfsense suricata
| rex field=_raw "(?<suricata_json>\{.*)"
| spath input=suricata_json
| table _time event_type src_ip src_port dest_ip dest_port proto
```
This is a very good test before changing Splunk configuration.

You should get something like:
```
_time                  event_type   src_ip          src_port   dest_ip        dest_port   proto
2026-08-07 14:29:11    http         192.168.1.202   56636      10.20.10.201   8000        TCP
2026-08-07 14:29:11    fileinfo     10.20.10.201    8000       192.168.1.202   8333        TCP
```

SECOND TEST: Test an actual Suricata alert

This is the one I'd test next.

Run:
```
index=* sourcetype=syslog source=pfsense suricata
| rex field=_raw "(?<suricata_json>\{.*)"
| spath input=suricata_json
| search event_type=alert
| table _time src_ip src_port dest_ip dest_port proto alert.signature alert.category alert.severity alert.action
```

If Suricata has generated alerts, you'll get something like:
```
_time
src_ip
src_port
dest_ip
dest_port
proto
alert.signature
alert.category
alert.severity
alert.action
```

That is much more useful for your SOC/SIEM setup than just displaying the raw event.
```
Note: Suricata's EVE JSON contains nested objects.

For example:

"alert": {
    "action": "allowed",
    "signature": "Some Suricata Rule",
    "category": "Attempted Information Leak",
    "severity": 2
}

For HTTP events:

"http": {
    "hostname": "...",
    "url": "...",
    "http_user_agent": "...",
    "http_method": "GET",
    "status": 200
}
Replace alert from event_type to http in the search query

index=* sourcetype=syslog source=pfsense suricata
| rex field=_raw "(?<suricata_json>\{.*)"
| spath input=suricata_json
| search event_type=http
```

At the moment port 5140 is receiving things like systemd, sudo, etc., as well as Suricata.

Instead, the clean solution is to create a separate Splunk UDP port for Suricata, e.g. UDP 5141.

Then:
```

                 Log Collector
                       │
              ┌────────┴────────┐
              │                 │
         Other logs          Suricata
              │                 │
          UDP 5140          UDP 5141
              │                 │
              ▼                 ▼
         sourcetype=syslog  sourcetype=suricata:syslog
```
### Step 7a:
Our current data proves:
```
pfSense Suricata
       ↓
Log Collector
       ↓
10.20.10.201:5140
       ↓
Splunk
       ↓
sourcetype=syslog
```
And our JSON extraction works.

Before creating a new 5141 input, lets see our existing Splunk input definition. Then we can decide whether to:

create a second UDP input for Suricata, or
use a different Splunk routing method.

This avoids accidentally breaking the working 5140 configuration.

Run these three commands and paste the output:

This command looks for file with name "input.conf"
```
sudo find /opt/splunk/etc -name "inputs.conf" -type f -print
```

This specifically looks for sections such as:

[udp://5140]

or:

[udp://10.20.10.201:5140]
```
sudo grep -Hn -B3 -A8 "5140" $(sudo find /opt/splunk/etc -name "inputs.conf" -type f)

OR

sudo grep -Hn -B3 -A10 -E '^\[udp://' $(sudo find /opt/splunk/etc -name "inputs.conf" -type f)
```

```
sudo ss -lunp | grep 5140
```
My current /opt/splunk/etc/apps/search/local/inputs.conf is:
```
[splunktcp://9997]
connection_host = ip

[udp://5140]
connection_host = ip
host = splunktest
index = homelab
source = pfsense
sourcetype = syslog
```

So currently:

```
Log Collector
     │
     │ UDP 5140
     ▼
Splunk
     │
     ├── index = homelab
     ├── source = pfsense
     └── sourcetype = syslog
```

Let's create a separate Suricata input on UDP 5141.

1. Create the Suricata index

In Splunk Web:

Settings → Indexes → New Index

Create:

pfsense_suricata

For your lab, the defaults are fine.

You will then have:
```
homelab
    └── general logs

pfsense_suricata
    └── Suricata EVE JSON
```

2. Add UDP 5141 to Splunk

On splunk server, edit:
```
sudo nano /opt/splunk/etc/apps/search/local/inputs.conf
```

So the complete file becomes:

```
[splunktcp://9997]
connection_host = ip

[udp://5140]
connection_host = ip
host = splunktest
index = homelab
source = pfsense
sourcetype = syslog

[udp://5141]
connection_host = ip
host = splunktest
index = pfsense_suricata
source = pfsense
sourcetype = suricata:syslog
```
Save it.

`The simple method that completely avoid the above step is to create Data inputs type and the index directly from the Splunk web GUI`

On the Splunk Web GUI:

Settings > Data > Data inputs >  UDP [+Add new] > 

Port: 5141

Source name override: pfsense

Click Next.

On the Input Settings page:

Source Type: New > sourcetype = suricata:syslog

App Context: Search & Reporting

Index: pfsense_suricata

If you haven't created "pfsense_suricata" Index you can do it here by clicking `Create a new index` > Type the index name > leave the rest as default > Save


![Alt Text](/images/Splunk-syslog5141.png)


3. Restart Splunk

Run:
```
sudo /opt/splunk/bin/splunk restart
```
Then verify that Splunk is listening on both ports:
```
sudo ss -lunp | grep -E '5140|5141'
```
You should see UDP listeners for:
```
5140
5141
```

4. Now change rsyslog

Our current rsyslog has:
```
*.* @10.20.10.201:5140
```
That sends everything to 5140.

We want:
```
Suricata → 5141
Everything else → 5140
```

On log collector VM:
```
sudo nano /etc/rsyslog.conf
```

Find:
```
*.* @10.20.10.201:5140
```

Replace it with Option A:
```
if $programname == 'suricata' then {
    action(
        type="omfwd"
        target="10.20.10.201"
        port="5141"
        protocol="udp"
    )
    stop
}

*.* @10.20.10.201:5140
```

## BREAKDOWN:

if $programname == 'suricata'

Your Suricata messages look like:
```
pfsense-test.hometest.arpa suricata[52824]: {"timestamp":...}
```
so rsyslog should identify the program as:

`suricata`

`omfwd` stands for `output module forwarding`.

It tells rsyslog:

Forward this log message to another syslog server.

The `stop` means:

Once this is identified as a Suricata message, send it to 5141 and don't continue processing it through the \*.* rule.

Without stop, you would get the same Suricata event on both 5140 and 5141.


Option B: Hybrid/Legacy Syntax
If you prefer keeping the shorter syntax:

```
# Forward Suricata JSON logs to Splunk port 5141
if $programname == 'suricata' then @10.20.10.201:5141
if $programname == 'suricata' stop

# Forward remaining logs to Splunk port 5140
*.* @10.20.10.201:5140
```

5. Validate rsyslog before restarting

On the log collector:
```
sudo rsyslogd -N1
```
You want the configuration check to complete without errors.

Then:
```
sudo systemctl restart rsyslog
```
And:
```
sudo systemctl status rsyslog
```

6. Generate some Suricata traffic

You already have traffic being detected.

For example, your existing events include:

event_type = dns
event_type = http
event_type = fileinfo
event_type = tls

Generate some traffic from your test machine so that Suricata produces new EVE events.
```
curl http://testmyids.com

OR

nslookup testmyids.com

OR

nmap -PR -sn scanme.nmap.org
```
Then go to Splunk.

7. Search the new Suricata input

Run:
```
index=pfsense_suricata
```

You should now see:

host = splunktest
source = pfsense
sourcetype = suricata:syslog

That is the result we're looking for.

Then:
```
index=pfsense_suricata sourcetype=suricata:syslog
```

8. Now test your JSON extraction

Your previous working search was:
```
index=* sourcetype=syslog source=pfsense suricata
| rex field=_raw "(?<suricata_json>\{.*)"
| spath input=suricata_json
| table _time event_type src_ip src_port dest_ip dest_port proto
```

Change it to:
```
index=pfsense_suricata sourcetype=suricata:syslog
| rex field=_raw "(?<suricata_json>\{.*)"
| spath input=suricata_json
| table _time event_type src_ip src_port dest_ip dest_port proto
```
You should get the same results:
```
event_type    src_ip          src_port    dest_ip        dest_port    proto
dns           10.20.10.201    55316       10.20.10.1    53           UDP
http          192.168.1.202   8333        10.20.10.201   8000         TCP
fileinfo      10.20.10.201    8000        192.168.1.202  1621         TCP
tls           10.20.10.201    52710       44.235.228.7   443          TCP
```

9. The important part: make JSON extraction automatic


Before Starting this step first verify that:

`index=pfsense_suricata`

returns events with: `sourcetype=suricata:syslog`

Once that works, we'll configure: `props.conf`

for: `[suricata:syslog]`

so that Splunk automatically understands the EVE JSON.

Then you won't need:
```
| rex field=_raw "(?<suricata_json>\{.*)"
| spath input=suricata_json
```
every time.

Eventually you'll be able to do:
```
index=pfsense_suricata event_type=alert
```

and Splunk will already know about fields such as:
```
event_type
src_ip
src_port
dest_ip
dest_port
proto
flow_id
in_iface
alert.*
http.*
dns.*
tls.*
fileinfo.*
```






To make Splunk automatically extract all nested fields from Suricata EVE JSON logs on ingestion, you need to set the **`KV_MODE = json`** property or assign the built-in **`sourcetype = _json`** to that data stream.

Here are the two ways to set this up:

### Option 1: Set Sourcetype to `_json` in Splunk Web (Easiest)

If you created a dedicated UDP input (e.g., port `5141`) for Suricata logs:

1. In Splunk Web, navigate to **Settings** $\rightarrow$ **Data Inputs** $\rightarrow$ **UDP**.
2. Click on port **`5141`** (or whichever port receives your Suricata traffic).
3. Change **Source Type** to **`_json`** (under the `Structured` category).
4. Save the input.

> **Why this works:** Splunk’s default `_json` sourcetype automatically inspects raw event strings starting with `{` and extracts every single top-level and nested key (`src_ip`, `dest_port`, `http.url`, `fileinfo.filename`, etc.) into searchable fields instantly upon search execution.
> 

### Option 2: Define Custom `props.conf` (Best for Custom Sourcetypes) [RECOMMENDED]

If your data input is configured with a custom sourcetype like `sourcetype = pfsense_suricata` or `pfsense:suricata`, tell Splunk to use JSON key-value extraction for that specific sourcetype name:

1. On your Splunk instance, edit or create `$SPLUNK_HOME/etc/apps/search/local/props.conf` (or `$SPLUNK_HOME/etc/system/local/props.conf`).
2. Add the following block:


```
[suricata:syslog]
KV_MODE = json
SHOULD_LINEMERGE = false
LINE_BREAKER = ([\r\n]+)
SEDCMD-strip_syslog = s/^[^{]*({.*})$/$1/
```
>OR USE A SIMPLER VERSION OF THE SED COMMAND [RECOMMENDED]

>THE BELOW SED COMMAND SIMPLY REMOVES EVERYTHING BEFORE THE FIRST {

```
[suricata:syslog]
SHOULD_LINEMERGE = false
LINE_BREAKER = ([\r\n]+)
SEDCMD-strip_syslog = s/^[^{]*//
KV_MODE = json
```

```
Here is the line-by-line breakdown of that Splunk props.conf configuration and what each parameter does for your Suricata logs:

[suricata:syslog]
What it is: The sourcetype header.

What it does: Tells Splunk that every setting listed under this section applies only to incoming events whose sourcetype is named suricata:syslog.

KV_MODE = json
What it is: Key-Value Extraction Mode.

What it does: Enables automatic JSON parsing at search time.

Why you need it: Splunk will look at the JSON payload in your Suricata logs and automatically extract every key-value pair into searchable fields (e.g., src_ip, dest_ip, event_type, proto, alert.signature) without requiring manual spath or rex commands in your queries.

SHOULD_LINEMERGE = false
What it is: Multi-line merge toggle.

What it does: Forces Splunk to treat every single line as its own separate event, disabling Splunk's default behavior of trying to group related lines together into multi-line events.

Why you need it: Suricata outputs EVE logs where one line = one complete JSON object. Setting this to false significantly increases indexing speed and prevents log corruption or line stitching.

LINE_BREAKER = ([\r\n]+)
What it is: Event Boundary Regular Expression.

What it does: Defines the exact character sequence that separates one event from the next—in this case, one or more newline (\n) or carriage return (\r) characters.

Why you need it: Paired with SHOULD_LINEMERGE = false, this ensures high-performance event boundary detection on raw log streams arriving over UDP.

SEDCMD-strip_syslog = s/^[^{]*({.*})$/$1/
What it is: A stream editor (sed) regex transformation applied during ingestion.

What it does: Finds everything from the start of the line up to the first { character and discards it, leaving only the {...} JSON structure.

Why you need it: When Suricata routes through rsyslog over UDP, rsyslog often prepends a syslog header to the front of the line (e.g., <13>Aug  8 02:50:00 logcollector suricata: {"timestamp": ...}).

If a syslog header exists before the opening {, standard JSON parsers may fail or treat the whole log as plain text. This line strips away <13>Aug  8 02:50:00 logcollector suricata:  before it hits the indexer so Splunk receives a clean, raw JSON string starting with {.

```
### Sed Command Breakdown: `s/^[^{]*({.*})$/$1/`

This is a stream editor substitute command using regular expression capture groups:

- **`s/`**: Initiates a **substitute** operation in the format `s/pattern/replacement/`.
- **`^`**: Asserts the match must start at the **very beginning of the line**.
- **`[^{]*`**: Matches zero or more consecutive characters that are **NOT** an opening curly brace (`{`). This consumes the entire syslog header (`2026-08-08T14:16:58+00:00 pfsense-test... suricata[19941]:` ).
- **`({.*})`**: **Capture Group 1** (enclosed in parentheses `()`):
    - **`{`**: Matches the first opening curly brace.
    - **`.*`**: Matches all characters following the brace up to the end of the line.
    - **`}`**: Matches the final closing brace of the JSON payload.
- **`$`**: Asserts the match extends to the **end of the line**.
- **`/$1/`**: Replaces the **entire line** with only the contents captured inside Group 1 (`$1`), effectively deleting everything preceding the opening brace `{`.s

3. Check the configuration

After saving props.conf:
```
sudo /opt/splunk/bin/splunk btool props list suricata:syslog --debug
```

Then restart Splunk:
```
sudo /opt/splunk/bin/splunk restart
```

4. Test in Splunk

Run:
```
index=pfsense_suricata sourcetype="suricata:syslog"
| table _time event_type src_ip src_port dest_ip dest_port proto
```
If it works, you'll get something like:
```
_time                 event_type  src_ip          src_port  dest_ip        dest_port  proto
2026-08-08 12:00:00   http        192.168.1.202   8333      10.20.10.201   8000       TCP
2026-08-08 12:00:01   fileinfo    10.20.10.201    8000      192.168.1.202   8333       TCP
```
And importantly, nested Suricata data such as:
```
"http": {
    "hostname": "...",
    "url": "...",
    "http_method": "GET"
}
```
can also be extracted.

Also test a specific field

```
index=pfsense_suricata sourcetype="suricata:syslog"
| stats count by event_type
```
- If you click the ">" icon on the events table, you will see all the field value pair extracted from the raw data!
- You should see fields like `src_ip`, `dest_ip`, `event_type`, `proto`, `http.hostname`, and `http.url` listed in the **Interesting Fields** sidebar on the left without needing any `spath` or `rex` commands in your search query.


### Troubleshooting

To see if Suricata logs are being populated on the pfsense.log file and if logs are being sent to Splunk Manager VM

ON log collector VM
Run:
```
sudo tail -n 0 -f /var/log/pfsense.log
```
AND
```
sudo tcpdump -ni any host 10.20.10.201 and udp port 5141
```
* 10.20.10.201 = IP addresss of Splunk Manager VM


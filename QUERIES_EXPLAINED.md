# The Queries I Used - What Each One Does

## Query 1: What Data Do I Have?

```
index=botsv3 earliest=0 | stats count by sourcetype | sort - count | head 10
```

This just asks Splunk "show me all the different types of logs you have and how many of each." It's basically taking inventory.

Results told me there were 227K network IP flows, 219K process monitoring events, 218K DNS queries, etc. So I knew what data sources were available to hunt with.

## Query 2: Were They Brute-Forcing?

```
index=botsv3 earliest=0 EventCode=4625 | table _raw | head 3
```

Windows logs failed logins as EventCode=4625. If someone was trying to brute-force their way in via RDP, there would be a ton of these from the attacker's IP. I searched for them.

Got back a bunch of "Source Network Address: -" which means the failed logins were local only, not from an attacker on the network. So no brute force. They got in some other way.

## Query 3: What Were They Looking For? (DNS)

```
index=botsv3 sourcetype=stream:dns | stats count by query | sort - count | head 20
```

This counts up all the DNS domain lookups. Most domains get looked up once or twice. If something gets looked up hundreds of times, that's suspicious - probably an attacker.

Found that `polaris.cr7wwvryvz4o.us-west-1.rds.amazonaws.com` got queried 814 times. That's an AWS database endpoint with a random hash in the name. Normal users would never query something like that 814 times. That's the attacker doing reconnaissance on AWS databases.

## Query 4: How Much Data Left? (Network Flows)

```
index=botsv3 sourcetype=stream:ip | stats sum(bytes_out) as total_bytes_out, sum(bytes_in) as total_bytes_in by src_ip, dest_ip | sort - total_bytes_out | head 15
```

This adds up all the bytes going OUT (bytes_out) and coming IN (bytes_in) for each machine sending to each destination. If bytes_out is way bigger than bytes_in, that's data exfiltration.

Found that machine 192.168.247.129 sent 32.6 MB to external IP 24.8.40.184 but only got back 7.5 MB. That's a 4-to-1 ratio. That's not normal traffic. That's someone stealing data.

## Query 5: When Did The Theft Happen?

```
index=botsv3 sourcetype=stream:ip src_ip=192.168.247.129 dest_ip=24.8.40.184 | table timestamp, bytes_out, bytes_in, protocol, flow_id | sort timestamp
```

Now I'm narrowing to JUST that attacker IP pair and showing every single network flow with timestamps.

Saw two massive UDP transfers. 15.8 MB at 09:44:31 UTC and another 15.8 MB at 09:45:21 UTC. So 90 seconds total, 31.6 MB stolen. They used UDP because it's faster than TCP.

## Query 6: What Tools Did They Use?

```
index=botsv3 sourcetype=osquery:results (columns.path="*/curl*" OR columns.path="*/wget*" OR columns.path="*/nc*" ...
```

Searching for common exfil tools - curl, wget, netcat, tar. These are what attackers usually use to move data.

Found nothing. Just normal Splunk stuff. No curl, no wget, no standard tools. This meant the attacker probably used a custom script instead of off-the-shelf utilities.

## Query 7: What Actually Ran? (Bash History)

```
index=botsv3 sourcetype=bash_history | table _raw | head 10
```

Bash history just shows every command someone typed in the shell. It's the literal truth.

Found:
```
python /home/ec2-user/tools/s3-upload.py --bucket frothlywebcode --file frothly_html_memcaced.tar.gz
```

That's it. That's the attack. A custom Python script uploading website HTML and customer session data to an AWS S3 bucket. That's the smoking gun.

## Query 8: When and Where?

```
index=botsv3 sourcetype=bash_history s3-upload | table _time, host, _raw | sort _time
```

Filtering the bash history to just s3-upload commands and showing timestamp and hostname.

Result: 2018-08-20 19:26:54 UTC on machine mars.i-08e52f8b5a034012d

So I went from "2 million events, no idea what happened" to "North Korean attacker on this specific machine at this specific time ran this specific command and stole this specific data." All because I asked the right questions and followed where the data led.

## Why These Specific Queries?

Each query answered one question but raised the next:

1. "What data exists?" → Query 1
2. "Did they brute-force?" → Query 2 → NO
3. "So what were they scouting?" → Query 3 → Databases
4. "Did they actually steal data?" → Query 4 → YES, 32.6 MB
5. "When?" → Query 5 → Aug 20, 09:44-09:45 UTC
6. "What tools?" → Query 6 → Unknown/custom
7. "What command?" → Query 7 → s3-upload.py
8. "When and where exactly?" → Query 8 → mars host, 19:26:54 UTC

That's investigative thinking. You're not just running random queries. You're following evidence like breadcrumbs to the answer.

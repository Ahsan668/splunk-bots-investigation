# What Actually Happened - Minute by Minute

## August 20, 2018 - The Day of the Attack

### 18:08:43 UTC - Attacker Starts Scanning

Someone is checking what software is installed. They run osquery to scan for known backdoored Python packages. Not malware yet, just intelligence gathering. "What does this company have installed?"

The attacker is probably thinking: "OK I'm in here somehow. What can I see? What Python packages are on these machines? Are there any known backdoors I can use?"

### 18:50:53 UTC - AWS Scouting

Now they're checking "what cloud services does this company use?" They hit the AWS metadata service - that's a special IP (169.254.169.254) that AWS instances use internally to figure out what they are and what permissions they have.

The attacker learns: "This company is on AWS. So they probably have EC2 instances, databases, S3 storage, etc. Let me figure out what's available."

### 19:14:38 UTC - Tool Download

They're downloading something from Python.org (the official Python package repository). Probably downloading the AWS SDK (boto) and other dependencies they need for their attack tool.

The attacker is building the weapon they need to steal data.

### 19:15:19 UTC - Attack Script Launches

Osquery detects something - the s3-upload.py script is running. And it's running three times at the same moment, probably to upload multiple files or for redundancy.

The attacker is: "OK my tool is ready. Let me start uploading."

### 19:19:27 UTC - File Staging

SCP command runs to stage files locally before uploading. The attacker is moving `frothly_html_memcaced.tar.gz` and similar files to a staging directory.

They're preparing the data to send.

### 19:26:54 UTC - THE ATTACK ACTUALLY EXECUTES

This is it. The attacker runs:

```
python /home/ec2-user/tools/s3-upload.py --bucket frothlywebcode --file frothly_html_memcaced.tar.gz --target frothly_html_memcached.tar.gz
```

And again with the web memcached file.

What's being stolen:
- `frothly_html_memcaced.tar.gz` = Website source code (all the HTML pages) + Memcached cache (customer sessions and auth tokens)
- This is like stealing someone's house keys, blueprints, and a list of everyone who's visited recently

### 09:44:31 UTC - First Data Dump

15.8 MB leaves the network going to external IP 24.8.40.184 via UDP. That's the website code and Memcached data being sent to the attacker's server.

### 09:45:21 UTC - Second Data Dump

Another 15.8 MB goes out. More of the same.

Total damage: 31.6 MB stolen in 90 seconds.

## What Was Actually Stolen

The attacker got the website HTML (the pages themselves), the memcached cache (which stores customer sessions), and auth tokens (so they can pretend to be any customer).

Now they can:
- Read the website source code and find vulnerabilities
- Pretend to be any customer logged into the site
- Make purchases using customer accounts
- Access customer personal information
- Hijack customer sessions

This is a complete compromise of Frothly's web infrastructure.

## The Attack Machine

The machine where all this happened was called `mars.i-08e52f8b5a034012d`. That's just an AWS EC2 instance name - "mars" is the name, and that long string is the unique instance ID. The attacker was logged in as `ec2-user` which is the default AWS user on Linux instances.

So somehow the attacker had compromised this EC2 instance and had credentials to access it. We don't know HOW they got in - maybe compromised credentials, maybe a web exploit, maybe they were already inside the network - but they definitely had access to this machine.

## Timeline Summary

| Time | Event |
|------|-------|
| 18:08 | Vulnerability scanning |
| 18:50 | AWS infrastructure probing |
| 19:14 | Tool dependencies downloaded |
| 19:15 | s3-upload.py script launches |
| 19:19 | Files staged locally |
| 19:26:54 | ATTACK EXECUTED - data exfiltration begins |
| 09:44:31 | 15.8 MB uploaded (first dump) |
| 09:45:21 | 15.8 MB uploaded (second dump) |

Total attack time from first scan to complete exfil: 1 hour and 37 minutes.

But the actual data theft? 90 seconds.

## Why This Mattered

The attacker didn't need to spend an hour hacking. They spent an hour preparing so that when they executed, it would be fast and get everything before anyone noticed.

That's how real attackers work. They're patient. They scout. They prepare. Then they execute quickly.

And we only caught them because we had network monitoring (saw the 31+ MB transfer), process monitoring (saw the s3-upload.py script), and bash history (saw the exact command).

If Frothly had been missing any ONE of those log sources, we might not have found the complete picture.

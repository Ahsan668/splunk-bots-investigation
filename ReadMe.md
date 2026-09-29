\# Splunk BOTS v3 Investigation: Frothly Breach



\## Scenario

North Korean group Taedonggang attacked Frothly (beer company) on Aug 20, 2018.



\## What We Found

\- Attacker accessed machine: mars.i-08e52f8b5a034012d

\- Stole: Frothly website HTML + Memcached data (31.6 MB)

\- Uploaded to: AWS S3 bucket called frothlywebcode

\- Method: Custom s3-upload.py script at 19:26:54 UTC



\## Key Queries

1\. DNS Recon: Found attacker querying AWS RDS endpoint 814 times

2\. Data Flow: Identified 32.6 MB exfil from 192.168.247.129 to 24.8.40.184

3\. Command: s3-upload.py uploading website + memcache tar.gz files



\## Timeline

18:08 - Vuln scan

19:15 - s3-upload.py detected

19:26 - Exfiltration executed


# SQL Injection on DVWA 

## What to monitor on the machine 

### 1) Web server access logs 

On DVWA-like stacks (Apache + PHP), the main evidence is the HTTP requests in:

* `/var/log/apache2/access.log`
* `/var/log/apache2/error.log`

Indicators that often appear in the URL query string:

* single quotes and comments: `'`, `--`, `#`
* SQL keywords: `union select`, `information_schema`, `concat`, `substring`
* boolean logic patterns: `or 1=1`, `and 1=0`
* time-based patterns: `sleep(5)`

Typical symptom:

* repeated hits to the same vulnerable endpoint with slightly different payloads (character-by-character extraction for blind SQLi).

### 2) Database logs (when enabled)

Depending on configuration, MySQL/MariaDB logs can show suspicious queries:

* General query log (very noisy)
* Slow query log (useful for time-based injection because `sleep()` creates long queries)

Time-based injection tends to create:

* unusual query durations
* spikes in slow queries even with low traffic

### 3) Application errors and anomalies

SQLi attempts often cause:

* HTTP 500 errors (if error-based SQLi triggers exceptions)
* abnormal response sizes (UNION extraction returning lots of rows)
* response-time anomalies (time-based SQLi)


### 4) Tool fingerprinting (sqlmap)

Automated exploitation with sqlmap often leaves obvious traces:

* very consistent request patterns
* repeated requests to the same parameter
* common user-agent strings (often includes "sqlmap" unless changed)


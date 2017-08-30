---
title: Using pgBagder and logsaw for scheduled reports
author: gilles
date: 2012-08-31 23:51:19 +0200
---
About creating reports with just pgbadger 2.x on a weekly basis, like in your example using crontab:

```
10 2 1-7 * 7  pgbadger -o /path/to/report-$(date +%Y%m%d).html -l /path/to/pgbadger-incremental.last `find /var/log/postgresql/postgresql.* -mtime -7`
```
---
title: More work and thoughts on index bloat estimation query
author: Michael Banck
date: 2014-07-01 21:34:03 +0200
---
Your query (still) requires superuser privileges due to using pg_statistic. This is not really great for monitoring queries, and can be easily avoided, as shown here (based on Josh Berkus' version of your earlier query): [https://gist.github.com/mbanck/9976015/revisions]().
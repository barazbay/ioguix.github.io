---
title: Normalizing queries for pg_stat_statements < 9.2
author: ioguix
date: 2012-08-07 11:07:01 +0200
---
@Peter Geoghegan: Ok, I understood this time !

Well, thank you to point this out, but this problem is inherent to pg_stat_statements anyway. The only thing I can think about to somehow limit this problem is to have a higher pg_stat_statements.max ...
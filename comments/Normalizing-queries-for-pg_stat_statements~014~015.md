---
title: Normalizing queries for pg_stat_statements < 9.2
author: ioguix
date: 2012-08-06 23:38:40 +0200
---
@Peter Geoghegan: About backporting, my goal here was to give an easy trick on &lt;9.2 without patching/compiling code, just  pure and easy SQL.


I'm not sure to understand the first part of your comment. If you talk about current implementation of pg_stat_statement in 9.2+, I'm not sure regular expression are the best way to normalize queries on the fly. I believe a tokenizer would be very useful in PostgreSQL core, and discussed it in past. Normalizing would just be another useful feature from it.
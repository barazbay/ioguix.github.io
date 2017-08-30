---
title: Normalizing queries for pg_stat_statements < 9.2
author: Peter Geoghegan
date: 2012-08-07 00:53:33 +0200
---
There is a low-level interface to the authoritative scanner (tokenizer) used for lexing SQL in Postgres. That's how I got 9.2's pg_stat_statements to produce a normalised query string after the parser stage (note that that's totally orthogonal to how the query fingerprinting works).  It's been used by plpgsql for a long time.


If you don't understand my remarks about the problem with what you've done here, run pgbench in just the same way against 9.1, but with -M prepared. Then run without -M prepared, and apply your technique. What you've done here is going to fill the internal pg_stat_statements table too fast to be useful, because there will be a distinct entry for every set of constants for every query. Minor variations will be evicted, and query execution costs will be dramatically underestimated. That's clearly what's happened in your example.
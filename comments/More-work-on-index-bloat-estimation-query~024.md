---
title: More work and thoughts on index bloat estimation query
author: Joe Abbate
date: 2014-06-25 08:04:55 -0400
---

Re: name being a fixed length type

See the bottom of this page: [http://www.postgresql.org/docs/9.3/static/datatype-character.html]() 

I think 'name' being fixed length is a tradeoff of space vs. time in the system catalogs.  It gives predictable row layouts in the critical catalogs, for initialization, recovery, etc.
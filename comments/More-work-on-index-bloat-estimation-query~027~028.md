---
title: More work and thoughts on index bloat estimation query
author: Jehan-Guillaume (ioguix) de Rorthais
date: 2014-07-02 13:08 +0200
---
Hi Michael,

That's true. However, note that using @pg_stats@, you must make sure you are using a supervision role that is always granted to read all tables. If supervision role can not read some of the tables, the query just ignore them.

I updated my query (here and on github) based on your diff and just added a comment about that. See: [https://gist.github.com/ioguix/c29d5790b8b93bf81c27/revisions]() 

Thanks,
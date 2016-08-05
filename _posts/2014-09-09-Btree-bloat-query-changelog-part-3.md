---
layout: post
author: ioguix
title: Btree bloat query changelog - part 3
date: 2014-09-09 17:00:00 +0200
tags: [bloat, postgresql]
category: postgresql
---

##Changelog

Here are some fresh news about my previous work (see
[part 1]({%post_url 2014-03-28-Playing-with-indexes-and-better-bloat-estimate%})
and [part 2]({%post_url 2014-06-24-More-work-on-index-bloat-estimation-query%}))
on a better Btree bloat query:

* add field `is_na` to filter out indexes we can not estimate the bloat with
  "accuracy" (currently only indexes referencing a fields using the `name` type).
* better estimation for variable length types (`varchar`, `text`, `bytea`, ...).
  Thequery now try to consider their header part
* Three different queries depending on the PostgreSQL version:
  * from 7.4 to 8.1: [https://gist.github.com/ioguix/5f60e24a77828078ff5f]()
  * for 8.2: [https://gist.github.com/ioguix/0675875e2780b362ef28]()
  * for 8.3 and more: [https://gist.github.com/ioguix/c29d5790b8b93bf81c27]()
* ignore non-valid indexes for 8.2 and more

##Known issue

While working on table bloat (I will blog about that very soon), I found a
large deviation on statistics on array types. I'm not sure how to handle
correctly these animals' header yet.

Cheers and happy monitoring!
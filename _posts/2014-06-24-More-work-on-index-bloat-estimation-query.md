---
layout: post
author: ioguix
title: More work and thoughts on index bloat estimation query
date: 2014-06-24 23:21:47 +0200
tags: [bloat, postgresql]
category: postgresql
---
A few weeks ago, I [published]( {%post_url 2014-03-28-Playing-with-indexes-and-better-bloat-estimate%} )
a query to estimate index bloat.  Since then, I went back on this query a few
times to fix some drawbacks:

* making it compatible from PostgreSQL 7.4 to latest releases
* restrict to B-tree index only
* remove psql variables (sorry for code readability and documentation)
* improve 64 vs. 32 bits detection

This last one is actually far from perfect.  Very bad estimation could arise if
the query is wrong about this size of pointers.

##New version

Here is the gist of the new version of this query if you want to comment/patch:
[https://gist.github.com/ioguix/c29d5790b8b93bf81c27](https://gist.github.com/ioguix/c29d5790b8b93bf81c27)

And the code itself:

{% highlight sql %}
-- WARNING: executed with a non-superuser role, the query inspect only index on tables you are granted to read.
SELECT current_database(), nspname AS schemaname, c.relname AS tablename, indexname, bs*(sub.relpages)::bigint AS real_size,
  bs*otta::bigint as estimated_size,
  bs*(sub.relpages-otta)::bigint                                     AS bloat_size,
  bs*(sub.relpages-otta)::bigint * 100 / (bs*(sub.relpages)::bigint) AS bloat_ratio
  -- , index_tuple_hdr_bm, maxalign, pagehdr, nulldatawidth, nulldatahdrwidth, datawidth, sub.reltuples, sub.relpages -- (DEBUG INFO)
FROM (
  SELECT bs, nspname, table_oid, indexname, relpages, coalesce(
      ceil((reltuples*(4+nulldatahdrwidth))/(bs-pagehdr::float)) + 1, 0 -- ItemIdData size + computed avg size of a tuple (nulldatahdrwidth)
    ) AS otta
    -- , index_tuple_hdr_bm, maxalign, pagehdr, nulldatawidth, nulldatahdrwidth, datawidth, reltuples -- (DEBUG INFO)
  FROM (
    SELECT maxalign, bs, nspname, relname AS indexname, reltuples, relpages, relam, table_oid,
      ( index_tuple_hdr_bm +
          maxalign - CASE -- Add padding to the index tuple header to align on MAXALIGN
            WHEN index_tuple_hdr_bm%maxalign = 0 THEN maxalign
            ELSE index_tuple_hdr_bm%maxalign
          END
        + nulldatawidth + maxalign - CASE -- Add padding to the data to align on MAXALIGN
            WHEN nulldatawidth = 0 THEN 0
            WHEN nulldatawidth::integer%maxalign = 0 THEN maxalign
            ELSE nulldatawidth::integer%maxalign
          END
      )::numeric AS nulldatahdrwidth, pagehdr
      -- , index_tuple_hdr_bm, nulldatawidth, datawidth -- (DEBUG INFO)
    FROM (
      SELECT
        i.nspname, i.relname, i.reltuples, i.relpages, i.relam, a.attrelid AS table_oid,
        CASE cluster_version.v > 7
            WHEN true THEN current_setting('block_size')::numeric
            ELSE 8192::numeric
        END AS bs,
        CASE  -- MAXALIGN: 4 on 32bits, 8 on 64bits (and mingw32 ?)
          WHEN version() ~ 'mingw32' OR version() ~ '64-bit|x86_64|ppc64|ia64|amd64' THEN 8
          ELSE 4
        END AS maxalign,
        /* per page header, fixed size: 20 for 7.X, 24 for others */
        CASE WHEN cluster_version.v > 7
          THEN 24
          ELSE 20
        END AS pagehdr,
        /* per tuple header: add IndexAttributeBitMapData if some cols are null-able */
        CASE WHEN max(coalesce(s.null_frac,0)) = 0
          THEN 2 -- IndexTupleData size
          ELSE  2 + (( 32 + 8 - 1 ) / 8) -- IndexTupleData size + IndexAttributeBitMapData size ( max num filed per index + 8 - 1 /8)
        END AS index_tuple_hdr_bm,
        /* data len: we remove null values save space using it fractionnal part from stats */
        sum( (1-coalesce(s.null_frac, 0)) * coalesce(s.avg_width, 1024) ) AS nulldatawidth
        -- , sum( s.stawidth ) AS datawidth -- (DEBUG INFO)
      FROM pg_attribute AS a
        JOIN pg_stats AS s ON (quote_ident(s.schemaname) || '.' || quote_ident(s.tablename))::regclass=a.attrelid AND s.attname = a.attname
        JOIN (
          SELECT nspname, relname, reltuples, relpages, indrelid, relam,
            string_to_array(pg_catalog.textin(pg_catalog.int2vectorout(indkey)), ' ')::smallint[] AS attnum
          FROM pg_index
            JOIN pg_class ON pg_class.oid=pg_index.indexrelid
            JOIN pg_namespace ON pg_namespace.oid = pg_class.relnamespace
        ) AS i ON i.indrelid = a.attrelid AND a.attnum = ANY (i.attnum),
        ( SELECT substring(current_setting('server_version') FROM '#"[0-9]+#"%' FOR '#')::integer ) AS cluster_version(v)
      WHERE a.attnum > 0
      GROUP BY 1, 2, 3, 4, 5, 6, 7, 8, 9, cluster_version.v
    ) AS s1
  ) AS s2
    JOIN pg_am am ON s2.relam = am.oid WHERE am.amname = 'btree'
) as sub
JOIN pg_class c ON c.oid=sub.table_oid
WHERE sub.relpages > 2
ORDER BY 2,3,4;
{% endhighlight %}

I left commented some debug code to help diagnosing bad estimations.

__**Update**__ Following the [comment of Michael Banck](#comment-027), I
updated the query and added a warning on top of the query. The query does not
require to be superuser anymore, but you have to make sure the role you are
using is able to access all your precious tables and indexes!

##Known bug

While testing the query, I found a weird bug where negative bloats show up on
some system indexes.  As instance:

{% highlight psql %}
postgres@pagila=# \i btree_bloat.sql
...
-[ RECORD 4 ]----+----------------------------------------------------
current_database | pagila
schemaname       | pg_catalog
tablename        | pg_attribute
indexname        | pg_attribute_relid_attnam_index
real_size        | 122880
estimated_size   | 262144
bloat_size       | -139264
bloat_ratio      | -113.3333333333333333
...
{% endhighlight %}

After hunting for some time on this, I found that this was related to the
`name` type.  This type has a fixed size of 64 bytes in stats, but
they become a simple cstring in index with a variable length depending on real
string values!

{% highlight psql %}
postgres@pagila=# \d pg_attribute_relid_attnam_index
Index "pg_catalog.pg_attribute_relid_attnam_index"
  Column  |  Type   | Definition 
----------+---------+------------
 attrelid | oid     | attrelid
 attname  | cstring | attname
unique, btree, for table "pg_catalog.pg_attribute"

postgres@pagila=# \d pg_catalog.pg_attribute
    Table "pg_catalog.pg_attribute"
    Column     |   Type    | Modifiers 
---------------+-----------+-----------
 attrelid      | oid       | not null
 attname       | name      | not null
[...]
Indexes:
    "pg_attribute_relid_attnam_index" UNIQUE, btree (attrelid, attname)
    "pg_attribute_relid_attnum_index" UNIQUE, btree (attrelid, attnum)


postgres@pagila=# SELECT pg_column_size(attname), avg(length(attname)) FROM pg_catalog.pg_attribute GROUP BY 1;
 pg_column_size |        avg         
----------------+--------------------
             64 | 9.5131713992473486
{% endhighlight %}

As the query rely on theses stats to compute the estimated size of the index
depending on indexed fields and number of lines in the table, this difference
with the real value size in indexes make the stats completely wrong.

I'm not sure how I should handle this.  Maybe by defining some incompatible
types with this query?  Moreover, I am curious about why `name` is a
fixed length type...

As usual, any feedback, help or answers on these two last posts is appreciated :)
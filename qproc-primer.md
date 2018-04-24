
# Primer on classic database execution 


Query Execution 

* Disk is slow for seeks, fast for scans
* Page-oriented layout, buffer pool, 
* What's stored in a page -- page header + NRows * (16byte header + data)
* Execution for select query no joins
* Execution for join query

Query compilation into plans

* Operators as atomic units
* Logical vs physical operators
  * use python code for filter and NL/Hash join
* Logical and physical plans
  * pull-based execution
* SQL to logical plan

Query Optimization 

* Logical rewriting rules
  * constant folding
  * operator pushdown
  * simple cardinality example for operator pushdown
  * Vacuum and explain
* Join optimization
  * two table example
  * 3 table example
* Interpreted vs Compiled
  * Python vs C analogy


Traditional optimizations

* Indexes
* Partitioning
  * if xacts touch well deined slices or subsets of data (e.g., bank users only access their own banking data) then
    partition the data along those dimensions.  Then different xacts can run in parallel within each partition
  * Example: FB is known to partition their user data by user.
  * Techniques: round robin, hash, range, lookup table
  * Downsides -- what if no natural partitioning?  :(
* Intermediate results/Cubes
  * Consider the following queries

        Q1: SELECT month, SUM(sales)
        Q2: SELECT month, SUM(sales) WHERE month = 'Feb'
        Q3: SELECT month, AVG(sales)
        Q4: SELECT month, region, SUM(sales)
        Q5: SELECT month, SUM(sales) WHERE region = "US"

  * No matter the dataset size, there's only three attributes touched.  Precompute intermediate results:
  * 1 dimensional data structure on Month can answer Q1 and Q2, but not the rest:

        Jan --> SUM(sales)
        Feb --> SUM(sales)
        Mar --> SUM(sales)

  * Can answer Q1-Q3, but not Q4,5

        Jan --> {sum:, count: }
        Feb --> {sum:, count: }
        Mar --> {sum:, count: }

  * Build a two dimensional data structure on Month & Region.  Can answer Q1-Q5

                  US         Eur          Asia
        Jan  {sum,count}  {sum,count}  {sum,count}
        Feb  {sum,count}  {sum,count}  {sum,count}
        Mar  {sum,count}  {sum,count}  {sum,count}


  * Attribute hierarchies 
    * roll up goes up a hierarchy: `SELECT month, SUM` to `SELECT year, SUM`
    * drill down, slice, dice

  
                        US         Eur          Asia
        2010   Jan  {sum,count}  {sum,count}  {sum,count}
        2010   Feb  {sum,count}  {sum,count}  {sum,count}
        2010   Mar  {sum,count}  {sum,count}  {sum,count}

  * When does this work?  For algebraic and distributive operators

        for a function f, there exists a function g such that
          f(set) = g(f(partition1), ... f(partitionN))

        COUNT({0,1,2,...10}) = SUM(COUNT({0}), COUNT({1,2}),... )
        SUM({0,1,...}) = SUM(SUM({0}), SUM({1,2...}), SUM(...))

* Materialized Views
  * if you run a subquery very very often and rarely update the underlying data, then cache it!

        CREATE VIEW myview AS ( ...query.... )

* Up-front vs execution tradeoff
  * Data cubes
  * Materialized Views
  * Indexes
  * DB vs MapReduce
  * Compiled vs Interpreted

One size doesn't fit all

* at scale, does everything poorly
* for modest datasets, good enough!
* Workload dependent alternatives: 
  * _most_ use cases dominated by a small number of workloads
    * could argue workloads driven by available optimizations..
  * Columnar, in memory, streaming, scientific

# Column Stores (OLAP)


Motivation

* Logging

        10k machines * 100hz = 1,000,000 
        * # Sensors * # data centers 
  * bulk writes, few updates, read-mostly

* Star schema
  * unnormalized retailer schema 

        store_id, store_name, store_owner, store_addr1, ....,
        emp_id, emp_name, emp_age, emp_salary, ...
        prod_id, prod_name, prod_id, prod_price, ...
        ...

  * Redundancy, big issuse, it's bad
    * update/delete anomalies
  * normalized "star" schema
    
        fact(store_id, emp_id, prod_id, cust_id, ...)
        store(store_id, name, owner, addr1, ...)
        ...

  * Why do this?
    * read less data since only care about a few attributes
    * avoid anomalies

Overview

* supports the logical relational model
* So much data that indexes don't help and in fact _hurt_ in most cases 
  * good if reading a few records/pages (we'll get to that for OLTP)
* performance bottlenecked by read throughput 
* want 100x speed up compared to existing solutions
  * data cubes 
    * expensive -- OLAP queries access different sets of attributes
    * restrictive -- for constrained queries (algebraic, distributive)
    * but can still be built (faster) on column stores


Physical layout

* Projection: Multiple columns
  * each column in project is stored separately
  * sorted on subset of the columns (sort key)

        EMP1(name, age | age)  # sorted on age

  * horizontally partitioned by value on sort key
    * sorted column is "chopped up"
  * The columns for EMP1:

        (name, storage_key)
        (age, storage_key)

        // storage key not stored, just its order number in the column
        // for joining between projections
  * Example query + projection
  * overlapping projects
  * Picking projections is an optimization problem limited by storage space
    * if don't have a projection for query, gets expensive
* Compression
  * few distinct values

        (val, startidx, endidx)
  * bitmaps

        (val, bitmap) e.g., (99, 0111100010100)

  * delta compression

        10, 12, 13, 14 --> 10,2,1,1
        reduce bits per val, then compress again

  * So fast that CPU decompression is often bottleneck!

Why not implement in a rowstore

* vertical partition the row store  
  * tuple overhead and storing pkeys substantial
    * 17 col table is 4GB compressed
    * single attr is 1GB compressed
      * 4 bytes for data, 8 bytes for header, 4 bytes for pkey
  * C-store: 240MB/col
* Mat views do well, but suffer the overheads, and limited to recurring queries (not adhoc)

Isn't storing all these projections blowing up disk costs by several X?

* Compression such a big deal, can afford to
* Disk _space_ usually not a problem, it's disk throughput per _effective_ byte of data
* Each query only accesses 1 projection,

Query execution walk through

        select avg(price)
        from data
        where symbol = 'GM' AND date = xxx

* Row store
  * read and send entire tuples throughout the plan
        
                         AVG price
                            |
                     SELECT date = xxx
                            |
                      SELECT sym = 'GM'
                            |
                            |   (GM, 1, ... xx, ...)
                            |
        [Symbol, price, nshares, exchange, date,...]
        [GM        1    ...                 xx     ]
        [GM        2    ...                 xx     ]
        [GM        3    ...                 xx     ]
        [AAPL      4    ...                 xx     ]
        ...
* Columnar using row store
  * send complete tuples up after `construct()`
  * due to record overhead, can be slower than vanilla row store

                           Avg price
                                |
                        SELECT date = xx
                                |
                        SELECT sym = GM
                                |
                                |   (GM, 1, xx)
                                |
                  Construct (Symbol, Price, Date)
               /     |                              \
        [Symbol]   [price]   [nshares]  [exchange]  [date] ...
          GM        1                                 xx  
          GM        2                                 xx
          GM        3                                 xx
          AAPL      4                                 xx


* CStore w/ late materialization
  * removes record headers
  * send compressed bitstrings through query plan

                           Avg price
                                |    (1,2,3)
                          Lookup price
                                |    (1,1,1,0)
                               AND
                 (1,1,1,0)  /        \   (1,1,1,1)
            SELECT sym = GM            SELECT date=xx
               /                                   \
              /                                      \
        [Symbol]   [price]   [nshares]  [exchange]  [date] ...
          GM        1                                 xx  
          GM        2                                 xx
          GM        3                                 xx
          AAPL      4                                 xx

* CStore w/ compression     

                           Avg price
                                |    (1,+1,+1)
                          Lookup price
                                |    (3x1, 1x0)
                               AND
                 (3x1,1x0) /        \   (4x1)
            SELECT sym = GM            SELECT date=xx
               /                                   \
              /                                      \
        [Symbol]   [price]   [nshares]  [exchange]  [date] ...
         3xGM         1                             4x "xx"
         AAPL        +1                                 
                     +1                                
                     +1                               


* Why the wins?
  * biggest wins from compression:   10x
    * compress better and read less
  * late materialization: bit strings instead of tuples:  ~3x
  * block-based execution: 
    * row-based: ~2 function calls e.g., `tuple.get_attr('id')` per tuple 
    * block: single operator call + loop   ~1.5x
    * has nice low level hardware properties (cache line, prefetch, simd, parallelization)


Highlights:

* >100x faster than RDBMS -- everyone uses it
* may not be as fast for random writes.
  * ensure sorted in every projection.
  * is this a problem?


# In-memory Stores (OLTP)

Characterization of OLTP databases

* Visa

        2B ppl, 5 xact/ppl / 3600 / 24.  / 2 for visa/mastercard ~= 57000 xacts/sec
* Simple transactions
  * remove 1 unit from product X (buy a product, swipe a card)
  * move 5 units between orgs (inventory, shipping)
* Memory is cheap


Where does the cost go in a classic database for a simple query?

* Costs (started with an inmemory classic database and started removing parts)
  * Indexes 10%
  * Logs 20%
  * Locks 25%
  * Latches 15%
  * BuffMgr 30%
  * Else: 5%
  * OK, only 5% of time is for doing work
* Why these costs?
  * Logs, locks, latches for concurrency control
  * BufMgr for pages for disk
  * Indexes for _disk_ resident data

New design

* High level
  * No parsing -- stored procedures

        Procedure:
        p1 = SELECT cost FROM data WHERE id = ?

        Query:
        p1(10)

  * No concurrency.  Serial execution
  * BufMgr not needded, just tuples in memory (scary?  failure?)
  * Memory-oriented indexes.


Recovery (replication options)

* Log shipping (store logs, ship updated values to backups)

                  Primary            Secondary
        qureies --> DBa  ---------->    DBb
                           logs     (replay logs)
  * extra work for primary, traditional logs are inefficient

* Active-active

                 /  DBa  -----> write query log (just params)
       queries --
                 \  DBb

Where are the costs now?

* Distributed transactions

# So what does this mean

Modern databases contain multiple specialized databases

        (SQLServer
          [hekaton OLTP database])

        (DB2) -- (BLU)


        (Spark
          (sparksql OLAP database)
          (spark graph))   --- Impala


# Tiny bit of distributed transactions

Remind about Time scales

* IPC costs: 5 microseconds = 0.05 ms
* Network costs
  * ping RTT -- 3-4ms
  * State of the art: 0.2ms
  * WAN: 200ms

Simple query

* a = b + c
  * a,b,c on different machines
  * Coordination is needed
    * read locks on b,c
    * write lock on a
* 2PC requires 2 round trips
  * IPC: 20k xacts/sec
  * Datacenter: 5k xacts/sec
  * WAN (megastore): 5 xacts/sec
* If replication, then consistency plays a part as well
  * wait for replicas for each value to say OK

When can you avoid coordination?

* blind inserts
* 2 increment statements (commutative -- CRDT)

Clarify Consistency vs Coordination

* Coordination
  * Concurrency and locking, you don't want to write/read the wrong results
  * ACID
* Consistency
  * When you have copies for replication, ideally any time someone reads a value from all replicas, they should be the same value
  * Can be expensive
  * Eventual consistency -- screw it, don't care


# Data/Query Processing

* Relational model and querying language
  * Alternatives?
  * Why?
  * Core set of primitive operators
* Why are databases used?
  * ACID?
  * 
* Setup towards modern systems
* Traditional layout, execution and design

* Columnar storage
  * Disk layout
* Graph
  * https://news.ycombinator.com/item?id=11257280

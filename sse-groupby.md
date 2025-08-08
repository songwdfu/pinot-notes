# SSE Aggregation Execution Enhancement

## Safe-Trim Case

### Background

Previously, there are spotted OOM errors due to very large intermediate result sizes during aggregation. 
The current solution is to keep up to `5 x LIMIT` number of rows per segment and per server (triggered by a threshold).
However, this results in inaccurate results. We face a tradeoff btw used mem and result acc. For example, when segment a has 
`[a, b, c]` and keeps `[a, b]`, while segment b has `[a, c, d]` and keeps `[a, c]`, we only get partially aggregated result for `c`. 
For most cases, this is the only solution we could live with, without having to spill to disk.

Another problem was, for `GROUP BY ... LIMIT ...` queries, i.e. when there is no `ORDER BY` clause in a group-by query, 
the results are indeterministic. The previous fix attempted to use a `ConcurrentLinkedListMap` to apply ordering according to the 
group-by key. This is logically correct, however, `ConcurrentLinkedListMap#compute` methods are **NOT** atomic, meaning the 
aggregation merge between two results could happen more than once. Since the merging is not idempotent, we would get incorrect results.
(Luckily this path is hidden behind a query option). See this [issue](https://github.com/apache/pinot/issues/16290).

As mentioned above, there's one case where we could safely trim to only LIMIT rows per segment and per server, that is when we ORDER BY
group-by keys and there is no HAVING clause (Safe-Trim). For example, `SELECT a, b, COUNT(*) FROM t1 GROUP BY a, b ORDER BY a DESC, b`. 
If we make the execution of this efficient, we could also convert no-order-by case to this by appending a order-by on group-by keys.

### Implementation

The new execution path for Safe-Trim essentially implements **sort-aggregation**. This allows cheap and timely trim of unwanted rows during both 
segment and combing level exections.

On segment level, each segment is sorted and trimmed to `LIMIT` rows only (which previously was `5 x LIMIT`). 

Then, we perform merging of sorted segment results during server combine. There are two strategy implemented for combining segment results:

First, when the number of segments is small, we simply do a **sequential merge** using the Producer-Consumer paradigm as in other server combine
executions, where producer threads process the segments, and a single consumer thread merges the results. Although theoretically its time complexity is suboptimal
compared to an N-way merge, this works well in practice due to the **streaming** fashion of merging. Without having to wait for all segments to be ready, 
this allows earlier release of in-memory segment results that will otherwise build up to mem and GC pressure.
Through benchmarking, we see this is efficient enough when the number of segments are below around `300`.

Second, when the number of segments is large, we do a **parallel pair-wise merge** that utilizes multiple threads for the first rounds of merge. 
In this case, each thread processes a segment, then either put the result into a pit or repeatedly pick up another result from the pit and 
merge it with current. This approach does not ensure strict hierarchy of merging, rather merges results that are ready greedily. In practice, 
this also benefit from early release of processed segment results as mentioned above, without having to wait for a result in the same level.

Compared to previous execution using `ConcurrentHashMap`, the sort-aggregate approach never allow the combine result to grow above `LIMIT` rows, 
and can terminate early without having to iterate through all rows for all segments. When the LIMIT is effective, 
benchmarking shows ~**30x** speedup on combine phase compared to previous approach. 
The speedup could reach a level of **300x** when there are significant GC pressure due to large number of rows.

A major drawback of sort-aggregation is, obviously, the time complexity of the sort operation. Also, the pair-wise merge approach may create more 
intermediate data structures than a single `ConcurrentHashMap` as before. Therefore, we limit this execution path to cases when LIMIT is smaller 
than a threshold (defaulted to 10,000), and expose a query option for users to adjust on-demand.

### Benchmarking Results

See [PR](https://github.com/apache/pinot/pull/16308)

## Partitioned Aggregation Execution

### Background

For cases other than Safe-Trim, the previous execution also uses a single `ConcurrentHashMap` shared between all worker threads for group-by combine.
Essentially, 8, 16, or even more threads writes to the same data structure, which might incur **high contention** even if the `ConcurrentHashMap` is highly 
optimized. Moreover, when there are trimming involved, the previous approach uses a **exclusive** lock to lock the map during sorting and trimming. 
During this all worker threads are blocked. This is unfriendly to execution environments where multiple cores are available on a single server, where we 
could exploit the parallelism further to reduce query latency.

### Implementation

DuckDB has a [blog](https://duckdb.org/2022/03/07/aggregate-hashtable.html) on partitioned parallel group-by execution that introduces a technique 
allowing zero contention between worker threads during hash group-by combine. 
The idea is to partition each segment results according to group key hashcode locally, then let each worker thread pick up specific partitions for merging. 
After this, all merged partitions are logically stitched together without the need of creating a large final table, 
exploiting the fact that group keys with different hashcodes would not need to be merged together. 

The original approach described by DuckDB chooses to process all segment results and spill them to disk before starting merging. 
This is not viable nor efficient for Pinot, as Pinot execution is fully in-memory. JFR Profiling suggests processing all segments before merging would incur 
significant memory pressure. 

Therefore, the implemented algorithm mixes up the two phases and effectively stream the segment processing and merging. 
The Producer-Consumer paradigm is again used, this time with the worker thread as both producer and consumer: 
- They produce by picking up a segment result, partitioning it, then shipping the partition blocks into 
the corresponding queue of that partition number (conceptually phase 1).
- When merging, each task has a one-to-one correspondance with a partition number and hence a queue. 
The task simply polls the blocks from the queue and merge them locally into a TwoLevelLinearProbingRecordMap (conceptually phase 2).
- When all tasks are done, the thread-local maps are stitched and returned.

The major benefit of this is that segments are merged timely, reducing required memory for holding on to the intermediate results as keys are merged. 
A new restriction introduced by this version is, the number of tasks launched must equal the number of partitions used for one-to-one matching between
task and partition.

To reduce the number of allocation needed during execution, the two-level linear probing hashtable mentioned in the same blog above is also implemented, 
with optimization of storing hashcode in the payload to avoid rehashing during resize, and storing hashcode as salt in the pointer table to reduce frequent random mem access. 
A linear in-place partitioning algorithm is used for the partition phase. Still, there are a few wrappers used to accommodate the current IndexedTable interfaces.

When there is an effective LIMIT and there are no significant memory pressure, the new approach could lower the query latency by up to **50%**, by knocking out the contention 
and stop-the-world trimming on the `ConcurrentHashMap`. 

However, there are two cases when the current implementation is inefficient:
Firstly, when the LIMIT is comparatively large, during the partition-merge phase the new approach would have to keep `numPartitions` times larger intermediate result before 
trimming after stitch to ensure correctness. This might incur larger memory pressure. The solution to this might be do earlier trimming, since partition-local trimming is cheap, 
or simply apply a threshold on LIMIT to enable this path.

Secondly, when there are lots of ties in the group-by result, when using the existing broker reduce execution path, 
(two weeks of) benchmarking and profiling located significantly **increased cache misses** that are causing slowdown during broker reduce phase compared to using 
the previous combine approach. More tracing suggested the cause to be that the previous combine phase uses `ConcurrentHashMap` same as the reduce phase, thus the 
combine output, ordered from iterating the `ConcurrentHashMap`, inserts smoothly into the `ConcurrentHashMap` in the reduce phase, while the new approach with a changed 
ordering doesn't have such favoring. In fact, any sorted order that I tried (e.g. sort by key, round-robin the partitions, using higher bits radix for partitioning) failed to beat the order from iterating the 
`ConcurrentHashMap`. A potential solution is to rewrite the reduce phase away fromm `ConcurrentHashMap` as well (we could also utilize that the combine output is also intrinsically partitioned). But this is to be tested.


### Benchmarking Results

See [PR](https://github.com/apache/pinot/pull/16452)

## Other Misc Enhancements on SSE

- Clear thread-local maps for `DictionaryBasedGroupKeyGenerator` immediately after use to free up memory and avoid OOM.
- Do linear merge of sorted combine outputs during broker reduce when server already sort the output results. 

# Query Planner / Optimizer Enhancements

Pinot does not use a cost-based query optimizer (QO) for several reasons: Firstly, cardinality estimate (CE)
for a real-time system with multiple, frequently-updating data sources like Pinot is difficult, and we don't have the CE system now. 
Often, cost-based planning with bad CE is worse than non cost-based. Another reason is the best open-source cost-based QO, Apache Calcite's 
VolcanoPlanner still has considerable inefficiencies in planning latencies. Using such system in Pinot may incur additional latency overheads.
(May wanna checkout this [talk](https://youtu.be/5wQojihyJDs?t=88)). Therefore, Pinot uses a **heuristic rule-based optimizer** with carefully 
placed sets of rules that fires in-order. Several enhancements are made on Pinot's rule-based query planner.

## Knobs for Runtime Optimizer Rule Adjustments

Previously, the sets of heuristic rules used in Pinot query planning were **static**. This means any rule in the static set will 
be fired as long as its condition matches. This has two inefficiencies: Firstly, there are rules in the default set that could backfire under 
certain scenarios. For example, `FilterJoinTranspose` rule pushes a Filter below a Join to reduce the join input cardinality. While this is desirable 
under most scenarios, if we have a case where the filter contains a very expensive UDF that takes seconds to evaluate, not pushing the filter 
down might allow the UDF to be evaluated over less tuples, which might end up running faster than the pushdown plan. Once things like this happen, 
we have no control over it. Second, due to above, we had to be very careful about what rules to put in the static sets. 
If there is a rule (e.g. aggregation pushdown) that is highly useful under specific scenarios, while being harmful under most other scenarios, 
we couldn't put it into the rule set.

To solve this, two sets of query options were supplied to allow enabling / disabling specific rules **during runtime**. This allows rules that are 
backfiring for specific queries to be turned off, and allows adding defaultly disabled rules to be used when appropriate.

Another internal change this brought is a convention on managing rules. Each rule is now initialized with a description that serves as an identifier, 
both for displaying on the query console and for controlling on/off. This allows distinguishing same rules fired in different stages of planning as well.

## Rule Enhancements

There are several peep-hole optimization rules added that are mostly conducive for simplifying query logical plan. This list includes 
`JoinPushTransitivePredicates`, `PruneEmptyCorrelate`, `ProjectRemove` (in `PRUNE_RULES`), and defaultly disabled `AggregateJoinTransposeExtended`.
The new Enriched Join is also disabled by default controlled by `JoinToEnrichedJoinRules`. 
See [pinot-docs page](https://docs.pinot.apache.org/users/user-guide-query/query-options) for complete usage.

To illustrate, one should consider enabling `AggregateJoinTransposeExtended` when aggregation reduces join input cardinality significantly. For example, 
for this query on TPC-H benchmark, enabling this rule cuts runtime from `4721ms` to `421ms`:
```
SELECT SUM(l_quantity)
FROM part p INNER JOIN lineitem l
ON p.p_partkey = l.l_partkey
GROUP BY l.l_partkey, p.p_partkey
```


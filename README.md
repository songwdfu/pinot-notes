# Apache Pinot Query Engine Optimizations at StarTree

## Introduction

The Apache Pinot project, originally developed at LinkedIn, is an open-source distributed real-time OLAP database system that is designed to provide ultra low-latency analytics at extremely high throughput, with an emphasis on data freshness. 

During summer 2025, my project at StarTree, Inc. focused on query engine optimizations for Apache Pinot. The project touched multiple components within the query engine. This repo serves as a summarization of most works I've done during the summer.

![Pinot query engine architecture](./resources/pinot-query-engine-architecture.png)
*Figure 1: Apache Pinot query engine architecture*

**My major responsibility includes:**
- Working on the Apache Pinot Project, optimizing the execution engine for reduced query latency, increased throughput, and reduced memory usage. 
- Shipping new features for the new Multi-Stage Execution Query Engine.
- Identifying and hot-fixing system inefficiencies  

**My major outcomes includes:**
- Reduced query execution latency in group-by execution by over 40% under multiple scenarios. Sped up sorted group-by Combine component execution by over 30 times.
- Reduced intermediate result size and reduce allocation by above 30% in Join, Sort, Group-By, and Scan execution.
- Supported run-time adjustment of query planner rules with knobs. Introduced planner rule management conventions. Supplied additional heuristic rules for query optimization.

## Links to Contents

- Single-Stage Query Engine Group-By Execution Enhancements [link](src/sse-groupby.md)

- Multi-Stage Query Engine Join Execution Enhancements [link](src/mse-join.md)

- Query Planner / Optimizer Enhancements [link](src/query-planner.md)

## Acknowledgements

I'd like to thank [Jackie Jiang](https://github.com/Jackie-Jiang), [Gonzalo Ortiz Jaureguizar](https://github.com/gortiz), [Bolek Ziobrowski](https://github.com/bziobrowski), [Yash Mayya](https://github.com/yashmayya), [Neha Pawar](https://github.com/npawar), and more wonderful people from [StarTree](https://github.com/startreedata) for their incredible help and support throughout the summer. I'm truely grateful❤️. 


---
layout: post
title:  Resolved Issues
categories: RELEASE_NOTES
parent: xap100.html
weight: 300
---


Below is a list of issues that have been fixed in GigaSpaces 10.0.X.



{: .table .table-bordered}
| Key | Summary | Fixed in Version | SalesForce ID | Platform/s |
|:----|:--------|:----------------|:---------------|:------------------|
| GS-11505 | StandaloneProcessingUnitContainerProvider#close() raise Interruption that cause problems in unit tests | 10.0.0 | 8518 | Java |
| GS-11536 | LRMI threads cause JVM-wide slowdown with parallel reads | 10.0.0 | 8527 | Java |
| GS-11587 | Concurrency bug in LRU/Off-heap - unpin is called when entry not locked when ifExists waiting templates removed | 10.0.0 | | All |
| GS-11616 | ReadModifiers missing default constructor | 9.7.1, 10.0.0 |8657 | Java |
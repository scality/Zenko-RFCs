# XDM hardware sizing requirements
## Context
Customers are buying servers like commodity hardware. They are making grouped
purchase for all their use-cases (virtualization, bigdata, Software-defined
  Storage... ) who leads to a very cheap price for standard server configurations.
On the other hand, if customers want to buy a specific server configuration, it
cost a lot more to buy and maintain(specific operation, firmware, spare part,
energy consumption...).

E.g:
-   stardard configuration: base server + 3,2TB SSD = 2k$
-   specific configuration: base server + 6,4TB SSD = 10k$

Today our sizing tool provides huge requirements in terms of SSD capacity
per server. This leads to specific configurations.

| Sizing Summary per server | Use Case: 700TB usage with an avg object size of 137KB
| --- | ---
| Boot Drive SSD | 0.3 TB
| Persistant Volume SSD | **7.27 TB**
| Total RAM per Server |	128GB
| Total CPU per Server |	32

Customers would like to have smaller SSD capacity requirements per server
and more servers  because it's cheapest to buy and easier to maintain.

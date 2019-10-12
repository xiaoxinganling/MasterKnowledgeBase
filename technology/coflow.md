coflow 一词由 UCB 的 Mosharaf 在 [Coflow**: a networking abstraction for cluster applications.**](https://dl.acm.org/citation.cfm?doid=2390231.2390237) 中提出。对于传输层而言，网络流是透明的。即在传输层无法区分网络流的不同之处。但这种做法有一些弊端，因为这些横跨不同 machine 的网络流往往包含很多 application-level 的信息，例如，mr-shuffle 的最后一个 flow 决定了 shuffle 阶段的完成时间。而上述抽象显然是忽略了这些信息。

coflow 涵盖许多 flow，并且包含这些 flow 的统计信息，例如之前提到的 application-level 的信息。

> We refer to a semantically-related collection of flows between two groups of machines as a *coflow*. Each *coflow* contains information about its structure and the collective objective of its flows(*Meet deadline* or *Minimize completion time*)
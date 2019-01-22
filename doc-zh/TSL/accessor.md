# 设计理由
GE有一个名为Memory Cloud（内存云）的分布式内存存储基础架构。这个memory cloud由一些memory trunks组成，集群中的每台机器都承载256个memory trunks。我们将一台机器的本地内存空间分为多个memory trunks的原因在于两点：
（1）可以不需要任何锁的开销实现trunk层面的并行；
（2）Memory trunk使用hash机制去做内存寻址。由于hash冲突的可能性较高，单个大hash表的性能是不太好的。


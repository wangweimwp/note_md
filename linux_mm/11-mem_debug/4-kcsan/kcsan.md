 一、KCSAN介绍

KCSAN(Kernel Concurrency Sanitizer)是一种动态竞态检测器，它依赖于编译时插装，并使用基于观察点的采样方法来检测竞态，其主要目的是检测数据竞争。

KCSAN是一种检测LKMM(Linux内核内存一致性模型)定义的数据竞争(data race)的工具，同时它也可以控制报告哪种类型的数据竞争。

KCSAN知道LKMM定义的所有标记原子操作，以及LKMM尚未提到的操作，例如原子位掩码操作(bit mask)。

KCSAN扩展了LKMM，例如通过提供data_race()标记，来表示存在数据竞争和缺乏原子可能性。

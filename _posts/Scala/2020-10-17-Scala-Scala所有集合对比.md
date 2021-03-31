---
title: Scala所有集合对比
categories:
- Scala
tags: [Scala]
---

# 不可变函数式集合

本质上在Java中无完全与Scala不可变集合对应的集合，这里列举只是因为功能相似（并且尽量是同级别的）。

| # | 类型 | 父类型 | apply()实现 | constant time or near | thread safety | Java中类似 |
|---| ---- | ------- | ----- | ------------------------------ | ------------ | ---------------- |
|Seq|trait|Seq|ListBuffer|last、length、update、append、toList|-|ArrayList|
|IndexedSeq|trait|IndexedSeq|VectorBuilder|length、apply|-|-|
|LinearSeq|trait|LinearSeq|ListBuffer|head、tail|-|-|
|Range|class|IndexedSeq|-|-|-|-|
|NumericRange|abstract class|IndexedSeq|-|-|-|-|
|List|abstract class|LinearSeq|ListBuffer|prepend、head、tail|-|AbstractList|
|Queue|class|LinearSeq|ListBuffer|enqueue、dequeue|-|Queue*|
|Vector|class|IndexedSeq|VectorBuilder|update、append、prepend|-|Vector(thread safety)|
|Map|trait|Map|MapBuilder| |-|Map|
|SortedMap|trait|SortedMap|MapBuilder| |-|SortedMap|
|HashMap|class|Map|MapBuilder| |-|HashMap|
|TreeMap|class|SortedMap|MapBuilder| |-|TreeMap|
|TrieMap|class|Map|MapBuilder| |是|-|
|ListMap|class|Map|MapBuilder|last、init|-|LinkedHashMap|
|Set|trait|Set|GenSetFactory| |-|Set|
|SortedSet|trait|SortedSet|SetBuilder| |-|SortedSet|
|ListSet|class|Set|GenSetFactory|last、init|-|LinkedHashSet|
|HashSet|class|Set|GenSetFactory| |-|HashSet|
|BitSet|abstract class|SortedSet、BitSet|Builder| |-|BitSet|
|TreeSet|class|SortedSet|SetBuilder| |-|TreeSet|
|RedBlackTree|object|Tree|-| |-|-|

虽然文档没有明确不可变集合的多线程下的安全性，一般的：对于不可变集合若其自身仅包含不可变元素，那么在多线程中应该是线程安全的，而里面的元素如果是可变的，那么集合仍然可能是非线程安全的。

# 流

TODO
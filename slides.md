---
title: Scala Slides
description: My Awesome Scala Talk ™️
author: Your Name Here
keywords: scala,
url: https://alterationx10.com
marp: true
---

# Scala Slides

A template project to build Scala-centric slide decks with mdoc and marp

---

# Fancy Example

Here is an example of some **Scala** code processed with `mdoc` to illustrate
how you can enhance your slides:

```scala mdoc:silent
val fibs: LazyList[BigInt] =
    BigInt(0) #:: BigInt(1) #:: fibs.zip(fibs.tail).map{ n => n._1 + n._2 }
```
```scala mdoc
fibs.take(10)
```



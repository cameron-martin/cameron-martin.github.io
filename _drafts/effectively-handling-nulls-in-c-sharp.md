# Effectively handing nulls in C#

## The problem

Tony Hoare has called the introduction of nulls his [Billion dollar mistake][billion-dollar-mistake]. The main problems with them are:

* You cannot express a non-nullable reference type. This means that if your method takes a `User` object,
  it can also take the value `null`. If this was not your intention, this leads to what is sometimes called
  *Overly Defensive Progamming*, i.e. your code becomes littered with checks for null as the first thing you do in every method.

[billion-dollar-mistake]: http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare

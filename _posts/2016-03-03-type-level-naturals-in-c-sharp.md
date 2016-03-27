---
layout: post
title:  "Type-Level Natural Numbers in C#"
date:   2016-03-03 20:05:00 +0000
categories: c-sharp
---

Over the past few weeks, I've been experimenting with [Idris]. One of the key things about this language is that types are just values (with type `Type`). This means you can use normal values in type signatures, such as natural numbers. Idris uses the [peano construction][peano] of the natural numbers. The parts of the axioms that are of use here are:

* 0 is a natural number.
* For every natural number n, S(n) is a natural number.

These can be easily expressed in Idris by

{% highlight haskell %}
data Nat = Z | S Nat
{% endhighlight %}

Then they can be just used in other type signatures, for example a length-encoded list

{% highlight haskell %}
data Vect : Nat -> Type -> Type where
  Nil  : Vect Z a
  (::) : (x : a) -> (xs : Vect k a) -> Vect (S k) a
{% endhighlight %}

This gives us types such as `Vect 1 Nat`, which contains only single-element lists with elements belonging to the type `Nat`. This is a different type to `Vect 2 Nat`, which contains all 2-element lists with elements belonging to the type `Nat`.

This is easy in Idris, but how would we do it in a more everyday language, C#?

{% highlight c# %}
public class Nat
{
    private Nat() {}

    public sealed class Zero : Nat
    {
        private Zero() {}
    }

    public sealed class Succ<T> : Nat where T : Nat
    {
        private Succ() {}
    }
}
{% endhighlight %}

This lets us construct natural numbers at the type level. So the type `Zero` represents the number `0`, `Succ<Zero>` represents the number `1`, `Succ<Succ<Zero>>` represents the number `2`, and so on.

We can then create a `Vector` type, a linked list which is parametrised by it's length. Just like the Idris example, a member of the type `Vector` is either an *empty vector* (of length 0), or it is a *vector of length n+1*, constructed from an element (known as the head) and a vector of length n (known as the tail). In the absence of sum types, we'll use inheritance to model these two possibilities.

{% highlight c# %}
public abstract class Vector<L, T> : IEnumerable<T>
    where L : Nat
{
    public abstract IEnumerator<T> GetEnumerator();

    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }

    public override string ToString()
    {
        return "[" + String.Join(", ", this) + "]";
    }
}

public class EmptyVector<T> : Vector<Nat.Zero, T>
{
    public override IEnumerator<T> GetEnumerator()
    {
        yield break;
    }
}

public class ConsVector<LMinusOne, T> : Vector <Nat.Succ<LMinusOne>, T>
    where LMinusOne : Nat
{
    public T Head { get; }
    public Vector<LMinusOne, T> Tail { get; }

    public ConsVector(T head, Vector<LMinusOne, T> tail)
    {
        Head = head;
        Tail = tail;
    }

    public override IEnumerator<T> GetEnumerator()
    {
        yield return Head;

        foreach(var item in this.Tail)
        {
            yield return item;
        }
    }
}
{% endhighlight %}

Because type parameter inference does not work for constructors, we need to create some helper functions to give us more terse syntax for creating these vectors.

{% highlight c# %}
class VectorDemo
{
    public static EmptyVector<T> Empty<T>()
    {
        return new EmptyVector<T>();
    }

    public static ConsVector<LMinusOne, T> Cons<LMinusOne, T>(T head, Vector<LMinusOne, T> tail)
        where LMinusOne : Nat
    {
        return new ConsVector<LMinusOne, T>(head, tail);
    }

    public static ConsVector<Nat.Zero, T> Cons<T>(T head)
    {
        return Cons(head, Empty<T>());
    }

    static void Main()
    {
        var vect1 = Cons(1, Cons(2));
        var vect2 = Cons(1.2, Cons(2.5, Cons(3.0)));

        Console.WriteLine(vect1); // => [1, 2]
        Console.WriteLine(vect2); // => [1.2, 2.5, 3]
    }
}
{% endhighlight %}

Okay, so we can create and print our length-encoded lists. But we can't do much else with them yet, so let's define some of the fundamental operations for lists: `Map`, `ZipWith` and both left and right fold.

{% highlight c# %}
public abstract class Vector<L, T> : IEnumerable<T>
    where L : Nat
{
    public abstract IEnumerator<T> GetEnumerator();

    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }

    public override string ToString()
    {
        return "[" + String.Join(", ", this) + "]";
    }

    public abstract Vector<L, B> ZipWith<A, B>(Func<T, A, B> zipper, Vector<L, A> otherVect);
    public abstract Vector<L, A> Map<A>(Func<T, A> f);
    public abstract A FoldL<A>(Func<A, T, A> f, A init);
    public abstract A FoldR<A>(Func<T, A, A> f, A init);
}

public class EmptyVector<T> : Vector<Nat.Zero, T>
{
    public override IEnumerator<T> GetEnumerator()
    {
        yield break;
    }

    public override Vector<Nat.Zero, B> ZipWith<A, B>(Func<T, A, B> zipper, Vector<Nat.Zero, A> otherVect)
    {
        return new EmptyVector<B>();
    }

    public override Vector<Nat.Zero, A> Map<A>(Func<T, A> f)
    {
        return new EmptyVector<A>();
    }

    public override A FoldL<A>(Func<A, T, A> f, A init)
    {
        return init;
    }

    public override A FoldR<A>(Func<T, A, A> f, A init)
    {
        return init;
    }
}

public class ConsVector<LMinusOne, T> : Vector <Nat.Succ<LMinusOne>, T>
    where LMinusOne : Nat
{
    public T Head { get; }
    public Vector<LMinusOne, T> Tail { get; }

    public ConsVector(T head, Vector<LMinusOne, T> tail)
    {
        Head = head;
        Tail = tail;
    }

    public override IEnumerator<T> GetEnumerator()
    {
        yield return Head;

        foreach(var item in this.Tail)
        {
            yield return item;
        }
    }

    public override Vector<Nat.Succ<LMinusOne>, B> ZipWith<A, B>(Func<T, A, B> zipper, Vector<Nat.Succ<LMinusOne>, A> otherVect)
    {
        var otherCasted = (ConsVector<LMinusOne, A>) otherVect;

        return new ConsVector<LMinusOne, B>(zipper(this.Head, otherCasted.Head), this.Tail.ZipWith(zipper, otherCasted.Tail));
    }

    public override Vector<Nat.Succ<LMinusOne>, A> Map<A>(Func<T, A> f)
    {
        return new ConsVector<LMinusOne, A>(f(Head), Tail.Map(f));
    }

    public override A FoldL<A>(Func<A, T, A> f, A init)
    {
        return this.Tail.FoldL(f, f(init, this.Head));
    }

    public override A FoldR<A>(Func<T, A, A> f, A init)
    {
        return f(this.Head, this.Tail.FoldR(f, init));
    }
}
{% endhighlight %}

Unfortunately, we have to do a typecast in `ZipWith`, because the compiler does not know that if something is a non-empty vector, it must be of type `ConsVector`. In fact, this assumption isn't actually true, because somebody else could subclass `Vector` and pass it to `ZipWith`. But we'll ignore this possibility.

So now we can do stuff like

{% highlight c# %}
var vect1 = Cons(1, Cons(2, Cons(4)));
var vect2 = Cons(2, Cons(3, Cons(6)));
var vect3 = Cons(1, Cons(4, Cons(5, Cons(7))));

Console.WriteLine(vect1.ZipWith((a, b) => a + b, vect2)); // => [3, 5, 10]
Console.WriteLine(vect3.FoldL((a, b) => a + b, 0)); // => 17
{% endhighlight %}

But if we try to zip `vect1` and `vect3`, we get a type error:

    The best overloaded method match for `Vector<Nat.Succ<Nat.Succ<Nat.Succ<Nat.Zero>>>,int>.ZipWith<int,int>(System.Func<int,int,int>, Vector<Nat.Succ<Nat.Succ<Nat.Succ<Nat.Zero>>>,int>)' has some invalid arguments
    Argument `#2' cannot convert `ConsVector<Nat.Succ<Nat.Succ<Nat.Succ<Nat.Zero>>>,int>' expression to type `Vector<Nat.Succ<Nat.Succ<Nat.Succ<Nat.Zero>>>,int>'

Awesome.

Open questions:

* To append two vectors, we need to sum two type level naturals. Is this even possible?
* How useful is this in practice? For example, how do we convert user input to these vectors?

[Idris]: http://www.idris-lang.org/
[peano]: https://en.wikipedia.org/wiki/Peano_axioms

---
layout: post
title: OO-style interfaces for Haskell
description: An implementation of an object-oriented concept in a functional language with type classes
date: 2015-03-04 18:08:00
categories: programming
project_id: interface
---
Haskell doesn't have interfaces which are a fundamental mechanism to achieve modularity in
object-oriented languages. This might seem strange at first, but it turns out that isn't a problem
because type classes fill a similar role.
In this post, however, I'll present a more direct way that can be used to simulate interfaces using them.

You might be able to deduce something about the expressivity of one mechanism compared to the other,
but I'm not nearly informed enough to do that. Besides, I'm sure it's already been done.
This is mostly meant as an interesting proof of concept. Not an actual, useful thing.

The class
---------

The actual machinery to do this is laughably short.
So short, in fact, that I'll couple it with an extra operator we'll use for convenience later.

{% highlight haskell %}
class Interface c i where
    i :: c -> i

infixl 1 -->

(-->) :: a -> (a -> b) -> b
(-->) a b = b a
{% endhighlight %}

At the type level, the class represents pairs `(c, i)` of types `c` that implement the interface `i`.
At the value level it provides a method to lift an object of type `c` so it can be used through one of
the interfaces it's type implements.

IEnumerable
-----------

The example I'm choosing to use is the `IEnumerable` interface from .NET. It's pretty much strictly
worse than the existing type classes in Haskell that do the same thing, but it's not like I'm
trying to make some point here. The interface itself is interesting enough to be a good candidate
for an example.

We're actually going to need two interfaces here. The `IEnumerator` and the `IEnumerable`.
The `IEnumerable` interface declares only one method: `getEnumerator` which returns an enumerator
of the same generic type. This will all be clear in the signatures.

The `IEnumerator` is slightly more involved and declares 3 methods and one property.
The property will just be a "parameterless" method. One of the methods is `Dispose` which we'll ignore.
I'll also take some liberties with the signatures of the rest to make them make more sense in Haskell.

The main idea behind an interface will be a structure that the implementer is required to fill.

So here are their definitions.

{% highlight haskell %}
data IEnumerator t = IEnumerator
                   { current :: t
                   , moveNext :: Maybe (IEnumerator t)
                   , reset :: IEnumerator t
                   } deriving Show

data IEnumerable t = IEnumerable
                   { getEnumerator :: IEnumerator t
                   } deriving Show
{% endhighlight %}

The `moveNext` traditionally mutates the current enumerator, but I decided to maybe return a new one.
The `Maybe` is there because the method should fail if we're at the end of the collection.
`reset` returns the enumerator to the beginning and `current` is self explanitory.

Making a list enumerable
------------------------

The only type I'll implement the interface for is the basic list.
To that end we declare a new type, `ListEnumerator` that will be our `IEnumerator`.

{% highlight haskell %}
data ListEnumerator t = ListEnumerator
                      { leCurrent :: [t]
                      , leStart :: [t]
                      } deriving Show

leMoveNext :: ListEnumerator t -> Maybe (ListEnumerator t)
leMoveNext le = case leCurrent le of
    (_ : x : xs) -> Just $ le { leCurrent = x : xs }
    _            -> Nothing

leReset :: ListEnumerator t -> ListEnumerator t
leReset le = le { leCurrent = leStart le }
{% endhighlight %}

And finally, we make the `ListEnumerator` implement `IEnumerator` and the list itself implement
`IEnumerable`.

{% highlight haskell %}
instance a ~ b => Interface (ListEnumerator a) (IEnumerator b) where
    i le = IEnumerator
         { current = head . leCurrent $ le
         , moveNext = fmap i . leMoveNext $ le
         , reset = i . leReset $ le
         }

instance a ~ b => Interface [a] (IEnumerable b) where
    i l = IEnumerable . i $ ListEnumerator l l
{% endhighlight %}

There's one interesting thing here, and that's the equality constraint (~).
My first approach was `instance Interface (ListEnumerator a) (IEnumerator a)` and this works, but
using it is problematic. Ideally, we can't expect `i x` to return anything sensible because we have
no idea which interface we want to lift the object `x` to. But what we would expect is that something
like `getEnumerator $ i x` does resolve. Unfortunately, with the above setup it wouldn't.

The reason is that, while `x :: [a]` might implement the interface `IEnumerable a`, it could potentially
also implement `IEnumerable b` for some other type `b`. Because of this, `getEnumerator $ i x` is still
too ambiguous of an expression to do something with it because the compiler can't decide which
`IEnumerable` we want.

The fix (and I was pretty surprised it works that well) is in the equality constraint.
If you look at the head of the instance (which is the only thing that matters when the compiler
checks for instances), we have `Instance [a] (IEnumerable b)`. This doesn't say anything about the
types `a` and `b` so it matches all of them. This lets the compiler know that there's only one
instance that could be used here.
Next, the `a ~ b` part lets it infer one type from the other, as long as one of the is known.

Usage
-----

.NET defines a ton of extension methods for the `IEnumerable` interface. Extension methods in Haskell
don't really differ from normal functions though.
I implemented some of them so there's something to play with. I won't go over them in this post,
but feel free to check out the code.

Here's an example use case of our interface using those extension methods

{% highlight haskell %}
  i ["hello", "world", "how", "are", "you"] --> where' (\s -> length s > 3)
                                          --> select (\s i -> s ++ "!")
                                          --> toList

> ["hello!", "world!"]
{% endhighlight %}

Neat!

The error messages are also pretty good.

{% highlight haskell %}
i 'a' --> where' (\s -> length s > 3)
      --> select (\s i -> s ++ "!")
      --> toList
{% endhighlight %}
`No instance for (Interface Char (IEnumerable [Char]))`

Summary
-------

This was certainly a fun venture, though I can't speak for the practicality of it.
One use case that comes to mind are functions like `C y => x -> y` where we would like to allow
functions to only be able to return some `y` of the class `C`. Not ANY `y`. To this end we could use
the described method in a relatively painless way.

The code as it looks at the time of the writing is available [here](https://github.com/LukaHorvat/Interface/tree/66537dcf7e41447ad22093378ed940b6f7b1dcd5).

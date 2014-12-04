---
layout: post
title: Type-level magic
description: "Using functional dependencies for type level computation"
date: 2014-12-03 18:43:00
categories: programming
project_id: compose
---
This time we're going to look at how we can use functional dependencies to do various type-level
computation. First we'll introduce the concepts used and then go over the specifics.
I won't walk through the whole code because most of it is just implementation or pretty straight
forward stuff.

Our goals are to introduce type-level Boolean algebra, natural number arithmetic and heterogeneous
lists.
We'll try to mimic value level code as close as possible and see if we can minimize the number of
hoops we have to jump through to get it working at type level.

A disclaimer: I am by no means an expert. I do not claim that everything I write here is true,
but most of the explanations have at least some value and are a valid way of thinking for a specific
set of problems. I'll gladly accept any corrections.

The code makes heavy use of GHC extensions and it wouldn't work without them.

Lifting to type level
---------------------

Here are the value-level to type-level mappings we're going to use.

Constructors map to matching data types declarations. This means that the LHS of a data type declaration
now has the same semantical value at the type-level, as the RHS had at value-level.

Types map to typeclasses. This will be true for type-level functions as well. We'll declare
their types using class statements.

Function implementations map to instance declarations. We'll see how we can have something similar
to `let` bindings and how we can do recursive calls.

Classes as relations, relations as functions
--------------------------------------------

It's useful to think of a typeclass (I'll be using 'class' for short) as a set of types, or tuples of 
types. The `Num` class is a set of all types that satisfy `Num`'s constraints. When we write `Num a`
we're saying "The following definition works for type a, if a is a member of the `Num` set".

We won't find much use for single parameter classes in this project. Classes with multiple parameters is
where it's at. We can use the same analogy as before, except this time, the sets don't contain types
but tuples of types. The reason this is useful is because a set of pairs defines a relation. If `a` is in
relation `R` with `b` then there exists a pair `(a, b)` in the `R` set. This membership is again expressed
as `R a b`. Similarly, we can extend the idea for any tuple size.

Why do we care about relations? Because they can be used to express functions. In fact, any function
is a relation whose elements are `(x, f x)`. But the converse doesn't hold. Not every relation is a
function.
A simple example is `{(x, y), (x, z)}` where `y /= z`. If this were a function then `f x = y`, but also
`f x = z`. Can't do that with functions.
This is where we'll need functional dependencies.

Boolean algebra
---------------

So now that we have our foundation, let's look at some code. Open up [TypeLevel](https://github.com/LukaHorvat/Compose/blob/b3b5f5cdc9c1fc762f31dc01abfbb2bc4db32b6a/src/TypeLevel.hs) and follow along.

{% highlight haskell %}
data True  deriving Typeable
data False deriving Typeable
{% endhighlight %}

We declare the only two values. `deriving Typeable` is there because why not.
Next, we'll use two classes that will represent the type of our values.

{% highlight haskell %}
class Bool' a
class Bool' a => Bool a
instance Bool' True
instance Bool' False
instance Bool' a => Bool a
{% endhighlight %}

The reason there are two of them and they both mean the same is because now we can hide one and export the
other. This way users can still use the exported one in type declarations, but they can't make new
instances of them because the hidden one is a superclass of the exported one.

The first operation we'll implement is the equivalence one. `a <=> b` is true if both of them are the same
Boolean value. I call this operation `Is`.

{% highlight haskell %}
class (Bool x, Bool y, Bool b) => Is x y b | x y -> b
instance Is True  True  True
instance Is True  False False
instance Is False False True
instance Is False True  False
{% endhighlight %}

Let's talk a bit more about the specifics of the implementation.
Firstly, notice how we use our `Bool` type here to restrict the function to proper values.
The `| x y -> b` is the functional dependency. This means that our three-member relation
is actually a function from `(x, y)` to `b`. This means that for every distinct pair of
`(x, y)` there can't be more than one distinct `b` in relation with them.

Then we just provide the whole truth table as instances of this relation.
If we tried adding another one like `True True False` for example we get an error because the pair 
`(True, True)` already has their assigned value.
The functional dependency isn't just to prevent us from writing faulty instances. It's there to actually 
enable computation as we'll see later.

And, Or and Not are implemented the same way. The only interesting thing is that Not is a function
of only one parameter

The next one is interesting. The opposite of `Is` is `Isnt` or `Xor` which turns out to be the same thing.
Let's see how `Isnt` is implemented.

{% highlight haskell %}
class (Bool x, Bool y, Bool b) => Isnt x y b | x y -> b
instance (Is x y b1, Not b1 b2) => Isnt x y b2
{% endhighlight %}

Let's see what's happening here.
To read the instance literally would sound something like "If `x`, `y` and `b1` are in the `Is` relation,
and `b1` and `b2` are in the `Not` relation, then `x`, `y` and `b2` are in the `Isnt` relation".
This description, however, isn't very operable.

A better way to think about it is "`b1` is the result of `Is x y`, and `b2` is the result of `Not b1`. Now
the result of `Isnt x y` is `b2`".
The reason this actually does something is because of our dependencies. The right side is the question
"What's the result of `Isnt x y`?". This introduces the `x` and `y` parameters.
Now, if `x` and `y` are some fixed values, then `Is x y b1` actually fills in the value of `b1`. This is
because our functional dependency says that there's only one possible `b1` that could fit in with a
fixed `x` and `y`. Now that we know `b1`, `Not b1 b2` fills in `b2`. The same reasoning applies.

This is how the bind results to names and how we call other functions.

Next we name some value level functions so we can test out our stuff.

Here's a function you can use to see it in action.

{% highlight haskell %}
u :: a
u = undefined

ex0 :: IO ()
ex0 = do
    let res1 = xor (u :: True)  (u :: False)
    let res2 = xor (u :: True)  (u :: True)
    let res3 = and (u :: False) (u :: True)
    print $ typeOf res1
    print $ typeOf res2 
    print $ typeOf res3
{% endhighlight %}

Notice that because we didn't bother with constructors for our data types, we can't instantiate the
actual types in code. We can however use `undefined` and just say that it's of some type we want.
We can't actually compute the VALUE of a function but we can compute it's TYPE and that's exactly what
we want.

Other tricks
------------

I won't go over the implementation of natural numbers. It's just applying the same thing we did before.
It does have a few specifics that I will go over.

Look at the implementation of `Plus`

{% highlight haskell %}
class (Nat x, Nat y, Nat z) => Plus x y z | x y -> z
instance Nat x => Plus Zero x x
instance Plus x y z => Plus (Succ x) y (Succ z)
{% endhighlight %}

Here, one instance is our edge case and the other one is the recursive call.
There's a catch with this kind of setup. It's a bit involved though to let's build up to it.

Say we want to implement the `Equals` function that works for ANY type. Our first attempt might look
something like this

{% highlight haskell %}
class Bool b => Equal x y b | x y -> b
instance Equal x x True
instance Equal x y False
{% endhighlight %}

No matter what extensions you enable to try and convince the compiler that this is sane, it won't work.
The reason is that the second instance says that the pair `(x, y)` is in a relation with `False` for any
`x` and `y`. The first instance claims this isn't true. We just can't do defaulting like this.

Now, here's what we CAN do

{% highlight haskell %}
class Bool b => Equal x y b | x y -> b
instance Equal x x True
instance Not True b => Equal x y b
{% endhighlight %}

This time, looking at just the right hand sides, there's no conflict. We state that for any `x`, the pair
`(x, x)` is in a relation with `True`. Also, for any `x` and `y`, the pair `(x, y)` are in a relation with
`b`. This `b` can, so far, be anything. 
When the instance actually gets resolved, `b` is calculated to be `Not True` which is
nothing other than `False`. We just smuggled it through a back door. Another way we could have written
that is `False ~ b` instead of `Not True b`.
I like mine better because it doesn't introduce another new thing.

Another slightly interesting case is `Minus`. Here our functional dependency tells us that the result of
the operation and the second operand actually determine the first operand. To see why this makes more
sense than what we would usually write is quite simple. `Minus One Two b` would need to fill in the value
of `b`, but there's obviously no natural number we can use.
Written the way it is it good because for any combination of natural numbers as the second operand and
the result, there's one possible first operand. It inverts our `Minus` operation into addition. And that's
precisely how it's implemented. Neat!

Now division is a beast. The same way we implemented `Minus` using `Plus`, we could implement `Over` using
`Times`. But somewhere along the way we introduce some complexity that the type inference engine simply
doesn't want to resolve. I don't know where and what the problem is, but it just doesn't work.

What does work is a simple accumulator-style recursion. For that purpose we use the `Over'` operation that
also takes an extra parameter.

The implementation isn't interesting so I'll omit it here.

Summary
-------

I've implemented some other operations you can look at. Also, there's an implementation of heterogeneous
lists. As a demonstration, there's also an implementation of numbers as lists of binary values. Addition
of those numbers is also there.

Check our the code as it is at the time of the writing here: [TypeLevel](https://github.com/LukaHorvat/Compose/blob/b3b5f5cdc9c1fc762f31dc01abfbb2bc4db32b6a/src/TypeLevel.hs), [Binary](https://github.com/LukaHorvat/Compose/blob/ba9fc98ca4c57ba16e662fd1f0a5a41e57f677dc/src/Binary.hs)

This shortish rundown doesn't serve a specific purpose other than to show some of the things that can
be done by bending the Haskell type system to your will.
Using similar techniques but with types actually attached to some values you can express arbitrary
constraints on parameters of your functions. All with the goal of concerting runtime bugs into compile
time errors.
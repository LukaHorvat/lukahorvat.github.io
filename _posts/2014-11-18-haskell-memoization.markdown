---
layout: post
title: Haskell memoization
description: "Using lazy evaluation to achieve function memoization"
date: 2014-11-18 00:28:00
categories: programming
---
Today we'll go through the process of writing a higher order function designed to
memoize it's argument.

While that probably sounds a bit too abstract, it will become clearer later. Hopefully :D

Our motivating example will be writing a function that calculates the n-th Fibonacci number.

Firstly, let's write a fast enough implementation that we'll use to test correctness
of our other implementations. This one will have a linear time complexity and will
be faster than our final memoized version.
Now, you might ask, why bother with doing all this if we're going to introduce a FASTER
version right from the start?
There are several benefits. One of them is that our memoized version will match the
recursive definition almost completely meaning that our code won't really have to adapt
to accommodate optimizations. Another benefit is that our `memo` combinator will apply
to a whole class of functions.

So, here's the fast implementation.
{% highlight haskell %}
fib = (fibs !!)
    where fibs = 1 : 1 : zipWith (+) fibs (tail fibs)
{% endhighlight %}

I won't go into detail how this works. If you're familiar with Haskell and it's lazy semantics
you can probably figure it out. In fact, I'm pretty sure most Haskell programmers have
even seen this specific implementation.

Now we can start building up. Let's write the naive version.

{% highlight haskell %}
nfib 0 = 1
nfib 1 = 1
nfib n = nfib (n - 1) + nfib (n - 2)
{% endhighlight %}

This is a pretty great definition. We get to use pattern matching so our code resembles the
mathematical definition. Unfortunately, every call to `nfib` spawns two more, so the
time complexity for this version is exponential. No bueno.

Let's imagine we already have our magical `memo` combinator available. How would we use it?
The first instinct might be something like this.

{% highlight haskell %}
mfib = memo nfib
{% endhighlight %}

While that certainly does something, it doesn't do what we want it to do. Every call to
`mfib` with a number previously used will be fast, but how many calls to `mfib` will we actually
do? The problem is that our `nfib` function isn't aware that there exists a memoized version
of it so even if we cache it's return value, it doesn't access that cache. It just recursively
calls itself.
After thinking a bit about this, we might come up with this version, which will be correct.

{% highlight haskell %}
mfib = memo fib'
    where fib' 0 = 1
          fib' 1 = 1
          fib' n = mfib (n - 1) + mfib (n - 2)
{% endhighlight %}

Here our memoized version actually wraps around a normal, recursive definition. The only
difference is that the recursive definition doesn't make calls to `fib'` but `mfib`.
The cached version.

A thing to note here is that this means we won't be able to memoize functions that we can't
redefine. If there's some function `f` that we know recurses, we can't make it faster.
That's too bad, but it would require some serious hackery to fix that.

So our objective will be to define the function `memo`. To see how we might do that, we'll
look at yet another implementation of the `fib` function. This one will be memoized.

{% highlight haskell %}
lfib = (map fib' [0..] !!)
    where fib' 0 = 1
          fib' 1 = 1
          fib' n = lfib (n - 1) + lfib (n - 2)
{% endhighlight %}

Now this is pretty good. It resembles our `memo` version and it doesn't perform half bad.
For me, it can do `lfib 50000` in GHCi in about 15 seconds. Not to mention we could extract
our `memo` function directly from here. It would look something like this.

{% highlight haskell %}
memo f = (map f [0..] !!)
{% endhighlight %}

Pretty sweet. Using Haskell's niftyness we managed to capture a pretty general concept
in a higher order function. Unfortunately, there's always a 'but'.
There are two things we can all will do better. Firstly, the type of our memo function is
`(Num a1, Enum a1) => (a1 -> a) -> Int -> a`. For all intents and purposes, we can look at
it as `(Int -> a) -> Int -> a`. This means it's generic in the functions return value,
which is nice, but it can only memoize functions that operate on Ints.
Secondly, this `memo` implementation is still too slow. It requires a traversal of the
cache list every time we want to get a value out because lists have linear time indexing.
The complexity of our `lfib` function is quadratic or so.

Still, this will serve as a motivation for our actual, real implementation.
Notice how it works. It maps our function over all possible inputs and stores the results.
However, because of non-strict evaluation it doesn't really do anything until it absolutely
needs to. This is what allows us to use an infinite structure to cache our results.
We'll never evaluate the 1000th element of the list if we only calculate the first 999.

The same idea can be applied to any data structure in Haskell. We can define them recursively
and Haskell will just say "Yeah, sure. Whatevs'", treating our definition like instructions
on how to generate any specific part of that structure.

To avoid the linear access time in our structure, we'll use a binary tree instead of a list.
Trees have nice, logarithmic time, access properties that we want.

The general idea is to somehow establish a mapping from the functions argument type, to the
set of nodes of our tree. Each unique argument should have a unique node assigned to keep
the result value.
The way I chose to do this, and I'm sure there are other, better, options, is to introduce
a new typeclass `BitsBijective` that represents types that support serialization into and 
from a list of boolean values.

Here's my tree definition, along with the typeclass definition.

{% highlight haskell %}
data Memo a b = Fork (Memo a b) b (Memo a b)
    deriving Show

type Bit = Bool
newtype Bits = Bits [Bit]

instance Show Bits where
    show (Bits b) = "[" ++ map (\x -> if x then '1' else '0') b ++ "]"

{-
Laws:
toBits . fromBits = fromBits . toBits = id
∀n
    zeros := replicate n False
    ∀x fromBits x = fromBits (x ++ zeros)
-}
class BitsBijective a where
    toBits :: a -> Bits
    fromBits :: Bits -> a
{% endhighlight %}

Some constraints that should be satisfied by all instances are written down in the comment.

We can make Ints an instance of our class relatively easily.

{% highlight haskell %}
instance BitsBijective Int where
    toBits n = Bits $ map (testBit n) [0..finiteBitSize n - 1]
    fromBits (Bits b) = foldl setBit 0 $ map fst $ filter snd $ zip [0..] b
{% endhighlight %}

And using two helper functions we can make any pair of `BitsBijective` values also
`BitsBijective`.

{% highlight haskell %}
interleave :: Bits -> Bits -> Bits
interleave (Bits l) (Bits r) = Bits $ interleave' l r
    where interleave' []       [] = []
          interleave' []       ys = False : interleave' ys []
          interleave' (x : xs) ys = x : interleave' ys xs

uninterleave :: Bits -> (Bits, Bits)
uninterleave (Bits b) = (Bits l, Bits r)
    where (l, r) = uninterleave' b
          uninterleave' []       = ([], [])
          uninterleave' (x : xs) = let (rs, ls) = uninterleave' xs in (x : ls, rs)

instance (BitsBijective a, BitsBijective b) => BitsBijective (a, b) where
    toBits (toBits -> l, toBits -> r) = interleave l r
    fromBits (uninterleave -> (l, r)) = (fromBits l, fromBits r)
{% endhighlight %}

So, what do we get from this? Well, we'll require that the functions we memoize have
arguments of a `BitsBijective` type. While this is kind of restrictive, it's already
better than our first attempt since Ints are `BitsBijective` AND we can write instances
for other types. But why impose that restriction? 
It's got everything to do with the
way we'll store the data in our tree. We'll turn the argument into ones and zeros,
and then, starting at the root of the tree, we'll take a left turn for each zero,
and take a right turn for each one. This will lead us to the node that will have
the result of our function stored.

One caveat to watch out for is one of the laws. We ignore the trailing zeros. This
enables us to serialize pairs of values that don't necessarily have the same number
of bits, and then still be able to reconstruct both of them. Here's the final `memo`
implementation. Notice that as we accumulate the path to each node when constructing the tree,
we have to reverse it before getting the actual value because we keep PREpending
each turn we take. That's not ideal, but it's good enough.

{% highlight haskell %}
memo :: BitsBijective a => (a -> b) -> a -> b
memo f = readMemo
    where memo' l = Fork (memo' $ False : l) (f $ fromBits $ Bits $ reverse l) (memo' $ True : l)
          tree = memo' []
          readMemo x = followPath b tree
              where (Bits (ker -> b)) = toBits x
                    followPath []           (Fork _ y _) = y
                    followPath (False : xs) (Fork y _ _) = followPath xs y
                    followPath (True : xs)  (Fork _ _ y) = followPath xs y
          ker xs | null ones = []
                 | otherwise = take (1 + fst (last ones)) xs
                 where ones = filter snd $ zip [0..] xs
{% endhighlight %}

And that's it! Now our original `mfib` definition works, and it works great!
For me it can calculate the 50000th number in just under 2 seconds. And that's
in GHCi.
The final complexity should be somewhere around `n * m` where n is the index and
m is the number of bits in our argument. We can also test correctness using our
original implementation. The first 1000 numbers match, which is proof enough for me.

Here's a [link](https://gist.github.com/cf4ea50ba78095eca341) to the complete version. 

Well, I hope this wasn't too dry of a read. I've certainly had fun writing it and about it.
If you're got comments, I'd be glad to hear them.


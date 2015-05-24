---
layout: post
title: HaskellForms
description: WinForms bindings for Haskell built around a .NET interop system
date: 2015-05-24 15:06:00
categories: programming
project_id: haskellforms
---

Windows GUI
-----------
I was looking for an easy to use GUI library for Haskell that worked on Windows. I'm sorry to say but
the whole experience was very frustrating. Various native bindings are a nightmare to setup and more often
than not, just decide not to work.

So naturally, I decided to write my own solution.

How it works
------------
One part of the library is the .NET component written in F#. It's an executable that manipulates the
controls, handles events and communicates over standard IO. The Haskell component starts that host
process and maps Haskell code into appropriate messages over std. IO, as well as mapping responses
and events back to the Haskell world.

Example usage
-------------
The main idea was that the library should feel like a native Haskell one. Here's a code example.
{% highlight haskell %}
main :: IO ()
main = do
    startHost False
    form <- newForm
    pb   <- newPictureBox
    controls form >>= add pb
    bmp  <- newBitmap 500 500
    image pb #= bmp
    red  <- newSolidBrush $ color 255 0 0 255
    graphicsFromImage bmp >>= drawString "test" (font "Consolas" 12) red (pointF 0 0)
    refresh form
{% endhighlight %}
![Result](http://i.imgur.com/jJH1lik.png)

Interop
-------
The interop layer is actually pretty feature rich. It handles marshaling of data on both ends.
Simple objects (like Point, Size...) get serialized to strings while more complex ones (like controls)
are kept on the .NET end and get assigned IDs which are then used by the Haskell side to refer to them.

In the above example, the `newX` functions create objects on the .NET side.
The `controls` function is the analog of the `Controls` property that returns a collection of child
controls. This collection is also an object that needs to be marshaled, and it does. Then the
`add` function is called on it passing in the `pb` object.
The Haskell code looks pretty much how it would look in C#/F#.

In this example you can see that using GDI functions works without a problem.

It's important to note that the bindings are nowhere near complete at this point. Only a couple
of functions and classes are supported. This isn't because it's a lot of work to bind things.
It just wasn't a priority to make a binding generator.

I've tried to mirror the OOP hierarchy as close as possible. In the example I'm creating a new `Bitmap`
object but I'm assigning it to a property that expects an `Image`. This works because of the subclassing.

Events
------
Events are supported and (as it turns out) actually more typesafe than their .NET counterparts.
{% highlight haskell %}
main :: IO ()
main = do
    startHost False
    form <- newForm
    btn  <- newButton
    controls form >>= add btn
    click btn >>= handle (\_ _ -> putStrLn "Test")
{% endhighlight %}

Here the first argument to the event handler isn't just some `object`. It's a `Button`.

Performance
-----------
...was not a concern. This doesn't mean that the bindings are slow, but serializing everything to
strings and piping over std. IO can't be too efficient. However, it really should be workable for
pretty much everything where you just need a UI (as opposed to a complicated rendering system).
The serialization is very much modularized so if there's actual need, it can always be replaced with
a more efficient system.

Technicalities
--------------
I've spend a decent amount of effort adjusting the implementation so it's as convenient as possible
to use.

For example,
{% highlight haskell %}
main :: IO ()
main = do
    form <- newForm
    btn  <- newButton
    controls form >>= add btn
    text btn #= "Test"
    test <- text btn
{% endhighlight %}

the `text` property the first time it's used behaves like a setter (since `#=` can be used with it),
while the second time behaves like an `IO String` and is bound to a name.

The bindings have support for get-only properties.

Take a look at [Controls.hs](https://github.com/LukaHorvat/HaskellForms/blob/7922f6b8a89e359be2525dcec912f107bb69c9ab/src/WinForms/Controls.hs) to see what the bindings look like.
They basically just connect the untyped, stringy layer to meaningful types.

The code as it looks at the time of the writing is available [here](https://github.com/LukaHorvat/HaskellForms/tree/7922f6b8a89e359be2525dcec912f107bb69c9ab).

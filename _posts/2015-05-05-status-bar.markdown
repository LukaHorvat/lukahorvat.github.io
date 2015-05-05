---
layout: post
title: An extendable status bar for Windows
description: A simple application to help you keep track of things happening on your computer
date: 2015-05-05 11:39:00
categories: programming
project_id: status-bar
---
Motivation
----------
I'm a part of a larger chat group of friends. The conversations are interesting but sometimes, the topic doesn't really concern you so the endless chat notifications only end up being a distraction. What I end up doing is turning them off and then,
more often than not, forgetting to turn them back on.

There's a better solution. When you're on a mobile device, there's a useful status bar always on top.
When a message arrives the bar conveniently shows it for a few seconds so you can keep track of the conversation
happening while still having the sound notifications turned off and without having to constantly switch windows.

Approach
-----------
I made something like that, but for Windows. One of the major differences between a mobile environment and
Windows is that I didn't have a centralized notification system to hook into. This means that the support
for each chat application would have to be written manually.
Naturally, I decided to implement a plugin system so that the workload can be distributed.

Plugins can be written in any .NET language that can produce assemblies. The types in the assembly are
scanned for static methods and properties with correct signatures and then hooked into the status bar.

The bar consists of the ticker and the status.
The ticker receives events from plugins and displays their messages while revealing the status bar
for a few seconds before hiding it again.
The status actively requests updates from plugins (only when the bar is visible) and updates the same
text.

The plugins should be placed in a `plugins` folder next to the exe. In that folder there should be a
`deps` folder where the managed dependencies of the various plugins should be placed. The unmanaged dependencies
should be placed next to the application's exe.

Example plugin: QBittorrent
---------------------------
Unrelated to the original chat-centric motivation, the status bar can be a convenient tool for various other programs.
Using QBittorrents web API, I wrote a plugin that displays the current speed, ETA and progress of the torrent being
downloaded. This happens in the status part of the bar. When the torrent finishes downloading, an event is fired that
appears in the ticker.

The plugins also share a configuration mechanism. A configuration consists of key-value pairs where both are strings.
The plugin first supplies a dictionary that names all the keys and provides their default/placeholder values.
Then the plugin is required to have a static method that accepts a new dictionary that contains the filled in configuration.

You can see all of that code at the time of writing [here](https://github.com/LukaHorvat/StatusBar/blob/deca6ef8c5c8b60571604ee1c016d85e0dfca7e7/QBittorrent/Torrent.fs).

The interface
-------------
The plugins are free to do what ever they want. They can run arbitrary code in their static initializers or during
the execution of any method in the interface. This means they are free to fork threads, start timers, or anything else.

The interface that they should provide so that the main application can talk to them is the following:

{% highlight csharp %}
// The configuration keys and their default/placeholder values
public static Dictionary<string, string> ConfigurationSchema { get; }

// The method to call when the configuration has been loaded. If there is no configuration available
// one will be created and the user will be given a chance to edit it before continuing.
public static void Configure(Dictionary<string, string>);

// This method takes an action that should be called every time some event happens. It should pass a
// message for the ticker and a numeric priprity value (0, 10)
public static void AddTickHandler(Action<string, int>);

// If the plugin wants to provide a status, this is where it provides the initial text for it.
public static string Status { get; }

// This method should asynchronously return a new status when called.
public static Task<string> UpdateStatus();
{% endhighlight %}

All three of those components (the configuration, the ticker and the status) are optional, but if you decide
to include one, you have to include it fully for it to work.

Technicalities
--------------
The base of the application is a Windows form. Unfortunately, to make it really borderless,
always on top, tiny and hidable, also to capture global key presses, I needed to use a ton of WinAPI
p/invokes. The consequence is that it only works on Windows. However, the whole WinAPI module is in a
separate file so if someone decided to port it to other operating systems, all they would have to do is replace
those methods with equivalents.

The code as it looks at the time of the writing is available [here](https://github.com/LukaHorvat/StatusBar/tree/deca6ef8c5c8b60571604ee1c016d85e0dfca7e7).

Everything but the Skype plugin is written in F#.

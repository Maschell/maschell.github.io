---
layout: post
title:  "Bug hunting - gotta catch 'em all! - Part 1"
date:   2018-03-13 14:25:00 +0100
categories: bugs
---
Yay! This is the first blog post! 
This post is about a weird bug that occurred while refactoring the [homebrew launcher](https://github.com/dimok789/homebrew_launcher) for the Wii U.
It's the first time I'm writing something like this, but I hope you enjoy reading it!

# Background information
Back in October 2017 I started to take common Wii U homebrew source code and put into multiple statically linked libraries. 
The result where for example [libutils](https://github.com/Maschell/libutils), [libgui](https://github.com/Maschell/libgui) or [dynamic_libs](https://github.com/Maschell/dynamic_libs). 
Having generic code has multiple advantages. 
One reason for doing this is it's way easier to update all existing application that uses this code. 
In the past updating common code (like the dynamic_libs, logging, file system accesses) was very time consuming and often not done properly. 
This lead to inconsistent code between applications.
One of the other reason is simply reducing the code size of the main application. 
When generic code parts that take up a significant amount are outsourced, the code often gets much smaller (and often even cleaner).

# The bug
Anyway. After creating these static libs, I tried to refactor as many existing application to make profit of this new libs. 
One of the these were the [homebrew launcher](https://github.com/dimok789/homebrew_launcher).
The homebrew launcher was created by dimok, it allows the user to load own homebrew applications from the sd card.
I didn't create or work on this, but I decided to still give it a shot.
After refactoring I was able to [**remove ~ 16000 lines of code**](https://github.com/dimok789/homebrew_launcher/commit/5ed93e14a8d8addf4f4fba1fce73783e11c7c8e2).
Together with [**~ 2000 lines of code**](https://github.com/dimok789/homebrew_launcher/commit/c43f85152a6e83f198f23e405f138656dfbd23f0) from a previous refactoring it's kinda a big deal.
But...for some reason it had random softlocks when loading applications.

So for some reason the launcher just kept running with a black screen and still running the background music. 
I went trough the whole code changes and searched for anything I might broke while refactoring - with not success.
After some testing and minor changes, I noticed that it only happens if [I disable the logging](https://github.com/dimok789/homebrew_launcher/commit/7f49e39e4224f7eacd796e19e87568afacf95c86). 
This was _really_ confusing me. 
Why would _disabling_ logging trigger anything bad? 
A few hours later without any progress on this, I decided to just leave it on the repository with logging enabled and moved on with other projects.

# Working on other projects
Jumping to the present time. 
In last couple of weeks I spent much time on developing a [Wii U Plugin System](https://github.com/Maschell/WiiUPluginSystem). 
Last week I added a new plugin that allows to user to take screenshots by pressing a button combination on the game pad. 
When testing this plugin alone everything was working as expected, but as soon as I tried to combine it with other plugins, I got random results.
The screenshot began to be inconsistent, from time to time the buttons combination to trigger taking a screen stopped working. 
This buttons combination is set by the user when the plugin was loaded via an splash screen. 
Internally the corresponding buttons combination is saved in a global variable (inside the .data section). 
This way it survives when the console switches the running game/application.

While debugging this issue, I realized it was suddenly gone. 
The only thing I changed was _adding_ logging. 
Once again I was really confused, but this time it made some sense. 
I logged the global variable, I did something with the variable.
It turns out it was (probably) a caching issue between the CPU-Cores of the Wii U, [adding a cache flush manually](https://github.com/Maschell/WiiUPluginSystem/commit/1157026b8b828c58d03e93ac74ba8ee598342e4c#diff-1c5db9ef9c1e5ce7a3aff3780d7d6f5b), the whole issue was resolved.

# Back to resolving this bug
This little bug in the screenshot plugin reminded me of the bug in the homebrew launcher. 
Both issues disappeared when I added/enabled logging, so I thought I give it another shot. 
Maybe it's a similar problem?

So checked the complete source code for places where variables were logged. 
After not finding anything useful, I decided to enable logging and remove one log message after another, until it breaks. 
But it didn't break.
I noticed the softlock only happens when logging is disabled __and__ when loading a file directly from the sd card. 
When the file is sent over the network via _wiiload_, everything was working as expected. 
WTF?.
This clearly indicates this has something to do with the difference in handling file loading, but I had still no clue how this was connected to logging.
Loading the file over the network is handled by a class called [TcpReceiver](https://github.com/dimok789/homebrew_launcher/blob/master/src/menu/TcpReceiver.cpp).
It creates a new thread, opens a socket and waits for a connection.
When someone connects, it receives the homebrew file and loads it into memory, otherwise it simply ends when the application was closed.
The fact the application softlocked screamed "THIS THREAD IS NOT ENDING". 
And in fact, when I removed the _TcpReceiver_ from the code, the softlock was gone.
Loading files from sd card was fine, even with logging disabled.
Now I found the bad part - I tried to solve it.
I checked the while logic of the _TcpReceiver_:

{% highlight c %}
while(!exitRequested){
    s32 clientSocket = accept(serverSocket, (struct sockaddr*)&clientAddr, &addrlen);
    if(clientSocket >= 0){
        // do stuff
        if(result)
            break;
    } else {
        os_usleep(100000);
    }
}
{% endhighlight %}
It waits for a connection in a loop which stops when `exitRequested` is set to true.
This happens in the destructor of this class, so maybe it's also a caching issue of this variable?
I added some [cache flushes](https://github.com/dimok789/homebrew_launcher/commit/ccbc7b040a3e90bf896eb134616ec369a4899ca4#diff-b85ab73f5bc3f19b58cf77d5b4341f74), like I did with the screenshot plugin, but still no success.

# One final attempt

There need to be any relation between having logging enabled and the _TcpReceiver_. 
But I already removed all logging messages, only he `log_init()` call was remaining.
And bingo!
I finally found the part that triggers the softlock.
Whenever the `log_init()` is missing and the file is loaded directly from the sd card, the launcher softlocked.
So whats happening inside the [`log_init()`](https://github.com/Maschell/libutils/blob/master/source/utils/logger.c#L14-L29)?
 
{% highlight c %}
void log_init_() {
    InitOSFunctionPointers();
    InitSocketFunctionPointers();

    int broadcastEnable = 1;
    log_socket = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (log_socket < 0)
        return;

    setsockopt(log_socket, SOL_SOCKET, SO_BROADCAST, &broadcastEnable, sizeof(broadcastEnable));

    memset(&connect_addr, 0, sizeof(struct sockaddr_in));
    connect_addr.sin_family = AF_INET;
    connect_addr.sin_port = 4405;
    connect_addr.sin_addr.s_addr = htonl(INADDR_BROADCAST);
}
{% endhighlight %}

It's loading some needed functions, creates a socket and sets some connection details.
The socket creation is the only thing that _may_ interfere with the _TcpReceiver_.
I tested adding a socket creation right after the `log_init()` and disabling logging.
And... bingo!
It worked, so it has something to do with the `socket` call?.
I [pushed a commit](https://github.com/dimok789/homebrew_launcher/commit/ccbc7b040a3e90bf896eb134616ec369a4899ca4#diff-7ec3c68a81efff79b6ca22ac1f1eabba) with this "fix" and made a short break.

# Finally fixing the bug

It was possible to disable logging without breaking the launcher, but I still didn't find the bug itself.
I put some more time in thinking about the relationship of creating this socket and the _TcpReceiver_.
What exactly is the `socket` function doing? 
It creates a socket and returns a socket handle on success, returning -1 on error.
The socket handle is just an integer, counting from 0.
Wait, counting from 0?
Finally I had the relationship!
`log_init()` is called __before__ the _TcpReceiver_ is created.
When `log_init()` is called, the socket function inside _TcpReceiver_ returns 1 instead of 0.
Let's take a look at the destructor:

{% highlight c %}
TcpReceiver::~TcpReceiver(){
    exitRequested = true;
    if(serverSocket > 0){
        shutdown(serverSocket, SHUT_RDWR);
    }
}
{% endhighlight %}

`exitRequested` is the flags that is set to stop the [while loop](https://github.com/dimok789/homebrew_launcher/blob/ccbc7b040a3e90bf896eb134616ec369a4899ca4/src/menu/TcpReceiver.cpp#L72-L90) as discussed earlier.
`shutdown` is closing the socket, which causes the `accept` to return with an error and leaving the loop.
But only if... the variable `serverSocket`, which contains the socket handle, is greater 0.
Ouch. There it is, there is the problem.
Because no other socket was created when logging is disabled, the `socket` function returned 0 as socket handle, which is not triggering the shutdown.
All we need to do is added one character, one `=`.

{% highlight c %}
TcpReceiver::~TcpReceiver(){
    exitRequested = true;
    if(serverSocket >= 0){
        shutdown(serverSocket, SHUT_RDWR);
    }
}
{% endhighlight %}

There it is.
After debugging and testing for multiple hours the bug was just this missing `=`.
The reason why loading a file over the network (using the _TcpReceiver_) was working is because the `accept` returned in the moment someone connected, without relying on the socket shutdown.

# Conclusion

So whats the conclusion?
The simplest bugs are often hardest one to resolve.
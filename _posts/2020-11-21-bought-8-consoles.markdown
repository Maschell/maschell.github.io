---
layout: post
title:  "The day I bought 8 \"broken\" Wii Us"
date:   2020-11-21 17:45:00 +0100 
categories: other
---

Sometimes when I am bored I scroll through ebay looking for newly listed "buy it now" 
consoles to see if there is something interesting/cheap. Once I stumbled across a box
 full of three untested PS1 consoles and ten controllers for 30€. A few years ago my
 own PS1 broke so I gave it a try. Even if just one of the three console and one
 controller would be working, it would've been worth it. When the box full of PS1
 consoles and controllers arrived a couple of days later, I tested everything. It
 turns out that one console couldn't read discs and two controller were broken. But
 in the end I still had two consoles (+ cables) and more controllers than I would ever
 need for 30€. It even included a multitap, so a couple a days later was playing
 Micro Machines for the first time with 4 players on a playstation!


## Why even bother to look for broken consoles?
Since then I also tend to specially search for console related items which have the
 condition `For parts or not working`. Even some "broken" consoles can be useful.
 For doing my Wii U related coding, I wouldn't care if the disc drive or HDMI port
 is broken. I almost never put in a disc in the console, and while coding 99% of the
 time I only look on the screen of gamepad anyway. In worst case I could use the old
 CRT that I have on my desk for some casual rounds NES tetris. Some consoles might be
 broken for "normal" people, but still be useful for me.

## Buying 8 Wii Us...
A few days ago I finally found a really interesting listing on ebay. A total of 8 "broken"
 Wii U consoles (no cables, no gamepad, but all are the 32GB version) for 119€. Normally
 you spent easily 50€ for a (working) replacement console in a reasonable condition.
 The listing even had some information about the condition of the consoles: (freely translated)
```
2x HDMI port defect
1x No image
3x locked( probably account data and password not known)
1x non-European sales version
1x Defect
```
All of them except the last one sounded like something that *could* be possible to fix
 or use. Like I said earlier, as long I as can somehow connect a gamepad, I don't care if
 the video output is broken. The "locked" consoles sounded like some parental controls
 are applied, but there is an [online tool](https://mkey.salthax.org/) to reset the
 parental controls, so they also shouldn't be an issue.

In theory only 3 of the 8 console would need to be usable for me until I wouldn't 
consider it a waste of money. Without spending too much time on thinking what I even
 want to do with 8 consoles, I pressed the "Buy it now" button.

## 36 hours later

Just 36 hours later a big box with 8 Wii U consoles arrived. They all seemed to be in a pretty
 good condition (~~much better than my two main consoles~~). I piled them up on a big
 "Wii U tower" and took the picture before testing them one by one.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">I have no idea what I am doing
 here. <br><img src="/res/consoles.jpg" width="600" align="middle"></p>&mdash; MaschellDev
 (@MaschellDev) <a href="https://twitter.com/MaschellDev/status/1329744651889348608?ref_src=twsrc%5Etfw">
 November 20, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

#### Console 1
At first I was interested in the "non-European sales version". Fairly quickly I identified
 the console and gave it a shot. Turns out it was a US console, so I paired my EU gamepad.
 I always thought the gamepads are region locked but to my surprise I was able to go 
 through the whole console setup process. After that the console restarted and tried 
 to update my gamepad - which failed, because the gamepad region was wrong. It's 
 possible to get into the Wii U menu without the gamepad (when paring a wiimote or 
 pro controller), but most apps can't be started without the gamepad. For quite a bit 
 time I tried to trick the console to accept my EU gamepad, without any success. For 
 now I am stuck until I have a US gamepad. I would still consider it a "success" as this 
 console is working. If I really want to use it, I can use 
 [drc-sim](https://github.com/rolandoislas/drc-sim) to emulate a gamepad using a PC.

#### Console 2
After that I picked up the next console. This one seemed to be one of the "locked" consoles. 
When I started the console it presented me the login screen with one password protected 
account. There is possibility to create a new account, but it's locked via the parent controls. 
Even if I would've known the password for the account (NNID), I wouldn't be able to login 
because the console was offline. For setting up the internet connection I also needed the 
PIN to bypass the parental controls. 

Earlier in the post I mentioned a website which helps to reset the parental controls. But 
I couldn't use it. In order to reset, you need the `Inquiry number`, which can only be 
accessed from the parental control app (which can only be accessed from the Wii U Menu). 
So my only option to get into the Wii U menu was guessing the parental control PIN and 
create a new account.

Luckily the PIN is only 4 digits long, in worst case I have to test 9999 combinations. 
Annoying, but with enough time this one is also solveable. I gave it a few tries but 
moved on to the other consoles. When I came back later, I systematically tried out all 
birthdates (this reduced the number of possible PINs down to 366!) and after ~30 minutes 
and ~250 tries I was in. Apperantly the birthdate of the previous owner is November 26th :) 

Knowing the code I was able to create a new account code and remove the parental controls. 
Everything else seemed to work fine. Another saved console!

#### Console 3

This console at first looked very promising. I turned it on and it wasn't locked, but after 
a few seconds after opening the Wii U menu I got the error code `160-0103` though. I gave it 
a few more tries but the console always froze or showed an error code. Very likely that 
the emmc has died. RIP console 3 =(

#### Console 4

The next console didn't output any image via HDMI, but when I tried a composite cable it 
worked! I was able to pair the gamepad and change the output back to HDMI. This console 
also seemed to work without any issues. So we far we have 3 working consoles out of 4! I 
have reached my target of three saved consoles :)

#### Console 5

This console just worked. Well. Maybe this console has some other issues after playing 
game for some hours, but so far it looks like a normal working Wii U! 4 out of 5!


#### Console 6

Another locked console like console 2. This time I only needed a few tries as the PIN was 
0505 which seemed like an obvious pattern to try. Probably the old console owners birthdate 
was May 5th though :D 5 out of 6!

#### Console 7

The next console didn't output anything via HDMI. I tried the same "composite"-trick like 
on console 4, but still had no success. Normally having no video output is fine, as long 
as you can use the gamepad. But to connect the gamepad to the console, you need to enter 
a PIN displayed on the TV. This reminded me of console 6 and 2, but this time I had to 
guess the connection PIN. Bruteforcing this was a bit more pain though as each attempt 
took like 10 seconds. The other issue was, that the syncing dialog is closing after some 
time and I have no way to check if the syncing dialog is even open. In worst case I'm 
trying to connect but the console is not even waiting for any connections.

As a possible solution for console 1 I mentioned [drc-sim](https://github.com/rolandoislas/drc-sim). 
In order to emulate a gamepad, you need to go through the same connecting process with 
the PIN. I thought I may can automate the bruteforcing, but in the end I spent like 5 hours 
fighting with serveral ubuntu vms, without any success. I went back to manual bruteforcing.

After testing like 30 PINs, I remembered some console have a button combinations to reset 
the video mode. Unfortunately the Wii U doesn't seem to have such feature, but I came 
across and random Youtube video that showed how to fix a "no image on TV"-issue. Because 
they already had a gamepad synced, they proposed to try different video output modes until 
it works. 

This was the moment where I realized, that the Wii U has more possible outputs than HDMI 
and composite. There is also component output. There is one problem: I don't have a component 
cable to test this. But what I have is a "AV Multi Out" to HDMI adapter for my Wii and I 
gave it a shot. On my first display I was shown a "invalid hdmi signal" error, but on my 
slighty more expensive one I finally saw the Wii U menu! I checked the settings and apparently 
the console was set to non-hdmi and 576i. After setting it to HDMI and 1080p the console 
worked perfectly! 6 out of 7!

#### Console 8

The last console had exactly the same issue as console 7. After connecting it using the 
"AV Multi Out" to HDMI adapter I was able to change the video output to HDMI. Everything 
seemed to work fine. 7 out of 8 consoles working!


# Conclusion

In the end everything turned out to be much better than I have expected. I could "save" 7 
of the 8 consoles without even opening any of them. The description of the consoles looked 
very promising and I was very lucky with the outcome. This could've easily been a waste of 
119€ ending up with a huge pile of broken Wii Us. ~~Now it was a waste of 119€ with a huge 
pile of working Wii Us.~~

If you're now also hyped to buy some "broken" consoles here are some tips:
- If you have fun at entering 9999 possible PINs, buy a "locked" console.
- From my experience the "console has no video output" problems are no real problem, you 
just need the right cables and display to "fix" these
- Be prepared to end with a pile of electronic garbage.
- Only spent the amount of money you have no problem with losing.
- Remember that you also need a gamepad (which are even more expensive than the consoles)

I even found another "no video ouput"-console on ebay for <20€ which sounded very promising, 
but I decided to not buy it, even though it probably works fine. I don't even know what to do 
with the 9 working consoles I currently own. 
---
layout: post
title:  "The day I bought 8 \"broken\" Wii Us"
date:   2020-11-21 17:45:00 +0100 
categories: other
---

Sometimes, when I get bored, I scroll through ebay looking for newly listed "buy it now"
 consoles to see if there is something interesting/cheap. Once I came across a box with
 three untested PS1 consoles and ten controllers for 30€. A few years ago my PS1 broke a
 few years ago, so I gave it a try. Even if only one of the three consoles and one controller worked,
 it would have been worth it. When the box full of PS1 consoles and controllers arrived a
 few days later, I tested them all. It turned out that one console couldn't read any discs
 and two controllers were broken. But I still had two consoles (+ cables) and more controllers
 than I would ever need for 30€. It even included a multitap, so a few days later
 I played Micro Machines for the first time with 4 players on a Playstation!
 
 **Update 2024-04-09: For quite some time there a now tools that can be used to [bypass parental controls via raspberry pi pico](https://github.com/GaryOderNichts/recovery_menu) or [fix consoles with the 160-0103 system errors](https://gbatemp.net/threads/ultimate-wii-u-troubleshooting-guide-system-memory-error-160-0103-stuck-wii-u-screen-stuck-factory-reset-black-screen-after-stuck-update.642339/).**

## Why even bother to look for broken consoles?
Since then, I also tend to look specifically for console-related items that have the condition
 `for parts or not working`. Even some "broken" consoles can be useful. For my Wii U-related
 coding, I don't care if the disc drive or HDMI port is broken. I almost never put a disc in
 the console, and 99% of the time I just look at the gamepad screen while coding anyway. Worst
 case scenario, I could use the old CRT I have on my desk for a few casual rounds of NES Tetris.
 Some consoles may be broken for "normal" people, but still useful for me.

## Buying 8 Wii Us...
A few days ago I finally found a really interesting offer on ebay. A total of 8 "broken" Wii U 
consoles (no cables, no gamepad, but all are the 32GB version) for 119€. Normally you could 
easily spend 50€ for a (working) replacement console in reasonable condition. The listing even 
had some information about the condition of the consoles: (freely translated)
```
2x HDMI port defect
1x No image
3x locked( probably account data and password not known)
1x non-European sales version
1x Defect
```
All of them except the last one sounded like something that *could* be possible to fix or use.
 Like I said earlier, as long I as can somehow connect a gamepad, I don't care if the video 
 output is broken. The "locked" consoles sounded like some parental controls are applied, 
 but there is an [online tool](https://mkey.salthax.org/) to reset the parental controls, 
 so they also shouldn't be an issue. 
 
 **Update 2024-04-09: There are [new ways](https://github.com/GaryOderNichts/recovery_menu) to reset the parental controls by using a raspberry pi pico.**
 
In theory only 3 of the 8 console would need to be usable 
 for me until I wouldn't consider it a waste of money. Without spending too much time on 
 thinking what I even want to do with 8 consoles, I pressed the "Buy it now" button.

## 36 hours later

Just 36 hours later, a large box with 8 Wii U consoles arrived. They all seemed to be in 
pretty good condition (~~much better than my two main consoles~~). I stacked them on a big 
"Wii U Tower" and took the picture before testing them one by one.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">I have no idea what I am doing
 here. <br><img src="/res/consoles.jpg" width="600" align="middle"></p>&mdash; MaschellDev
 (@MaschellDev) <a href="https://twitter.com/MaschellDev/status/1329744651889348608?ref_src=twsrc%5Etfw">
 November 20, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

#### Console 1
At first I was interested in the "non-European sales version". I quickly identified 
the console and the console and gave it a try. It turned out to be a US console, so 
I plugged in my EU gamepad. I always thought the gamepads were region locked, but to 
my surprise I was able to go through the entire  through the entire console setup process. 
The console then restarted and tried to  to update my gamepad - which failed because the 
gamepad region was wrong. It's  possible to get into the Wii U menu without the gamepad 
(if you are using a Wiimote or  Pro Controller), but most apps won't launch without the gamepad. 
For quite some time I tried to trick the console into accepting my EU gamepad, 
but to no avail. For  now I am stuck until I get a US gamepad. I would still consider it a 
"success" since this console works. If I really want to use it, I can use [drc-sim](https://github.com/rolandoislas/drc-sim) 
to emulate a gamepad with a PC.

#### Console 2
I then picked up the next console. This seemed to be one of the "locked" consoles. When I started the 
console, it presented me with the login screen with a password-protected account. It's possible to create 
a new account, but it's locked by the parent controls. Even if I knew the password for the account (NNID), 
I couldn't log in because the console was offline. To set up the Internet connection, I also needed the PIN 
to bypass the parental controls. 

Earlier in the post, I mentioned a website to help reset the parental controls. But I couldn't use it: In order to reset it,
you need the 'Inquiry Number', which can only be accessed through the Parental Controls app (which can only be accessed from
the Wii U Menu). So my only option to get into the Wii U menu was to guess the parental control PIN and create a new account. 

**Update 2024-04-09: There are [new ways](https://github.com/GaryOderNichts/recovery_menu) to reset the parental controls by using a raspberry pi pico.**

Luckily the PIN is only 4 digits, in the worst case I have to try 9999 combinations. Annoying, but with enough 
time it can be solved. I gave it a few tries, but moved on to the other consoles. When I came back later, 
I systematically tried all birthdates (this reduced the number of possible PINs to 366!) and after ~30 minutes and 
~250 tries I was in. Apparently the previous owner's birthdate is November 26th :) 

Knowing the PIN, I was able to create a new account and remove the parental controls. Everything else seemed to work 
fine. Another saved console!
#### Console 3

This console looked very promising at first. I turned it on and it didn't lock up, but after a few seconds of opening 
the Wii U menu, I got an error code of `160-0103`. I tried a few more times, but the console kept freezing or giving 
me an error code. Most likely the emmc has died. RIP console 3 =( 

**Update 2024-04-09: There are now [ways to recover](https://gbatemp.net/threads/ultimate-wii-u-troubleshooting-guide-system-memory-error-160-0103-stuck-wii-u-screen-stuck-factory-reset-black-screen-after-stuck-update.642339/) the consoles with these error.** 

#### Console 4

The next console didn't output anything over HDMI, but when I tried a composite cable, it worked! I was able to pair
 the gamepad and change the output back to HDMI. This console also seemed to work fine. So far, 3 out of 4 consoles
 working! I have reached my goal of three saved consoles :)

#### Console 5

This console just worked. Well. Maybe this console will have some other problems after playing for a few hours, 
but so far it looks like a normal working Wii U! 4 out of 5!


#### Console 6

Another locked console like console 2. This time it only took me a few tries because the PIN was 0505, which seemed 
like an obvious pattern to try. Probably the old console owner's birthday was May 5th :D 5 out of 6!

#### Console 7

The next console didn't output anything over HDMI. I tried the same "composite" trick as on console 4, but still had no 
success. Normally having no video output is fine as long as you can use the gamepad. But to connect the gamepad to the 
console, you have to enter a PIN that is displayed on the TV.  This reminded me of console 6 and 2, but this time I had to guess the
 connection PIN. Bruteforcing this was a bit more painful though, as each attempt took about 10 seconds. The other problem was that 
 the sync dialog closes after some time and I have no way to check if the sync dialog is even open. Worst case, I try to connect, but the console doesn't even wait for a connection.
 
 **Update 2024-04-09: It is [possible](https://github.com/GaryOderNichts/recovery_menu/issues/19) to obtain the PIN if you have a raspberry pi pico.**

I mentioned [drc-sim](https://github.com/rolandoislas/drc-sim) as a possible solution for console 1. To emulate a gamepad, 
you have to go through the same connection process with the PIN. I thought I might be able to automate the bruteforcing, 
but in the end I spent about 5 hours fighting with several ubuntu vms without success. I went back to manual bruteforcing.

After testing ~30 PINs, I remembered that some consoles have a key combination to reset the video mode. Unfortunately, the 
Wii U doesn't seem to have such a feature, but I came across a random Youtube video that showed how to fix a "no picture on 
TV" problem. Since they had already synced a gamepad, they suggested trying different video output modes until it worked. 

That was when I realized that the Wii U has more output options than just HDMI and composite. It also has component output.
 The problem is that I don't have a component cable to test this. But what I do have is an "AV Multi Out" to HDMI adapter 
 for my Wii, and I gave it a try. The first monitor I tried showed an "invalid hdmi signal" error, but on my slightly more
 expensive one I finally saw the Wii U menu! I checked the settings and apparently the console was set to non-HDMI and 576i.
 After setting it to HDMI and 1080p, the console 
worked perfectly! 6 out of 7!

#### Console 8

The last console had exactly the same problem as console 7. After connecting it using the "AV Multi Out" to 
HDMI adapter, I was able to change the video output to HDMI. Everything seemed to work fine. 7 out of 8 consoles work!

# Conclusion

In the end, it all worked out much better than I expected. I was able to "rescue" 7 of the
 8 consoles without even opening any of them. The description of the consoles looked very 
 promising and I was very lucky with the outcome. This could have easily been a waste of 119€, 
 ending up with a huge pile of broken Wii Us. ~~Now it was a waste of 119€ with a huge pile of 
pile of working Wii Us.~~

If you're looking for a "broken" console, here are some tips:
- If you enjoy entering 9999 possible PINs, buy a "locked" console.
- In my experience, the "console has no video output" problems are not a real problem, you just need the right 
just need the right cables and monitors to "fix" them.
- Be prepared to end up with a pile of electronic junk.
- Only spend the amount of money you are comfortable with losing.
- Remember that you also need a gamepad (which is even more expensive than the consoles).

I even found another "no video ouput" console on ebay for <20€ that sounded very promising, 
but I decided not to buy it even though it probably works fine. I don't even know what to do
 with the 9 working consoles I currently own. 
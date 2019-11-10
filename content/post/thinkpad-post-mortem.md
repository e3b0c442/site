---
date: 2019-11-10
title: "The ThinkPad experiment: a post-mortem"
tags: ["Apple", "Lenovo", "ThinkPad", "MacBook Pro", "opinion"]
---

After a week of tweaking, fixing, and just generally trying to get up to full productivity with it, I packed the ThinkPad X1 Extreme up to return this morning.

There are some really nice things about this computer. The keyboard is top-notch. The computer seems to be very solid and durable (though this is hard to measure after just one week). The screen is utterly amazing in its sharpness, color accuracy and saturation, and contrast and brightness, and I ended up suprising myself with how much I used the touch functionality. On paper, and at first glance, it is a great machine. I just couldn't get up to 100% productivity on it.

I can't really quantify how much of my time I spent trying to get things just right. I'll admit, I'm a bit of a perfectionist in that regard, but I feel like I have a right to be. I expect to be productive on my laptops; I have a desktop at home for when I want to tweak.

So, what exactly went wrong? At the end of the week, there were three blockers:

* In Linux, I was still trying to get the trackpad behavior just right. In particular, there was a very annoying issue with the trackpad not correctly detecting my thumb, making click-and-drag nearly impossible. Given enough time, I probably could have gotten it there, but it was a painful process.
* Because of the trackpad issue, I probably spent more time than I otherwise would have trying to get my Yubikey working seamlessly in Windows. I thought I was finally there last night, through a convoluted process involving no less than three tools that I wouldn't otherwise actively use, that all needed to be running. And then I discovered that the SSH agent wasn't forwarding even though I had configured it to, and by that point I had lost my will to continue, because...
* UI performance was downright awful. It took me a couple of days to really notice it, but when I did, it was [like the glass-breaking scenes from How I Met Your Mother](https://www.youtube.com/watch?v=BcmFkmDcFac). It became like nails on a chalkboard, and I spent the better part of Friday and Saturday trying to resolve it through a series of voltage and performance tweaks. I knew Windows could do better; Windows runs like a champ on my desktop which has similar specs. The breaking point, though, was when I needed to pick my old MacBook Pro up to do something while I was waiting for Windows to reboot on the other machine. My MacBook Pro is 6 years old, and compared to the ThinkPad, it felt brand new. The UI was responsive, animations were silky-smooth... I couldn't believe how stark the difference was. Would my little-MacBook-Pro-That-Could keep up with the ThinkPad on compute-intensive tasks? No. But at least for now, I have the option of offloading most of those onto my desktop at the sacrifice of some convenience, so to me the day-to-day user experience is far more valuable.

The first issue: I could have resolved with enough time. The second issue: was annoying, but working and I'm sure I could have figured out the agent forwarding. The third issue: despite the hours I spent, I didn't feel like I made much ground, and I'm just not sure I could have fixed it in the first place. There were three likely culprits: thermal transfer issues, bad drivers, and a screen that was just too much for the GPU. I could have only helped one of those.

Even if I had managed to get across the finish line with all of these problems... what if I needed to reinstall the OS for some reason? Then I need to figure it all out again (or figure out how to effectively back up and restore it).

In addition to those show-stoppers, there were a number of minor annoyances, many of which I expected. The trackpad just wasn't as good as a Mac trackpad. I couldn't open the screen with one hand, and it didn't stay put through the whole range of being open if I let go. I sorely missed some of the Apple ecosystem integrations (Messages, I'm talking to you). 

Before I made the decision to pack everything up, I thought back on the reasons I decided to attempt this switch in the first place, and the keyboard is the only reason on that list that still holds up. On upgradeability, I realized it was a false justification, because I've had my MacBook Pro now for 6 years, and neither memory nor storage is the reason I'm looking to upgrade. On the value proposition, if I applied a monetary value to the amount of time I spent this week just trying to get to my previous level of productivity, I would have nearly wiped out the difference. On the matter of hardware/software and ecosystem integration, I'm not proud to say I was flat-out wrong.

In my previous article, my thesis was "the value proposition of a Mac notebook no longer makes sense." 

Apparently, it still does.
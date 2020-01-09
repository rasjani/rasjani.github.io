---
layout:	post
title:	"Saga of two Sierra's"
date:	2018-02-18
description: When things go wrong and aftermath is epic
tags: [mac, rant]

---

Let me start off by admitting - I'm myself to blame here. I'm idiot and lazy penny pintcher who should either buy new hardware or rtfm.

So, some weeks ago I read somewhere that the soundcard I have (Fast Track Pro from M-Audio but then bought by Avid and who hasn't update drivers in ~4 years) works with High Sierra. Great, lets update. And this is where it all went wrong...

Yes, there where reports of Fast Track Pro to work in High Sierra *(10.13.1)* ... But at the time I was installing, newer version was out. Changelog said something along the lines "Updated compatibility with some 3rd party sound cards". And installed it got and ofcourse it broke the compatibility!

So, next I spent 3-4 hours to save all the documents to my NAS. I could really do with a faster disk.

Next stop App Store - and ofcourse you cannot find Sierra installer via normal search - you have to search the store link from your favourite search engine. Got it eventually. Download takes roughly about 15 minutes but nothing happens?! Commence to download it few times again and same thing. Only to realize that download did work but for some reason when the installer started, it showed an error dialog and that wasn't brought to the front.

Interesting dialog. Apparently High Sierra is too new operating system to support running the Sierra installer.

![](/img/tooold.png)#TooOldSo, what next ? Recovery Mode. Nope, didn't work but more about this later. Even though Apple has said that the recovery mode should always install the osx version the device came with, recovery mode showed that it wants to reinstall High Sierra. USB Stick might be an option then. After the command line tool initially said the recently downloaded Sierra installer was not valid and yet-an-another re-download, i got the stick! Mind you, few times bailing due to my own error - had the usb stick open in finder and shell open there too. Off i go to boot menu and as its already apparent with my luck - usb did not show up! Maybe the issue is that the usb is just not bootable or partitioned wrong. Some pages mentioned that the usb stick should have GUID partition. Quick reinstall to the stick and still the same issue. Tried even a GUI tool to do it, thinking that maybe I missed some crucial step along the way.

At this point I'm totally lost at what to try out next except maybe finishing few quests at Witcher3. Queue few hours and monsters later, out of thin air, fellow musician sents a message that mentioned that different keyboard shortcuts provide different targets to install via internet recovery mode. Oh my, did I hit paydirt?! I went with install the OS X laptop came with or the latest available and after about 30 minutes I'm greeted with Mountain Lion installer - which doesn't recognize the SSD at all. Grown man is about to cry and it won't be pretty.

At this point I have one option left and I'm willing to take it. Install High Sierra again and hope it's not the one with sound card compatibility fixes. While I'm trying that route, I want to say yuge fuck you to Apple for making a excellent experience out of downgrading and avid for keeping up with their drivers.

The Saga continues aka removing apfs partition first finally let me to install Mountain Lion! Let's how things will go downhill from here

![](/img/9hours.jpeg)After about 4 hours , two of which I was already asleep- my trusty laptop has been blessed with newly installed Mountain Lion! Happy happy joy joy and off to the AppStore once again to download Sierra since apparently the one I have on usb is missing The Executable. And because everything else has worked so far, AppStore craps out an unknown error when I try to download Sierra. El Capitan it is then about hour later and once again I try to download Sierra from AppStore to no avail. Download button just turns dark like in between double click. Atleast with El Capitan I can run Logic and sound card works so I start to install my environment. One of the last pieces of software I install is Firefox as it didn't support ML at all so I'm almost there. And for the fuck of it, I tried once more to go to Apple support page to try and download Sierra.

It worked!

It really did work! Saga is at its end. I have a working - latest OS X installed with working sound card and daw around 6am!

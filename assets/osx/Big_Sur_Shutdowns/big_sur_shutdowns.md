---
title: Repetitive Shutdowns under BigSur Mac OS 11
layout: post
date: 28/11/2020
---

# Your macbook powers off randomly

## Fix Summary

Normal boot
```
sudo fdesetup disable
fdesetup status => should read FileVault is Off
```
Boot in recovery mode
```
csrutil disable
csrutil authenticated-root csrutil
mount -uw /Volumes/"SYSTEM_HDD_NAME"
cd /Volumes/"SYSTEM_HDD_NAME"/System/Library/Extensions
mv AppleThunderboltNHI.kext AppleThunderboltNHI.kext.BAK
rm -rf /System/Library/Caches/*
kmutil install -u --force --volume-root /Volumes/"SYSTEM_HDD_NAME"/System/Library/Extensions
bless -folder /Volumes/"SYSTEM_HDD_NAME"/System/Library/CoreServices —bootefi --create-snapshot
```

Boot normaly

Done

## Symptoms

- You are working and suddenly the machine powers off. 
- The apple logo (and maybe the keyboard retro-lighting) stays on for a few seconds. 
- Then the machines powers off.
- When you press the power key, the machine boots normally and no error message appears.

If this has happened to you, you are probably at the right place.

## What happened ?


For certain Macbooks, there is a faulty component on their logic board (see [this Stack Echange post](https://apple.stackexchange.com/questions/306714/macbook-pro-15-retina-mid-2014-random-shutdowns), [this MacRumor thread](https://forums.macrumors.com/threads/help-updated-to-macos-10-12-4-mbp-randomly-shuts-off.2039446/page-6), [this change.org petition](https://www.change.org/p/apple-admit-to-the-problem-of-macbooks-randomly-shutting-down-and-recall)) that forces the machine into deep sleep.

Changing this component requires skill, time and money that note everyone may have. If you macbook is under warranty, I would recommend you to go to the nearest AppleStore to get it fixed (in some case, it has resulted to have been not enough, and the presented solution was also required).

Otherwise this guide is for you.

Here, we are going to prevent the file *AppleThunderboltNHI.kext* from being used during boot. This file is a driver for connecting Ethernet through thunderbolt connector. It is also faulty on certain macbook and contributes in the go to sleep mode behaviour by wrongly changing the voltage of the CPU.

**WARNING**

The solution will permanently require you to deactivate SIP and FireVault. There is no other way here untill Apple does something about it.


## Detailed solution

Here is the cooking recipe for solving the problem under Big Sur:

1. Boot normally

2. Play a video in the background, for [instance this one](https://www.youtube.com/watch?v=GEZhD3J89ZE) to keep the CPU busy while we work.

3. Open a Terminal

4. Deactive FileVault as follows:
	```
	sudo fdesetup disable
	```

	And press the "Enter" key to execute the command.

	> you can get you username using the command
	> `whoami`
	> your password is the session password

5. Now we have to wait for the hardrive to be de-encrypter. To check on the process you may run from time to time:

	```
	fdesetup status
	```

	> for those using brew, I recommend using `watch fdesetup status`

6. When the decryption processus has reached `100` or when you will read `FileVault is Off`, we can move on.

7. Reboot into recovery mode (hold down `"cmd" + "R"` keys when the machine powers up)

	> You should see the following screen at some point

	![Recovery Mode Window](recovery_mode_window.png)

8. On the top menu bar, goto Utilities > Terminal

9. Now we are going to turn off SIP
	```
	csrutil disable
	csrutil authenticated-root csrutil
	```

10. We are going to mount the root HDD

	> If you don't know what is your root HDD, you can run :
	> ```
	diskutil apfs list
	```

	> Your root HDD is the one whose (Role) in the label *APFS Volume Disk* matches "(System)"
	![Root HDD](root_hdd.png)

	To mount in writing mode the HDD, use the following command and susbtitute `SYSTEM_HDD_NAME` with the name for you root HDD:
	```
	mount -uw /Volumes/"SYSTEM_HDD_NAME"
	```

	>In the previous image SYSTEM_HDD_NAME was "HD\_macbook".
	>To avoid issues with spaces in a name, use quotes "" around the name, for instance "HD Macbook"
	>>You can also escape spaces using backslashe -> HD\ Macbook

11. Now let's move to the folder */Volumes/"SYSTEM_HDD_NAME"/System/Library/Extensions*, deactivate the faulty driver and clear the caches.
	Don't forget to change `SYSTEM_HDD_NAME` for the actual name of your HDD:

	```
	cd /Volumes/"SYSTEM_HDD_NAME"/System/Library/Extensions
	mv AppleThunderboltNHI.kext AppleThunderboltNHI.kext.BAK
	rm -rf /System/Library/Caches/*
	```

12. We have to update the kext cache and select the snapshot for the next boot:
	```
	kmutil install -u --force --volume-root /Volumes/"SYSTEM_HDD_NAME"/System/Library/Extensions
	bless -folder /Volumes/"SYSTEM_HDD_NAME"/System/Library/CoreServices —bootefi --create-snapshot
	```

13. We will not enable SIP anymore. Enabling SIP has proven to create boot loops after modyfing operating system files.

14. You are done and can reboot in normal mode.

15. (Optional) reactivate FileVault (may not work)

	```
	sudo fdesetup enable
	```

	Or from the [System Preferences GUI](https://support.apple.com/fr-fr/HT204837).

16. **If you install any OS updates**, it is very likely that **you will need to redo all the steps.**

17. To check that it worked, you can have a look at the *About This Max > System Report > Hardware > Thunderbolt* and you should read "No driver loaded". You can also run the folowing command

	```
	kextstat | grep NHI
	```

	and you should not see any mention of "AppleThunderboltNHI.kext"

## Resources

- The solution for OSX Catalina can be found here:
[https://outluch.wixsite.com/rmbp-crash](https://outluch.wixsite.com/rmbp-crash)
> However the step are incomplete for Big Sur
> We can't use "sudo mount -uw" directly to modify "AppleThunderboltNHI" as in the guide because of read-only restrictions
- The reason why the system is locked in read-only can be found here:
[https://developer.apple.com/forums/thread/649832]
> It shows that it worked under Catalina but no more under Big Sur
- This reference shows the updated answer for Big Sur
[https://developer.apple.com/forums/thread/666567](https://developer.apple.com/forums/thread/666567)
- The previous post is also found on Macrumors forum: [https://forums.macrumors.com/threads/big-sur-and-applethunderboltnhi-kext.2267818/](https://forums.macrumors.com/threads/big-sur-and-applethunderboltnhi-kext.2267818/)
> for "csrutil authenticated-root disable", FireVault on "Mackbook HD" (root HDD) must be turned off.
- How to turn FireVaul off was found here: 
[https://www.whileifblog.com/2017/07/09/macos-manage-filevault-from-command-line/](https://www.whileifblog.com/2017/07/09/macos-manage-filevault-from-command-line/)
- Another interesting read about making permanent change to the OS and issues with kmutil
[https://egpu.io/forums/mac-setup/macos-up-to-11/](https://egpu.io/forums/mac-setup/macos-up-to-11/)
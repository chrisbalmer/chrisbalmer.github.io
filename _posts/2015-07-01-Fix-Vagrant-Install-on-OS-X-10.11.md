---
layout: post
title: Fix Vagrant Install on OS X 10.11
categories: [osx]
tags: [vagrant]
fullview: true
---

When you upgrade to OS X 10.11, items from `/usr/bin/` are moved to `/Previous System/usr/bin/`. I believe this is part of the new "rootless" security feature in OS X 10.11. Vagrant is one of the executables moved. To fix this, you will need to disable rootless and then reinstall Vagrant.

First make sure there aren't other boot-args that you will overwrite with this process by running:

    nvram -p | grep boot-args

If you end up with any results, you'll want to adjust the following commands as needed to preserve the settings.

To disable rootless, run ([source](http://forums.macrumors.com/threads/rootless-kernel-level-protection.1890654/#post-21430368)):

    sudo nvram boot-args="rootless=0"
    # Save work before the next line
    sudo reboot

Once the reboot completes, reinstall Vagrant. Now your `vagrant` command should work again. Settings, box files, etc all appear to be intact after the OS upgrade and Vagrant reinstall.

After you're done, if you wish to restore rootless you can run ([source](http://forums.macrumors.com/threads/rootless-kernel-level-protection.1890654/#post-21435567)):

    sudo nvram -d boot-args
    # Save work before the next line
    sudo reboot

Alternatively you may be able to just copy over the Vagrant binary after disabling rootless. But if other files from Vagrant end up in one of the `/Previous System/` folders, it may not work. I opted to just reinstall to be sure.

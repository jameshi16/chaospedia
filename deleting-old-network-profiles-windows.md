---
description: Ah yes, I've connected to StarBucks Public Wifi 1239
---

# Deleting old network profiles (Windows)

Sometimes, Windows creates multiple network profiles for the same WiFi connection even though no one has reconfigured the router, your laptop, or anything in-between.

There is no built-in way to remove these extra profiles fromthe GUI. Instead, use RegEdit: navigate to `Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\NetworkList\Profiles`

Under "Profiles" there should be a bunch of keys that you can go through one by one to either delete them, or observe their contents. I like to delete all the old profiles (i.e. anything less than StarBucks Public Wifi 1239), and rename the `ProfileName` value to just "StarBucks Public Wifi".

After a restart, the changes should be reflected.

---
layout: post
title: Draft post
comments: false
---

In my previous post, we looked at how to produce a list of the drive letters protected by volume shadow copy on a server, and how how many shadow copies are being retained per volume.  In that case we did it with a calculated property or two.

Before I'm done I want to accomplish the following things
1) Find the weak spots in the protection coverage (we're well on out way to that now!)
2) Mitigate those weaknesses - Adjust the storage settings to get more copies on certain volumes.
3) Formalize things a bit - Let's try to make a tool!

This time around let's get a little more formal

$Storage = Get-CimInstance Win32_ShadowStorage

We can get the information about the storage, but we can't seem to be able to *change* it.
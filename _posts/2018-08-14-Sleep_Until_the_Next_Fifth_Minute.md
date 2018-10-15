---
layout: post
title: Sleep until the Next Fifth Minute
comments: true
toc: true
toc_icon: "cog"
---

Tim Curwick of the [Mad With PowerShell](https://www.madwithpowershell.com/) blog deserves credit for this, but I don't think he's ever blogged about it.

```PowerShell
start-sleep -seconds (300 - (get-date).TimeOfDay.TotalSeconds % 300)
```

This sleeps until the next 300th second.

```PowerShell
(get-date).TimeOfDay.TotalSeconds % 300
```
Gives us the remainder of the total seconds elapsed today, divded by 300, or how many seconds it's been since the last multiple of 300 seconds.

Subtracting that from 300 gives us how many seconds until the next multiple of 300.

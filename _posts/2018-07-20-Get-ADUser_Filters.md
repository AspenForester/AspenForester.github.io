---
layout: post
title: Get-ADUser Filters
comments: true
---
The filters for Get-ADUser are weird, and perhaps poorly, or more generously, "under-" documented.  There's the about_ActiveDirectory_Filter topic, and there are examples in the command's help.  There's [this Technet wiki page](https://social.technet.microsoft.com/wiki/contents/articles/28485.filters-with-powershell-active-directory-module-cmdlets.aspx) maintained by [MVP Richard Mueller](http://www.rlmueller.net/), and any number of other blog posts.  But with so many options, it's nearly impossible to provide examples for every combination.
  
Getting the filter right on the first attempt remains as elusive as getting a Regular Expression right on the first try.
  
For example `Get-ADUser -filter "enabled -eq '$True'"` works just fine.  But what if you want to narrow that down further to enabled user accounts with some value for the HomeDirectory?  You can't just add to that.  `"enabled -eq '$true' -and HomeDirectory -ne "$null"'` does not work!  Instead, you have to completely shift gears and end up with `{(enabled -eq $true) -and (HomeDirectory -ne "$null")}`. Of particular note is the requirement to enclose the `$null` in double quotes, but not the `$true`.
```PowerShell
Get-ADUser -filter {(enabled -eq $true) -and (HomeDirectory -ne "$null")}
```
Post a comment if you have a different filter format that delivers the same results.

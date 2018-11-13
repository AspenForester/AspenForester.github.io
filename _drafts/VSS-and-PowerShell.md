---
layout: post
title: Draft post
comments: false
---

# Volume Shadow Copies and PowerShell
I manage a number of files servers, each of which has a number of volumes that have user data on them.  In addition to conventional backup and recovery procedures, I also enable Volume Shadow Copies, so that the end users or the service desk can perform trivial file restores without escalating a ticket to the backup and recovery team (of which I am a member!)  The size of the data volumes vary, as does the level of activity on each.  Plus, to throw in one more variable, some of the volumes contain files that are, individually, very large, digital videos, for example.

The management interface for shadow copies lets you select the drive on which shadow copies are stored, the "on" parameter in vssadmin, and how much of that drive to use for protected volume, ether as a percentage of the or as an absolute number of mega-, giga-, terabytes. 

Once enabled, and let's assume for the moment that we're using the default schedule, VSS collects a point-in-time snapshot twice a day at 7am and noon.  What I have found, in my environment, is that with all those variables at play, you  end up with different numbers of copies being retained for each protected volume.  From a customer service perspective I would like to provide the same number of copies, or put another way, the same 'depth' of history.

So, with this in mind I set out to come up with a way to determine the NUMBER of shadow copies that are being held for each volume on a given server. 

I started with knowing that I could get a list of the shadow copies on a server with

```powershell
Get-WMIObject Win32_ShadowCopy
```

But if you look at the data for one shadow copy you can see that it identifies the volume with a UID path in the property "VolumeName"
```powershell
VolumeName         : \\?\Volume{220a4c15-1da3-4c12-8992-e7ff6b68a1e3}\

PS C:\Windows\system32> $shad = (Get-WmiObject win32_shadowcopy)[0]
PS C:\Windows\system32> $shad


__GENUS            : 2
__CLASS            : Win32_ShadowCopy
__SUPERCLASS       : CIM_LogicalElement
__DYNASTY          : CIM_ManagedSystemElement
__RELPATH          : Win32_ShadowCopy.ID="{8D75F263-A3D7-4AC3-AB51-1A0178D42552}"
__PROPERTY_COUNT   : 28
__DERIVATION       : {CIM_LogicalElement, CIM_ManagedSystemElement}
__SERVER           : ServerA
__NAMESPACE        : root\cimv2
__PATH             : \\ServerA\root\cimv2:Win32_ShadowCopy.ID="{8D75F263-A3D7-4AC3-AB51-1A0178D42552}"
Caption            :
ClientAccessible   : True
Count              : 1
Description        :
DeviceObject       : \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy49
Differential       : True
ExposedLocally     : False
ExposedName        :
ExposedPath        :
ExposedRemotely    : False
HardwareAssisted   : False
ID                 : {8D75F263-A3D7-4AC3-AB51-1A0178D42552}
Imported           : False
InstallDate        : 20150501120034.213457-300
Name               :
NoAutoRelease      : True
NotSurfaced        : False
NoWriters          : True
OriginatingMachine : ServerA.domain.local
Persistent         : True
Plex               : False
ProviderID         : {B5946137-7B9F-4925-AF80-51ABD60B20D5}
ServiceMachine     : ServerA.domain.local
SetID              : {4EF826D1-D1CA-46AA-90D7-023041C8C80D}
State              : 12
Status             :
Transportable      : False
VolumeName         : \\?\Volume{220a4c15-1da3-4c12-8992-e7ff6b68a1e3}\
PSComputerName     : ServerA
```
While consistent, that UID isn't very friendly.  Get-Volume will give us the linkage between drive letters and the UID, if we want it. 
```powershell
PS C:\Windows\system32> get-volume -DriveLetter C | fl


DriveLetter     : C
DriveType       : Fixed
FileSystem      : NTFS
FileSystemLabel : C Drive
HealthStatus    : Healthy
ObjectId        : \\?\Volume{8f79d014-b218-11e4-80c8-806e6f6e6963}\
Path            : \\?\Volume{8f79d014-b218-11e4-80c8-806e6f6e6963}\
Size            : 42580570112
SizeRemaining   : 18876383232
PSComputerName  : 
```

Let's say for this discussion we do.  So let's stash that away for later reference.
  
```powershell
$volumes = Get-Volume
```

And while we're at it, let's store the collection of ShadowCopy objects, too.  
```powershell
$shadows = get-wmiobject win32_shadowcopy
```

So we want to count the number of shadow copies for each volume. For this exercise I decided to use calculated properties, but for a production script, I probably would create a custom object for the output.
```powershell
$shadows `
    | select volumename,
             @{n="drive";e={$vol=$_.volumename;($volumes | where {$_.path -eq $vol}).DriveLetter}},
             @{n="Date";e={$_.converttodatetime($_.installdate)}} `
    | Group-Object -Property Drive

Count Name Group                                                                                                             
----- ---- -----                                                                                                             
   61 Z    {@{volumename=\\?\Volume{220a4c15-1da3-4c12-8992-e7ff6b68a1e3}\; drive=Z; Date=5/1/20...
   10 Y    {@{volumename=\\?\Volume{1b7da7ce-3b66-4534-9596-02f96c510df6}\; drive=Y; Date=6/8/20...
   31 U    {@{volumename=\\?\Volume{c12ffb10-8a6f-4041-b326-e0f0c8aa14b2}\; drive=U; Date=5/22/2...
    6 T    {@{volumename=\\?\Volume{db23afab-ecce-11e4-80cc-005056b52c40}\; drive=T; Date=6/10/2...
   11 V    {@{volumename=\\?\Volume{86130522-ffe6-4254-8d86-589d508ea67b}\; drive=V; Date=6/5/20...
    9 H    {@{volumename=\\?\Volume{5f22f6d7-e882-11e4-80cc-005056b52c40}\; drive=H; Date=6/8/20...
```

So you can see that in this case, the T and H volumes are only keeping 6 and 9 previous versions respectively. We can move on to look at the current storage settings. That's Up Next!
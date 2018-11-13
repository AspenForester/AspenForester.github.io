```PowerShell
$Computers = 'Server1','Server2' # Generate your own list of servers

foreach ($computer in $computers)
{
    $System = Get-WmiObject -Class Win32_OperatingSystem -ComputerName $Computer

    $BornOn = $System.ConvertToDateTime($System.installDate)

    [PSCustomObject]@{
        ComputerName = $Computer
        InstallDate = $BornOn
        OS = $System.Caption
    }
}
```
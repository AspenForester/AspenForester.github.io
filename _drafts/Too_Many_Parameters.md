---
layout: post
title: Replacing Parameters with a Data File
comments: true
---

Sometimes eight required parameters are too many when your end user only wants to pick between taking action on their UAT environment or their production environment.  In cases like that it might be more desirable to provide the module you wrote for them with a data file that contains the pertinent information.

In my case I was delivering two scripts in the form of two functions bundled as a module.  We have a simple internal NuGet repository, and being able to deliver updates to that location, and showing them how to do Update-Module made life easy when it came to working out kinks 
with their process and the scripts.

I decided to replace the eight mandatory parameters with a single "Environment" parameter.  I control the content of $Environment [validateSet()]. Then a datafile with all the required info, keyed with the stop and actions, and the environment names fill in the information that once came from the eight parameters.  The two functions Start-MyThings and Stop-MyThings (Not the real names, BTW) each start by loading the datafile from the $psscriptroot location.

If we break the logic down there's a "start" and "stop" for each environment.  So the datafile ends up looking like this:

```
Environments = @{
    UAT        = @{
        Stop  = @{
            ServersMQ     = "ServerA"
            ServicesMQ    = 'MQ_Installation1'
            ServersMB     = ("ServerB", "ServerC")
            ServicesMB    = ('MQSeriesBrokerWMBUAT1', 'MQSeriesBrokerWMBUAT2')
            ServersQP     = "ServerD"
            ServicesQP    = ('qpea', 'qpcfg', 'qpmon')
            ServicesQPasa = ('qpcgateway', 'qphs', 'qpes', 'qpts', 'qpas')
            Myweb        = 'http://Webuat/foo/logon/logon.xhtml'
        }
        Start = @{
            ServersMQ     = "ServerA"
            ServicesMQ    = 'MQ_Installation1'
            ServersMB     = ("ServerB", "ServerC")
            ServicesMB    = ('MQSeriesBrokerWMBPROD1', 'MQSeriesBrokerWMBPROD2')
            ServersQP     = "ServerD"
            ServicesQP    = ('qpea', 'qpcfg', 'qpmon')
            ServicesQPasa = ('qpas', 'qpts', 'qpes', 'qphs', 'qpcgateway')
            Myweb        = 'http://Webuat/foo/logon/logon.xhtml'
        }
    }

    Production = @{
        Stop  = @{
            ServersMQ     = "ServerE"
            ServicesMQ    = 'MQ_Installation1'
            ServersMB     = ("ServerF", "ServerG")
            ServicesMB    = ('MQSeriesBrokerWMBPROD1', 'MQSeriesBrokerWMBPROD2')
            ServersQP     = "ServerH"
            ServicesQP    = ('qpea', 'qpcfg', 'qpmon')
            ServicesQPasa = ('qpcgateway', 'qphs', 'qpes', 'qpts', 'qpas')
            Myweb        = 'http://Webprod/Web/logon/logon.xhtml'
        }
        Start = @{
            ServersMQ     = "ServerE"
            ServicesMQ    = 'MQ_Installation1'
            ServersMB     = ("ServerF", "ServerG")
            ServicesMB    = ('MQSeriesBrokerWMBPROD1', 'MQSeriesBrokerWMBPROD2')
            ServersQP     = "ServerH"
            ServicesQP    = ('qpea', 'qpcfg', 'qpmon')
            ServicesQPasa = ('qpas', 'qpts', 'qpes', 'qphs', 'qpcgateway')
            Myweb        = 'http://Webprod/Web/logon/logon.xhtml'
        }
    }
}
```

I had originally written the functions with all those parameters, and the easiest way to swtich to consuming the data file was to just start by creating the variables that had once come from the parameters.

First, import the datafile
```
$ImportedData = Import-PowerShellDataFile -Path $PSScriptRoot\Configuration.psd1
```
We get `$Environment` as a parameter, so we can do the following
```
$StopSettings = $ImportedData.$Environment.Stop
```
And if we assume they chose "UAT", it leaves us with a simple Hashtable for Stopping the UAT environment.

Then we can iterate over the keys and create the variables that once came from the parameters, and we don't need to do anything to the rest of the code.
```
foreach ($key in $StopSettings.Keys)
    {
        New-Variable -Name $key -Value $StopSettings.$key -Verbose:$false -whatif:$False
        Write-Verbose "New Variable $key = $($StopSettings.$key)"
    }
```

From here the functions stop or start the various services, on the various machines in the right order, waiting for the URL in $MyWeb to be responsive or not as needed.  If the users of the function decide that they want to apply this to a Dev Environment, we can add that info to the configuration data file easily.  The consumers of these tools are not pros at PowerShell, and thus I decided to keep the configuration file "hidden" within the Module, but I think that's the rare case.  More often than not you'd want the configuration data separated from the implementation code.

If you look at [Jaykul's Optimize-Module Gist](https://gist.github.com/Jaykul/176c4aacc477a69b3d0fa86b4229503b#file-optimize-module-ps1 "Optimize-Module"), you can see how he implements an optional data file as an example.

Data files like this have many uses, and seem to support good DevOps practices.  I'm looking forward to re-examining some of my other Scripts, Functions, and Modules, with this new tool in my kit.  I hope you are too!

P.S. I have a confession to make, and maybe you've spotted what I've done.  After talking to another, more PowerShell savvy coworker, I came to the realization that I could have implemented these service control tasks as a DSC configuration.
---
layout: posts
title: Get-WindowsFirewallBlocks
classes: wide
date: 2021-11-10
---

# Introduction  
I've had some exposure to Microsoft Defender here and there, but I was in a class with Microsoft recently where they were going over some more features in depth.  At one point, while discussing the firewall aspect, I asked if there was any good place to see logs of what Windows Firewall was blocking.  I was directed to two places; Event Viewer, and a static text file.  This post will be about the Event Viewer portion.

# The Logs  
If you open Event Viewer and expand the "Applications and Services Logs", then "Microsoft", "Windows", and finally "Windows Firewall With Advanced Security" you'll find the "Firewall" log.  
![GWFB1]({{ "/assets/images/GWFB_01.png" | absolute_url }})  
This log contains entries regarding firewall rule changes, network profile changes and so on.  It also contains entries under the ID '2011' where Defender *would* have notified the user that an application was blocked from accepting an inbound connection.  Instead that notification is not enabled (on the systems I'm talking about) so it is instead logged in Event Viewer.  
  
It's simple enough to filter the current log in Event Viewer and just look at the ones with Event ID 2011, but the actual message that's logged is where the good info is:  
![GWFB2]({{ "/assets/images/GWFB_02.png" | absolute_url }})  
Unfortunately if you're looking for a high level view of these logs it's kind of difficult.  Or worse yet, if there's a specific executable, or port that you're curious about, there's no way to filter the logs for that info.  Of course my first thought was Powershell.  
  
# Powershell Info Gathering
I first went to search the internet for how to view these logs in Powershell. I'm familiar with searching through the "Security", "Application" and "System" logs with "Get-EventLog" but wasn't sure about logs within "Applications and Services Logs."  One of the first places I landed had this nice one-liner. I just changed the Event ID to match what I was looking for:  
{% highlight Powershell %}
$Events = Get-WinEvent -FilterHashtable @{Logname="Microsoft-Windows-Windows Firewall With Advanced Security/Firewall"; id=2011}
{% endhighlight %} 
Now that I have an array full of events I could start to look at them and figure out where the info I want it.  
![GWFB3]({{ "/assets/images/GWFB_03.png" | absolute_url }})  
Pretty similar to what I see in the graphical Event Viewer. Looking deeper at the "Message" property I could see the info I want:  
![GWFB4]({{ "/assets/images/GWFB_04.png" | absolute_url }})  
Unfortunately I could also see that the "Message" property was a string object.  If I wanted the individual items listed within I'd have to parse the string object.  Another quick search and I found someone's example that they used the "ToXML" method of the original object to convert the whole entry to XML.  
![GWFB5]({{ "/assets/images/GWFB_05.png" | absolute_url }})  
Great, we should be able to work with this.  The only thing that stood out when exploring the XML data was that the "IP Version", "Reason" and "Protocol" were all numerical represented.  This would mean that this object, "System.Diagnostics.Eventing.Reader.EventLogRecord", is automatically converting those numeric values to pretty string values for us to look at. Well, two can play at that game.  
# Make A Class  
From looking at the info within a single event log I knew I'd want the date stamp, and then all of the info contained within the message itself.  I started writing down a custom Class with all of the properties I'd want. Like this:  
{% highlight Powershell %}
Class FirewallEvent {
    [DateTime]$DateTime
    hidden[String]$ReasonCode
    [String]$ApplicationPath
    [String]$IPVersion
    [String]$Protocol
    [Int32]$Port
    [Int32]$ProcessID
    [String]$User
}
{% endhighlight %} 
I marked the "ReasonCode" as hidden because so far the only example I've seen is "Windows Defender Firewall was unable to notify the user that it blocked an application from accepting incoming connections on the network." which get's translated as a '1'.  Now I've got an object with the properties I want, but I'm going to want to manipulate some of the input data when I create the object. We need a Class Constructor:  
{% highlight Powershell %}
Class FirewallEvent {
    [DateTime]$DateTime
    hidden[String]$ReasonCode
    [String]$ApplicationPath
    [String]$IPVersion
    [String]$Protocol
    [Int32]$Port
    [Int32]$ProcessID
    [String]$User
    FirewallEvent ([DateTime]$DateTime, [Array]$MessageInfo) {
        $this.DateTime          = Get-Date $DateTime -Format 'MM/dd/yy HH:mm:ss'
        $this.ReasonCode        = $MessageInfo[0].'#text'
        $this.ApplicationPath   = $MessageInfo[1].'#text'
        $this.IPVersion         = Switch ($MessageInfo[2].'#text') {
            1 {'IPv6'}
            0 {'IPv4'}
        }
        $this.Protocol          = Switch ($MessageInfo[3].'#text') {
            6 {'TCP'}
            17 {'UDP'}
        }
        $this.Port              = $MessageInfo[4].'#text'
        $this.ProcessID         = $MessageInfo[5].'#text'
        $this.User     = ([System.Security.Principal.SecurityIdentifier]($MessageInfo[6].'#text')).Translate([System.Security.Principal.NTAccount]).Value
    }
}
{% endhighlight %}      
[This](https://adamtheautomator.com/powershell-classes/) Adam The Automator post is how I got started with Classes so I'll just summarize what I did here and why.  Firstly, I saw I wanted the date stamp.  That was easy enough as it's a property of the original Event Log object:  
![GWFB6]({{ "/assets/images/GWFB_06.png" | absolute_url }})  
Then looking at the XML some more, the original "Message" text that I wanted is now an array of objects with two properties: "Name" and "#text".  
![GWFB7]({{ "/assets/images/GWFB_07.png" | absolute_url }})  
Since it's conveniently in array I could pass the entire array to my class constructor and just manually index through the array to assign the values I want.  The class constructor also gives me the opportunity to convert the original values to something else (see the Switch blocks for "IPVersion" and "Protocol").  Then I just had to look up a way to convert an SID to a readable username and I'm all set.  
# Put It Together  
Still working in VS Code I had the following:  
{% highlight Powershell %}
# Get the events
$Events = Get-WinEvent -FilterHashtable @{Logname="Microsoft-Windows-Windows Firewall With Advanced Security/Firewall"; id=2011}
# Define our class
Class FirewallEvent {
    [DateTime]$DateTime
    hidden[String]$ReasonCode
    [String]$ApplicationPath
    [String]$IPVersion
    [String]$Protocol
    [Int32]$Port
    [Int32]$ProcessID
    [String]$User
    FirewallEvent ([DateTime]$DateTime, [Array]$MessageInfo) {
        $this.DateTime          = Get-Date $DateTime -Format 'MM/dd/yy HH:mm:ss'
        $this.ReasonCode        = $MessageInfo[0].'#text'
        $this.ApplicationPath   = $MessageInfo[1].'#text'
        $this.IPVersion         = Switch ($MessageInfo[2].'#text') {
            1 {'IPv6'}
            0 {'IPv4'}
        }
        $this.Protocol          = Switch ($MessageInfo[3].'#text') {
            6 {'TCP'}
            17 {'UDP'}
        }
        $this.Port              = $MessageInfo[4].'#text'
        $this.ProcessID         = $MessageInfo[5].'#text'
        $this.User     = ([System.Security.Principal.SecurityIdentifier]($MessageInfo[6].'#text')).Translate([System.Security.Principal.NTAccount]).Value
    }
}
# Loop through and convert them to FirewallEvent objects
$LogObjects = ForEach ($Event in $Events) {
    $EventXML = [xml]$Event.ToXml()
    [FirewallEvent]::new($EventXML.event.System.timecreated.systemtime, $EventXML.event.EventData.data)
}
{% endhighlight %} 
Now I could look at "$LogObjects" and see my results:
![GWFB8]({{ "/assets/images/GWFB_08.png" | absolute_url }})  
Great! Now I can use Where-Object to filter them, or Export-CSV to save them to a .csv file.  
# Tell you what. Throw a little hot-rod red in there
Might as well use the filtering logic that I just created the other week for looking at my logging module logs.  Here's the first iteration of this function:
{% highlight Powershell %}
Function Get-WindowsFirewallBlocks {
    <#
    .Synopsis
    Gets the block messages from the "Windows Firewall with Advanced Security" Event Viewer Log
    .NOTES
    Version:        1.0
    Author:         C. Bodett
    Creation Date:  11/10/2021
    Purpose/Change: Initial function development
    #>
    [cmdletbinding()]
    [Alias("GWFB")]
    Param (
        [DateTime]$After,
        [DateTime]$Before,
        [DateTime]$Timestamp,
        [String]$Executable,
        [ValidateSet("IPv4","IPv6")]
        [String]$IPVersion,
        [ValidateSet("TCP","UDP")]
        [String]$Protocol,
        [ValidateRange(1,65535)]
        [Int32]$Port,
        [Int32]$ProcessID,
        [String]$User
    )

    Begin {
        $Events = Get-WinEvent -FilterHashtable @{Logname="Microsoft-Windows-Windows Firewall With Advanced Security/Firewall"; id=2011}

        $WhereArray = [System.Collections.ArrayList]@()

        If ($After) {
            [void]$WhereArray.Add('(Get-Date $_.Datetime) -gt $After')
        }
        If ($Before) {
            [void]$WhereArray.Add('(Get-Date $_.Datetime) -lt $Before')
        }
        If ($Timestamp) {
            [void]$WhereArray.Add('(Get-Date $_.Datetime) -eq $Timestamp')
        }
        If ($Executable) {
            [void]$WhereArray.Add('$_.ApplicationPath -Match $Executable')
        }
        If ($IPVersion) {
            [void]$WhereArray.Add('$_.IPVersion -Match $IPVersion')
        }
        If ($Protocol) {
            [void]$WhereArray.Add('$_.Protocol -Match $Protocol')
        }
        If ($Port) {
            [void]$WhereArray.Add('$_.Port -eq $Port')
        }
        If ($ProcessID) {
            [void]$WhereArray.Add('$_.ProcessID -eq $ProcessID')
        }
        If ($User) {
            [void]$WhereArray.Add('$_.User -Match $User')
        }

        $WhereString = $WhereArray -join " -and "
        $WhereBlock = [ScriptBlock]::Create($WhereString)

        Class FirewallEvent {
            [DateTime]$DateTime
            hidden[String]$ReasonCode
            [String]$ApplicationPath
            [String]$IPVersion
            [String]$Protocol
            [Int32]$Port
            [Int32]$ProcessID
            [String]$User
            FirewallEvent ([DateTime]$DateTime, [Array]$MessageInfo) {
                $this.DateTime          = Get-Date $DateTime -Format 'MM/dd/yy HH:mm:ss'
                $this.ReasonCode        = $MessageInfo[0].'#text'
                $this.ApplicationPath   = $MessageInfo[1].'#text'
                $this.IPVersion         = Switch ($MessageInfo[2].'#text') {
                    1 {'IPv6'}
                    0 {'IPv4'}
                }
                $this.Protocol          = Switch ($MessageInfo[3].'#text') {
                    6 {'TCP'}
                    17 {'UDP'}
                }
                $this.Port              = $MessageInfo[4].'#text'
                $this.ProcessID         = $MessageInfo[5].'#text'
                $this.User     = ([System.Security.Principal.SecurityIdentifier]($MessageInfo[6].'#text')).Translate([System.Security.Principal.NTAccount]).Value
            }
        }

    }

    Process {
        $LogObjects = ForEach ($Event in $Events) {
            $EventXML = [xml]$Event.ToXml()
            [FirewallEvent]::new($EventXML.event.System.timecreated.systemtime, $EventXML.event.EventData.data)
        }

        If ($WhereString) {
            $Results = $LogObjects | Where-Object $WhereBlock
        } Else {
            $Results = $LogObjects
        }
    }

    End {
        $Results
    }
}
{% endhighlight %} 
# Conclusion  
This is still just one piece of the puzzle.  There are also firewall logs located at "%SystemRoot%\system32\LogFiles\Firewall\pfirewall.log" if you have logging enabled there.  I will have to do some more testing and see what other information lies there.  
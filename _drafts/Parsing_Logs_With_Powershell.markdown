---
layout: posts
title: Parsing log files with Powershell
classes: wide
date: 2021-07-07
---

Early on when I first started using Powershell I was dealing with some firewall logs from a perimeter firewall.  They were exported from a SIEM in CSV format, which I appreciated, but the format within was odd and not condusive to what I was trying to do.  I was having a hard time wrapping my mind around how to deal with them in Powershell and some helpful person on Stackoverflow suggested I use regex to match each row, capture a value, and when the last row of a particular entry was matched, spit out a PSCustomObject with all the property/value pairs I wanted.  
I no longer have an example of this data handy but I can replicate something similar:

![Parsing-Logs2]({{ "/assets/images/Parsing_Logs_02.png" | absolute_url }})

Now you can see the issue. The way the SIEM exported the data I actually needed 3 consecutive rows to represent a single query output.  What I wanted was a single row with all of that data on it in a spreadsheet.  The advice I was given was to iterate through each row in the CSV, and use Regex with a capture group for each row to grab the data I wanted and store it in a variable. Then on the last row after matching with Regex, also return a PSCustomObject with all the values I had collected.  Something a little like this:  
{% highlight Powershell %}
$RE1 = '^(?<IpDst>(\d{1,3}\.){3}\d{1,3}),'
$RE2 = '\s*ip\.dstport\s*(?<IpDstPort>\d+)'
$RE3 = '\s*ip\.src\s*(?<IpSrc>(\d{1,3}.){3}\d{1,3}),\s*(?<Sessions>\d+)'

$Data = Get-Content $CSVpath | ForEach-Object {
    If($_ -match $RE1){$IpDst = $Matches.IpDst}
    If($_ -match $RE2){$IpDstPort = $Matches.IpDstPort}
    If($_ -match $RE3){
        [PSCustomObject]@{
            'DestIP'        = $IpDst
            'DestPort'    = $IpDstPort
            'SrcIP'        = $Matches.IpSrc
            'Session_count' = $Matches.Sessions 
        }
    }
}
{% endhighlight %}  
The resulting output for a single entry would look like this:
![Parsing-Logs1]({{ "/assets/images/Parsing_Logs_01.png" | absolute_url }})
If the first three Regex statements look a little intimidating I would highly recommend checking out [Regex101.com](https://www.regex101.com). It's a great way to write and test regex statements in real time against data you provide.  Each Regex statement above is stored in a corresponding variable with the intent being to match against the first row of one of my intended entries, then the second, and lastly the third. Upon matching the 3rd and final row of an entry it returns a PSCustomObject with the property names of my choosing, and the values I've captured.  Again, this was suggested to me by another user on Stackoverflow but I loved the logic of it, and it absolutely worked for what I was doing.  

Fast forward to the last month and a user on Reddit asked a similar question to what I did when I first encountered this. I happily provided the above answer, but decided to dig a little deeper on my own.  This user had some McAfee ENS firewall logs they wanted to parse. Having dealt with these logs before I know they can get quite big, and even trying to do "Find and Mark" in Notepad++ can sometimes cause the program to stop responding because there's so many rows in the text file.  Come to think of it, I don't know why I never thought to use Powershell to try to deal with these.

The user provided a redacted sampling of what a McAfee ENS log entry looks like:

![Parsing-Logs3]({{ "/assets/images/Parsing_Logs_03.png" | absolute_url }})

Using the same logic before we can see that 7 rows represent a complete log entry. Starting with the "Time:" row and ending with the "Matched Rule:" row.  We just need to build our Regex for each row, with capture groups to pull the data we want, then on the last matched row return a PSCustomObject.  
Regex:

{% highlight Powershell %}
$RETime = '^Time:\s+(?<Time>.*)'
$REEvent = '^Event:\s+(?<Event>.*)'
$REIP = '^IP Address:\s+(?<IP>.*)'
$REDescription = '^Description:\s+(?<Description>.*)'
$REPath = '^Path:\s+(?<Path>.*)'
$REMessage = '^Message:\s+(?<Message>.*)-\s+Source\s+(?<SrcIP>\S+)\s+\:\s+\((?<SrcPort>\d{1,5})\)\s+Destination\s+(?<DstIP>\S+)\s+\:\s+\((?<DstPort>\d{1,5})\)'
$RERule = '^Matched Rule:\s+(?<Rule>.*)'
{% endhighlight %}

Then we could do something similar to before and get PSCustomObjects saved in an array

{% highlight Powershell %}
$Log = Get-Content C:\scripts\Testing\ENSlog.txt

$Results = Foreach ($Line in $Log){
    If ($Line -match $RETime){$Time = $Matches.Time}
    If ($Line -match $REEvent){$Event1 = $Matches.Event}
    If ($Line -match $REIP){$IP = $Matches.IP}
    If ($Line -match $REDescription){$Description = $Matches.Description}
    If ($Line -match $REPath){$Path = $Matches.Path}
    If ($Line -match $REMessage){
        $Message = $Matches.Message
        $SrcIP = $Matches.SrcIP
        $SrcPort = $Matches.SrcPort
        $DstIP = $Matches.DstIP
        $DstPort = $Matches.DstPort
    }
    If ($Line -match $RERule){
        [PSCustomObject]@{
            Time = $Time
            Event = $Event1
            IP = $IP
            Description = $Description
            Path = $Path
            Message = $Message
            SourceIP = $SrcIP
            SourcePort = $SrcPort
            DestinationIP = $DstIP
            DestinationPort = $DstPort
            Rule = $Matches.Rule
        }
    }
}
{% endhighlight %}

Output for the 3 log entries shown above would instead look like this:

```
Time            : 06/29/2021 08:06:56
Event           : Traffic
IP              : Redacted
Description     :
Path            :
Message         : Blocked Incoming UDP
SourceIP        : Redacted
SourcePort      : 54915
DestinationIP   : Redacted
DestinationPort : 54915
Rule            : Block all traffic

Time            : 06/29/2021 08:06:57
Event           : Traffic
IP              : Redacted
Description     :
Path            :
Message         : Blocked Incoming UDP  
SourceIP        : Redacted
SourcePort      : 5353
DestinationIP   : 224.0.0.251
DestinationPort : 5353
Rule            : Block all traffic

Time            : 06/29/2021 08:06:57
Event           : Traffic
IP              : Redacted
Description     :
Path            :
Message         : Blocked Incoming UDP
SourceIP        : Redacted
SourcePort      : 54915
DestinationIP   : Redacted
DestinationPort : 54915
Rule            : Block all traffic
```

I realize this looks pretty similar, but if you pipe the output to Format-Table it changes the way you can look at these logs significantly:

```
Time                 Event    IP       Description Path Message                SourceIP SourcePort DestinationIP DestinationPort
----                 -----    --       ----------- ---- -------                -------- ---------- ------------- ---------------
06/29/2021 08:06:56  Traffic  Redacted                  Blocked Incoming UDP   Redacted 54915      Redacted      54915
06/29/2021 08:06:57  Traffic  Redacted                  Blocked Incoming UDP   Redacted 5353       224.0.0.251   5353
06/29/2021 08:06:57  Traffic  Redacted                  Blocked Incoming UDP   Redacted 54915      Redacted      54915
```

In real-world examples where "Description" and "Path" have values it would likely push the table off the viewable screen, but now that our logs are objects with properties we can take advantage of Where-Object to help us filter:

{% highlight Powershell %}
$Results | Where-Object {$_.DestinationPort -eq "5353"} | Select-Object SourceIP,DestinationIP,DestinationPort

SourceIP DestinationIP DestinationPort
-------- ------------- ---------------
Redacted 224.0.0.251   5353
Redacted 224.0.0.251   5353
Redacted 224.0.0.251   5353
...
{% endhighlight %}

## Let's speed it up

This worked pretty well for the small sampling the user provided, but then I remembered just how painfully big those ENS text files could get and I wondered if there wasn't a faster way than all of those "if" statements.  After a little bit of searching I found someone mention that you could actually specify a text file with the Switch command and use a switch block to process all the regex patterns. 
Same Regex patterns as before, just swapping out how the text is processed:

{% highlight Powershell %}
$Results = switch -regex -file C:\scripts\Testing\ENSlog.txt {
    $RETime {$Time = $Matches.Time}
    $REEvent {$Event1 = $Matches.Event}
    $REIP {$IP = $Matches.IP}
    $REDescription {$Description = $Matches.Description}
    $REPath {$Path = $Matches.Path}
    $REMessage {
        $Message = $Matches.Message
        $SrcIP = $Matches.SrcIP
        $SrcPort = $Matches.SrcPort
        $DstIP = $Matches.DstIP
        $DstPort = $Matches.DstPort
    }
    $RERule {
        [PSCustomObject]@{
            Time = $Time
            Event = $Event1
            IP = $IP
            Description = $Description
            Path = $Path
            Message = $Message
            SourceIP = $SrcIP
            SourcePort = $SrcPort
            DestinationIP = $DstIP
            DestinationPort = $DstPort
            Rule = $Matches.Rule
        }
    }
}
{% endhighlight %}

In some short examples this proved to be a bit faster (in milliseconds) than the previous method.  I artificially bloated up my sample log file by copying and pasting the data over and over again until I had reached 1,000,000+ lines. The Switch block was definitely faster than the If statements but it was the difference between 4 and a half minutes and 4 minutes. 
I found a [blog post](http://www.happysysadm.com/2014/10/reading-large-text-files-with-powershell.html) that tested a bunch of different methods for reading text files and they found a .NET method that proved to work nicely.
All together now:

{% highlight Powershell %}

Start-Job -ScriptBlock {
$RETime = '^Time:\s+(?<Time>.*)'
$REEvent = '^Event:\s+(?<Event>.*)'
$REIP = '^IP Address:\s+(?<IP>.*)'
$REDescription = '^Description:\s+(?<Description>.*)'
$REPath = '^Path:\s+(?<Path>.*)'
$REMessage = 'Message:\s+(?<Message>.*)-\s+Source\s+(?<SrcIP>\S+)\s+\:\s+\((?<SrcPort>\d{1,5})\)\s+Destination\s+(?<DstIP>\S+)\s+\:\s+\((?<DstPort>\d{1,5})\)'
$RERule = '^Matched Rule:\s+(?<Rule>.*)'
$Results = Switch -Regex ([system.io.file]::readlines('C:\scripts\testing\ENSlog.txt')){
    $RETime {$Time = $Matches.Time}
    $REEvent {$Event1 = $Matches.Event}
    $REIP {$IP = $Matches.IP}
    $REDescription {$Description = $Matches.Description}
    $REPath {$Path = $Matches.Path}
    $REMessage {
        $Message = $Matches.Message
        $SrcIP = $Matches.SrcIP
        $SrcPort = $Matches.SrcPort
        $DstIP = $Matches.DstIP
        $DstPort = $Matches.DstPort
    }
    $RERule {
        [PSCustomObject]@{
            Time = $Time
            Event = $Event1
            IP = $IP
            Description = $Description
            Path = $Path
            Message = $Message
            SourceIP = $SrcIP
            SourcePort = $SrcPort
            DestinationIP = $DstIP
            DestinationPort = $DstPort
            Rule = $Matches.Rule
        }
    }
}

$Results

} -Name 'ensjob'
Do {
    $anim=@("|","/","-","\","|")
    $anim | % {
        write-host "`r$_" -NoNewline -ForegroundColor Yellow
        start-sleep -m 75
        }
} While ((Get-Job 'ensjob').state -eq 'Running')
$job = Receive-Job -name 'ensjob'
$job
{% endhighlight %}

You'll notice I wrapped it in a Start-Job because, despite the performance increase of the .NET method over Get-Content, processing a million lines of text still seems to take upwards of a minute depending on the circumstances.  If it's running as a job then I can create a nice little Do/While loop to provide an animation to watch so I at least feel like something is happening.

This is all still based on a Reddit user's provided log example.  I will have to get my hands on some real logs to do some more testing, but if the performance is there I can envision a Powershell module for dealing with ENS logs.  I will have to update this post more in the future as I learn more. For now, just take it as an example that Regex and Powershell together can help with processing log files in a variety of formats.

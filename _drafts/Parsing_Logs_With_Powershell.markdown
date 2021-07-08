---
layout: posts
title: Parsing log files with Powershell
classes: wide
date: 2021-07-07
---

Early on when I first started using Powershell I was dealing with some firewall logs from a perimeter firewall.  They were exported from a SIEM in CSV format, which I appreciated, but the format within was odd and not condusive to what I was trying to do.  I was having a hard time wrapping my mind around how to deal with them in Powershell and some helpful person on Stackoverflow suggested I use regex to match each row, capture a value, and when the last row of a particular entry was matched, spit out a PSCustomObject with all the property/value pairs I wanted.  
I no longer have an example of this data handy but I can replicate something similar:

|        |       |            |          |  
| ------------- | ------------ | ----------------- | ---------------- |  
| 192.168.10.15 |              |                   |                  |  
|               | ip.dstport 443 |                  |                 |  
|               |                | ip.src 10.15.120.2 |      sessions 1492 |  

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

If the first three Regex statements look a little intimidating I would highly recommend checking out [Regex101.com](https://www.regex101.com). It's a great way to write and test regex statements in real time against data you provide.  Each Regex statement above is stored in a corresponding variable with the intent being to match against the first row of one of my intended entries, then the second, and lastly the third. Upon matching the 3rd and final row of an entry it returns a PSCustomObject with the property names of my choosing, and the values I've captured.  Again, this was suggested to me by another user on Stackoverflow but I loved the logic of it, and it absolutely worked for what I was doing.  

Fast forward to the last month and a user on Reddit asked a similar question to what I did when I first encountered this. I happily provided the above answer, but decided to dig a little deeper on my own.  This user had some McAfee ENS firewall logs they wanted to parse. Having dealt with these logs before I know they can get quite big, and even trying to do "Find and Mark" in Notepad++ can sometimes cause the program to stop responding because there's so many rows in the text file.  Come to think of it, I don't know why I never thought to use Powershell to try to deal with these.

The user provided a redacted sampling of what a McAfee ENS log entry looks like:

```
Time:    06/29/2021 08:06:56 
Event:  Traffic 
IP Address:  Redacted
Description:   
Path:   
Message:      Blocked Incoming UDP  -  Source  Redacted :  (54915)   Destination  Redacted :  (54915) 
Matched Rule:  Block all traffic

Time:    06/29/2021 08:06:57 
Event:  Traffic 
IP Address:  Redacted
Description:   
Path:   
Message:      Blocked Incoming UDP  -  Source  Redacted :  (5353)   Destination  224.0.0.251 :  (5353) 
Matched Rule:  Block all traffic

Time:    06/29/2021 08:06:57 
Event:  Traffic 
IP Address:  Redacted
Description:  
Path:   
Message:      Blocked Incoming UDP  -  Source  Redacted :  (54915)   Destination  Redacted :  (54915) 
Matched Rule:  Block all traffic
```

Using the same logic before we can see that 7 rows represent a complete log entry. Starting with the "Time:" row and ending with the "Matched Rule:" row.  We just need to build our Regex for each row, with capture groups to pull the data we want, then on the last matched row return a PSCustomObject.  
Regex:

{% highlight Powershell %}
$RETime = '^Time:\s+(?<Time>.*)'
$REEvent = '^Event:\s+(?<Event>.*)'
$REIP = '^IP Address:\s+(?<IP>.*)'
$REDescription = '^Description:\s+(?<Description>.*)'
$REPath = '^Path:\s+(?<Path>.*)'
$REMessage = 'Message:\s+(?<Message>.*)-\s+Source\s+(?<SrcIP>\S+)\s+\:\s+\((?<SrcPort>\d{1,5})\)\s+Destination\s+(?<DstIP>\S+)\s+\:\s+\((?<DstPort>\d{1,5})\)'
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

This worked pretty well for the small sampling the user provided, but then I remembered just how painfully big those ENS text files could get and I wondered if there wasn't a faster way then all of those "if" statements.  After a little bit of searching I found someone mention that you could actually specifiy a text file with the Switch command and use a switch block to process all the regex patterns. 
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

In some short examples this proved to be a bit faster (in milliseconds) than the previous method




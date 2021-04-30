---
layout: posts
title: Get-Connections; netstat for Powershell
classes: wide
date: 2021-04-29
---

One of the tools I feel like I've been using for years is Netstat. It exists in both Linux and Windows (with some differences) and has similar syntax.  It's often helpful for determining if you've got a service listening on the port you expect, or if you're really making that outbound connection that the GUI says you are. In security it was helpful for these same reasons.  One of the places I would look for Indicators Of Compromise (IOC) was within netstat. 
At one job I was beginning to use Powershell constantly for remote inspection of hosts, or even remediation, and I was curious what the Powershell alternative for netstat was. Something object oriented so I could better control the output.  
Get-NetTCPConnection has entered the chat:  
![Get-NetTCPConnection]({{ "/assets/images/Get_connections_01.png" | absolute_url }})  
![Get-NetTCPConnection2]({{ "/assets/images/Get_connections_02.png" | absolute_url }})  
It has a lot of the same functionality as netstat just a little more long winded on the syntax side:  
{% highlight Powershell %}
Get-NetTCPConnection -State Listen
{% endhighlight %}  
Running this will return all of my listening connections. One of the things I was always interested in when searching for IOCs is which process did that state belong to.  If there's an active listening state running on port 4444 it'd be nice to know if that's a Metasploit process or McAfee.  
Even more long winded, but you can include the owning process ID in the results:  
![Get-NetTCPConnection3]({{ "/assets/images/Get_connections_03.png" | absolute_url }})  
The process ID is great, but I'd like a human readable name:  
![Get-NetTCPConnection4]({{ "/assets/images/Get_connections_04.png" | absolute_url }})  
Wow. That's even worse, and there's no way I'm going to remember all of that.  I'm leveraging a technique that works with Format-Table, Format-List and Select-Ojbect (maybe others?) where you can customize a property you want returned by supplying a hashtable where the Key is the name you would like to give this property and the value is an expression, often more Powershell ran against the current object in the pipeline.  In the above example I do this twice, both times using the OwningProcess ID number from Get-NetTCPConnection and passing it to the Get-Process cmdlet.  The first time I retrieve just the ProcessName property and the second time I execute Get-Process with the "-IncludeUserName" parameter and retrieve that value. The last part only works if you're running your session as administrator, if not it'll just return an empty value for that property. 

To make our request a little bit easier to look at we could use a technique called splatting that I like:
{% highlight Powershell %}
$SelObjArgs = [ordered]@{
    Property = @("Local*",
                "Remote*",
                "State",
                @{Name="Process";Expression={(Get-Process -Id $_.OwningProcess).ProcessName}},
                @{Name="User";Expression={(Get-Process -Id $_.owningprocess -IncludeUserName).Username}}
    )
}
{% endhighlight %}  
I know I want to pass the "-Property" parameter to Select-Object and then supply an array of properties that I wanted returned. So I make an ordered hashtable that really only contains one key/value pair. The key is "Property" (or any parameter you want to use this technique with). The value is then an array of properties I want, and for ease of reading I put a line break after each item.  Then when you want to call this variable instead of prepending a ```$``` you use an ```@```. Then it kind of "splats" the list of parameter/value pairs you've put in your hash table.  
![Get-NetTCPConnection5]({{ "/assets/images/Get_connections_05.png" | absolute_url }})  
To fix the output and make it look like a table again we just need to pipe it to Format-Table:
![Get-NetTCPConnection6]({{ "/assets/images/Get_connections_06.png" | absolute_url }})  
That's pretty much it. We could just wrap that in a Function declaration and we'd be good, but the more I used this new function the more I realized it'd be nice to do some filtering when you first call Get-NetTCPConnection, rather than storing the results in a variable and then filtering them with a Where-Object after. Just like in the examples I've been showing I've been focusing on just local port 3389. I crafted a parameter block for the function and a switchblock to deal with the parameters.  
Param:  
{% highlight Powershell %}
Param(
    [Parameter(Mandatory=$false)]
    [ValidateSet("LocalAddress","LocalPort","RemoteAddress","RemotePort","State","Process","User")]
    $Sort = "State",
    [Parameter(ParameterSetName = "Filter", Mandatory=$false)]
    $LocalAddress = $null,
    [Parameter(ParameterSetName = "Filter", Mandatory=$false)]
    $LocalPort = $null,
    [Parameter(ParameterSetName = "Filter", Mandatory=$false)]
    $RemoteAddress = $null,
    [Parameter(ParameterSetName = "Filter", Mandatory=$false)]
    $RemotePort = $null,
    [Parameter(ParameterSetName = "Filter", Mandatory=$false)]
    [ValidateSet("Listen","Bound","Established","CloseWait","Timewait")]
    $State = $null
)
{% endhighlight %}  
Switchblock:  
{% highlight Powershell %}
$GNTCArgs = [ordered]@{}

If ($PSCmdlet.ParameterSetName -eq "Filter"){
    Switch ($PSCmdlet.MyInvocation.BoundParameters.keys) {
        'LocalAddress' {$GNTCArgs.Add('LocalAddress',$LocalAddress)}
        'LocalPort' {$GNTCArgs.Add('LocalPort',$LocalPort)}
        'RemoteAddress' {$GNTCArgs.Add('RemoteAddress',$RemoteAddress)}
        'RemotePort' {$GNTCArgs.Add('RemotePort',$RemotePort)}
        'State' {$GNTCArgs.Add('State',$State)}
    }
}
{% endhighlight %}
Forgive the abbreviations. Let's start at the top. The first parameter, "Sort", accepts the property names that would commonly be returned by this function.  It's used later with Sort-Object to allow you to control the sort at function execution.  The rest of the parameters as you move down allow you to filter on certain properties based on a value you provide. If *any* of these parameters are specified the "If" statement before the switchblock is triggered and the switchblock will process each parameter by name. Just prior to the switchblock I initiate an empty ordered hashtable.  The switchblock adds the parameter/property name and the user supplied value to the hashtable. This way, just like we splatted a hashtable to the Select-Object cmdlet we can splat our "filter" arguments to the Get-NetTCPConnection cmdlet. 
The function might look something like this:  
{% highlight Powershell %}
Function Get-Connections {
[CmdletBinding()]
Param(
    [Parameter(Mandatory=$false)]
    [ValidateSet("LocalAddress","LocalPort","RemoteAddress","RemotePort","State","Process","User")]
    $Sort = "State",
    [Parameter(ParameterSetName = "Filter", Mandatory=$false)]
    $LocalAddress = $null,
    [Parameter(ParameterSetName = "Filter", Mandatory=$false)]
    $LocalPort = $null,
    [Parameter(ParameterSetName = "Filter", Mandatory=$false)]
    $RemoteAddress = $null,
    [Parameter(ParameterSetName = "Filter", Mandatory=$false)]
    $RemotePort = $null,
    [Parameter(ParameterSetName = "Filter", Mandatory=$false)]
    [ValidateSet("Listen","Bound","Established","CloseWait","Timewait")]
    $State = $null
)

Begin {
    $GNTCArgs = [ordered]@{}
    
    If ($PSCmdlet.ParameterSetName -eq "Filter"){
        Switch ($PSCmdlet.MyInvocation.BoundParameters.keys) {
            'LocalAddress' {$GNTCArgs.Add('LocalAddress',$LocalAddress)}
            'LocalPort' {$GNTCArgs.Add('LocalPort',$LocalPort)}
            'RemoteAddress' {$GNTCArgs.Add('RemoteAddress',$RemoteAddress)}
            'RemotePort' {$GNTCArgs.Add('RemotePort',$RemotePort)}
            'State' {$GNTCArgs.Add('State',$State)}
        }
    }
    
    $SelObjArgs = [ordered]@{
        Property = @("Local*",
                    "Remote*",
                    "State",
                    @{Name="Process";Expression={(Get-Process -Id $_.OwningProcess).ProcessName}},
                    @{Name="User";Expression={(Get-Process -Id $_.owningprocess -IncludeUserName).Username}}
        )
    }
}# end Begin block

Process {
    $Results = Try {
        Get-NetTCPConnection @GNTCArgs -ErrorAction Stop | Select-Object @SelObjArgs
        } Catch [Microsoft.PowerShell.Cmdletization.Cim.CimJobException] {
            Throw "Could not find any states matching that parameter"
        } Catch {
            $Error[0].Exception
        }
    $Results | Sort-Object -Property $Sort | Format-Table -AutoSize -Wrap
}# end Process block

End {
}
} #end of Get-Connections 
{% endhighlight %}  
And finally, some examples:
Revisiting the port 3389 example we've been seeing  
![Get-NetTCPConnection7]({{ "/assets/images/Get_connections_07.png" | absolute_url }})  
Maybe you're only interested if there's a connection to a particular destination IP:  
![Get-NetTCPConnection8]({{ "/assets/images/Get_connections_08.png" | absolute_url }})  

While not revolutionary or groundbreaking Get-Connections is a great example of how you can shape Powershell to better help you in your every day tasks. It's also a good evolution of how I've come to embrace hashtables, splatting, and switchblocks in my Powershell. 

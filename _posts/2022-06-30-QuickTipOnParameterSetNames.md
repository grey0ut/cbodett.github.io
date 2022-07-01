---
layout: posts
title: Quick Tip on ParameterSetNames
classes: wide
date: 2022-6-30
---
  
I was writing a new function today.  Oddly enough I was actually re-writing a function today and hadn't realized it.  Let me explain.  
  
## Story Time
About a half dozen times a month I find myself inspecting a remote computer and invariably the question comes up "how long has this computer been up?"  I always find myself looking up how to get computer uptime in Powershell and I always look for Doctor Scripto's [blog post](https://devblogs.microsoft.com/scripting/powertip-use-powershell-to-find-system-uptime/) where he shows a one-liner that can tell you a computer's uptime:  
  
{% highlight Powershell %} 
PS$> (get-date) – (gcim Win32_OperatingSystem).LastBootUpTime
{% endhighlight %}  
  
The output of the command looks like this:  
![Uptime1]({{ "/assets/images/Get_ComputerUpTime_01.png" | absolute_url }})  
  
Totally sufficient.  I'm usually in a remote Powershell session anyway and just copy/paste the command.  I decided today that I would finally just buckle down and write a function to incorporate in my every day module that would allow me to query my own computer's uptime as well as a remote computer.  Since the info is also in the Win32_OperatingSystem class I thought I would include the date of install as this is often the "imaged" date at work which can be helpful to know as well.  
  
Within about 10 minutes I had a rough draft of the function and it was successful in returning the information I wanted.  
  
![Uptime2]({{ "/assets/images/Get_ComputerUpTime_02.png" | absolute_url }})  
  
I clicked "Save As..." in VS Code and navigated to the folder where the other functions were stored only to see that there was already a function of the same name there dated just two months ago.  I opened it to look at the contents and found that I had indeed written it, with some inspiration borrowed from online (and credited as such) but I didn't actually like the way it was returning info.  It also had a bit of a problem when it came to handling pipeline input.  I relocated the "old" one and saved the new one in its place.  
  
## Tip of the day: ParameterSetName="none"  
The beginning of my new function was looking a lot like most of my new functions; I had declared cmdletbinding and a parameter block.  Originally I had a "ComputerName" parameter to allow for specifying a computer other than the one you're on, and then I thought to add a "Credential" parameter in case I needed to provide different credentials for the remote computer.  

{% highlight Powershell %}  
[Cmdletbinding()]
Param (
    [Parameter(Mandatory=$false,ValueFromPipeline=$true,Position=0)]
    [String]$ComputerName,
    [Parameter(Mandatory=$false,Position=1)]
    [pscredential]$Credential
)
{% endhighlight %}  
  
The rest of the function was going fine but I had the thought that I wanted the function to *require* the "ComputerName" parameter **if** the Credential parameter was provided.  I knew about parametersets and grouping parameters together by a name, but I wasn't sure how this might work.  I read a couple quick blog posts and saw that if I cast the "ComputerName" parameter as Mandatory and put it in the same parameterset as the "Credential" parameter it would work, but then it was ALWAYS asking for a value for the "ComputerName" parameter.  In other functions where I had multiple parametersets this was handled by specfying a "DefaultParameterSetName" in the CmdletBinding definition.  However, this function was only ever going to have one parameterset name.  My first thought was "what if I just set it to something that doesn't exist?" 
   
I quickly changed the first part of my function to look like this, and it worked!  
  
{% highlight Powershell %}  
[Cmdletbinding(DefaultParameterSetName="none")]
Param (
    [Parameter(ParameterSetName="Remote",Mandatory=$false,ValueFromPipeline=$true,Position=0)]
    [String]$ComputerName,
    [Parameter(ParameterSetName="Remote",Mandatory=$false,Position=1)]
    [pscredential]$Credential
)
{% endhighlight %}  
  
Now when calling the function with no arguments it does **not** ask for a "ComputerName" and returns information about the current computer.  If I call the function with the "Credential" parameter but don't supply a "ComputerName" value it will ask for one.  
  
![Uptime3]({{ "/assets/images/Get_ComputerUpTime_03.png" | absolute_url }})  
  
It's a pretty unlikely scenario that I, or someone else, would call the function with JUST the "Credential" parameter and not the computer name, this was more of a proving ground type situation.  
  
If you'd like to see the function in its entirety you can find it on my [Github](https://github.com/grey0ut/Powershell-General/blob/main/Get-ComputerUpTime.ps1)
  
That's all for now. Until the next light bulb moment.

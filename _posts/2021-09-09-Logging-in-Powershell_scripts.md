---
layout: posts
title: Logging in Powershell scripts
classes: wide
date: 2021-09-12
---

Everyone has a different use for Powershell.  Some people use it for daily administrative tasks at work. Some people are hard at work developing Powershell modules.  Personally I find that I use it a lot for administrative work for my own consumption. I may work within an IDE for half the day selectively executing code that I've worked on for a given task.  When I decide to write a function it's typically because I've found a repetitive task that would be made simpler with a function. My [Get-ADPasswordInfo](https://github.com/grey0ut/ActiveDirectoryPS) function is a great example of this. It's probably one of the first functions I ever wrote, and has seen quite a few changes as I've learned more.  It stemmed from wanting to know when an Active Directory user's password was set to expire. AD has this information, but stores it in File Time format, which means nothing to any of us.  I had searched how to convert this on the internet and for a time just saved the one-liner in a notepad and would copy and paste it as I needed.  It didn't take long to realize this should just be a function.  What started as a one-liner is now more than 50 lines, but the result is more or less the same.  

On this particular function I don't really need to know what it's doing line by line as it processes, or be able to refer to a log file after the fact.  Sometimes if I'm troubleshooting why a loop isn't working as expected I will iterate through it line by line, and manually check the contents of variables as I go. Or I might temporarily add some Write-Host statements to make things more visible. However, if I'm writing a script that will be ran unattended, or I'm providing it to someone else for their use, I will include more console output as well as some kind of text log file.  If you search Github you can find a lot of good logging functions that people have written.  I don't claim that mine is any better than any of these, but it may include something you will find useful.  
# Output  
As the simplest example I will often use Write-Host with colors to display information as the script progresses. Consider this simple function:
{% highlight Powershell %}
Function Start-SleepUntil {
    Param(
    [Parameter(Mandatory=$true,Position=0)]
    [DateTime]$Time    
    )
    $CurrentTime = Get-Date
    $Duration = ($Time - $CurrentTime).TotalSeconds
    Start-Sleep -Seconds $Duration
} 
{% endhighlight %}
Instead of providing the Start-Sleep cmdlet with the number of seconds you want to sleep you can provide this function with the desired end time of the sleep and it will do the math for you.  However, when executed It tells you nothing:
![Powershell-Logging1]({{ "/assets/images/Powershell_Logging_02.png" | absolute_url }})  
Maybe it would be nice to have some of that information output on the console.

{% highlight Powershell %}
Function Start-SleepUntil {
    Param(
    [Parameter(Mandatory=$true,Position=0)]
    [Alias('EndTime','Stop')]
    [datetime]$Time  
    )
    Write-Host "Provided end time: $Time" -ForegroundColor Cyan
    $CurrentTime = Get-Date
    Write-Host "Current Time: $CurrentTime" -ForegroundColor Yellow
    $Duration = ($Time - $CurrentTime).TotalSeconds
    Write-Host "Starting sleep for $Duration seconds" -ForegroundColor Red
    Start-Sleep -Seconds $Duration
} 
{% endhighlight %}
This gives you a little bit more information about what's going on behind the scenes:  
![Powershell-Logging1]({{ "/assets/images/Powershell_Logging_03.png" | absolute_url }})  
You could also swap the Write-Host statements for Write-Verbose statements and then people could use the common parameter "-Verbose" to see the message:  
![Powershell-Logging1]({{ "/assets/images/Powershell_Logging_04.png" | absolute_url }})  

# Output to a file  
Sometimes for auditing purposes it can be nice to have common output saved to a file. Let's consider the same silly example from above but in addition to providing console output we're also going to save that information to a file.  
{% highlight Powershell %}
Function Start-SleepUntil {
    Param(
    [Parameter(Mandatory=$true,Position=0)]
    [Alias('EndTime','Stop')]
    [datetime]$Time  
    )
    $LogFile = "C:\Users\Courtney Bodett\Temp\Logfile.txt"
    Write-Host "Provided end time: $Time" -ForegroundColor Cyan
    Add-Content -Path $LogFile -Value "Provided end time: $Time"
    $CurrentTime = Get-Date
    Write-Host "Current Time: $CurrentTime" -ForegroundColor Yellow
    Add-Content -Path $LogFile -Value "Current Time: $CurrentTime"
    $Duration = ($Time - $CurrentTime).TotalSeconds
    Write-Host "Starting sleep for $Duration seconds" -ForegroundColor Red
    Add-Content -Path $LogFile -Value "Starting sleep for $Duration seconds" 
    Start-Sleep -Seconds $Duration
} 
{% endhighlight %}  
![Powershell-Logging1]({{ "/assets/images/Powershell_Logging_05.png" | absolute_url }})  
The output from the function still looks the same, but now there is also a record of it in a text file.  The Add-Content cmdlet will append to the specified file so it can be used repeatedly without overwriting existing information. Unfortunately though we had to add two lines each time we wanted to print some information and it's starting to get tedious.  
# Enter the logging function
As I mentioned before, there are a lot of good examples of logging functions on Github, but I wrote one that was well suited to the environment I work in. At its simplest it just needs to shorten the amount of time it takes to include logging in your script. If you have to provide the path to the logfile every time you want to log something it could get pretty annoying. Since this is going to be a running logfile that input is appended to it would also be good to have timestamps next to everything that's added. It might start something like this:
{% highlight Powershell %}
Function Write-LogMessage {
    [Alias("LogMsg")]
    Param(
        [Parameter(Position = 0, ValueFromPipeline ,Mandatory=$false)]
        $Msg
    )
    $Destination = "C:\Users\Courtney Bodett\Temp\LogFile.txt"
    $Timestamp = (Get-Date).ToString() + " - "
    Add-Content -Path $Destination -Value ($Timestamp + $Msg)
}
{% endhighlight %}  
We start off with an appropriate Powershell verb-noun combo but notice I include an alias statement right after the definition. This will allow me to call the function via the short alias rather than the long name.  If we use it in our previous example it would look like this:
{% highlight Powershell %}
Function Start-SleepUntil {
    Param(
    [Parameter(Mandatory=$true,Position=0)]
    [Alias('EndTime','Stop')]
    [datetime]$Time  
    )
    $LogFile = "C:\Users\Courtney Bodett\Temp\Logfile.txt"
    Write-Host "Provided end time: $Time" -ForegroundColor Cyan
    LogMsg "Provided end time: $Time"
    $CurrentTime = Get-Date
    Write-Host "Current Time: $CurrentTime" -ForegroundColor Yellow
    LogMsg "Current Time: $CurrentTime"
    $Duration = ($Time - $CurrentTime).TotalSeconds
    Write-Host "Starting sleep for $Duration seconds" -ForegroundColor Red
    LogMsg "Starting sleep for $Duration seconds" 
    Start-Sleep -Seconds $Duration
} 
{% endhighlight %} 
With the new logging function in place in addition to the existing Write-Host statements you can see that the output looks the same, but when looking at the log file our latest 3 entries have timestamps in front of them:  
![Powershell-Logging1]({{ "/assets/images/Powershell_Logging_06.png" | absolute_url }})  
After that it can be nice to add the ability to add a "line" to the file as a separator, or maybe a header when you start logging a new invocation of something just to make things easier to read.  For my environment I wanted to be able to use this function in all scripts and dictate per script where the log file would be located as well as specify whether logging to a network location would be included. Then to save time when writing scripts allow the logging function to also output to console if needed. At the beginning of each script specify the following three variables that will then be used by the logging function:
{% highlight Powershell %}
$Logfile = "C:\Users\Courtney Bodett\Temp\LogFile.txt"
$ServerLogfile = "\\NAS\Logging\Logfile.txt"
$LogMsgOutput = $true
{% endhighlight %} 
When looking at the Get-Help info for the Add-Content cmdlet I found that the -Path parameter will actually accept an array of values.  The logging function can then write to either a single local location, or the local location and the network location without having to include extra lines.  We just need to set up our destinations beforehand.  An If statement is then used to control whether or not console output is preferred.  The whole function looks something like this:
{% highlight Powershell %}
Function Write-LogMessage {
    [Alias("LogMsg")]
    Param(
        [cmdletbinding()]
        [Parameter(Position = 0, ValueFromPipeline ,Mandatory=$false)]
        $Msg,
        [Switch]$Line,
        [String]$Head,
        [ValidateSet("Continue","SilentlyContinue")]
        [Alias("EAO")]
        $EAOverride = "SilentlyContinue"
    )

    Begin {
        $Destination = @()
        If (!($LogFile)){
            $LogFile = "C:\Temp\Logfile"
        }
        If ($ServerLogFile){
            $Destination += $ServerLogFile
        }

        $Destination += $LogFile
        $Separator = "-"*70
        $Timestamp = (Get-Date).ToString() + " - "
        $ErrorActionPreference = $EAOverride
    } 

    Process {
        switch ($PSBoundParameters.keys) {
            'Line' {
                Add-Content -Path $Destination -Value $Separator 
                If ($LogMsgOutput){
                    Write-Output $Separator
                }
            }
            'Head' {
                $OutputValue = @($Separator,($Timestamp + $Head.ToUpper()),$Separator)
                Add-Content -Path $Destination -Value $OutputValue
                If ($LogMsgOutput){
                    Write-Output $OutputValue
                }
            }
            'Msg' {
                Add-Content -Path $Destination -Value ($Timestamp + $Msg)
                If ($LogMsgOutput){
                    Write-Output ($Timestamp + $Msg)
                }
            }
        }
    }
}
{% endhighlight %} 
Our simple Start-SleepUntil function can now use just a single instance of LogMsg and output to the console as well as log to one or many destinations.  
{% highlight Powershell %}
Function Start-SleepUntil {
    Param(
    [Parameter(Mandatory=$true,Position=0)]
    [Alias('EndTime','Stop')]
    [datetime]$Time  
    )
    $LogFile = "C:\Users\Courtney Bodett\Temp\Logfile.txt"
    $LogMsgOutput = $true
    LogMsg "Provided end time: $Time" -Head "Start-SleepUntil"
    $CurrentTime = Get-Date
    LogMsg "Current Time: $CurrentTime"
    $Duration = ($Time - $CurrentTime).TotalSeconds
    LogMsg "Starting sleep for $Duration seconds" 
    Start-Sleep -Seconds $Duration
} 
{% endhighlight %} 
![Powershell-Logging1]({{ "/assets/images/Powershell_Logging_07.png" | absolute_url }})  
As you can see from the output the text displayed in the console matches what gets logged.
# Conclusion
I encourage everyone to include some form of console output or logging to a file if you're writing scripts that will run unattended or be consumed by other business areas. It can be immensely helpful when diagnosing errors or trying to understand why the output isn't as desired.  This is just one example of many, but I hope that serves as a good example of the value-add that can come from decent logging.

---
layout: posts
title: Powershell all of the things. And more logging
classes: wide
date: 2022-3-16
---
  
## "If all you have is a hammer, everything looks like a nail" - Abraham Maslow.  
I use a variation of this quote a lot, and I typically use it in jest, but it's also fairly true.  I'm more than willing to admit when there is a better solution than trying to write a Powershell script.  But I do love writing Powershell and often make the argument that since we predominately use Windows, it makes sense to script things via Powershell.  I recently had occassion to script something in Powershell to automate a task.  While the purpose of the script was to simplify a routine operation, I took it as an opportunity to leverage my in development logging module.
  
# The Need  
I recently learned that virtual F5 BIG-IPs should never be snapshot via a hypervisor as it can cause processes to stall out and high available clusters to fail over.  Instead, F5 recommends that you create a configuration backup called a "UCS."  This is typically done with the web GUI and can then be downloaded from there and stored for safe keeping. 
Of course my first thought when learning this was "we can do that with Powershell."  I looked to see if F5's BIG-IP had a REST API, and they did.  Invoke-RestMethod to the rescue!  Unfortunately I would say that F5's documentation about their API leaves a little bit to be desired, especially concerning their UCS backups.  I couldn't find any examples of people using Powershell to automate the creation and download of UCS backups.  
  
that being said, getting connected to the F5 BIG-IP with Invoke-RestMethod wasn't too bad and you can authenticate with the built-in -Credential parameter and a PSCredential object. Like this:  
{% highlight Powershell %}  
$Uri = "https://$IP/mgmt/tm/sys/ucs"
$Headers = @{
    "Content-Type" = "application/json"
}
$UCSResponse = Invoke-RestMethod -Method GET -Headers $Headers -Uri $Uri -Credential $F5Creds
{% endhighlight %}  
The request above will return some objects, stored in $UCSResponse, and among the properties you can get information about the current UCS archives on the appliance.  I parse some of this information like this:  
{% highlight Powershell %}  
$CurrentUCS = Foreach ($UCSFile in $UCSResponse.items.apirawvalues) {
    [PSCustomObject]@{
        Filename = $UCSFile.filename.split('/')[-1]
        Version = $UCSFile.version
        InstallDate = $UCSFile.install_date
        Size = $UCSFile.file_size
        Encrypted = $UCSFile.encrypted
    }
}
{% endhighlight %}    

The operation for creating a new UCS archive looks like this:  
{% highlight Powershell %}  
$Headers = @{
    "Content-Type" = "application/json"
}
$Body = @{
    name = "$BackupName"
    command ="save"
}
$Response = Invoke-RestMethod -Method POST -Headers $Headers -Body ($Body | ConvertTo-Json -compress) -Uri $Uri -Credential $F5Creds -ContentType 'application/json'
{% endhighlight %} 
  
That's really all there is to it. You hit the right URL, pass the API commands and options in the body of the request, and authenticate with a PSCredential object.  However, there is no documentation for how to download the resulting UCS backup.  They cover where it's located on the machine, and there is another API call for downloading files from the BIG-IP however when I worked through that I found out the file download ended up only being 1MB instead of 300+MB.  
  
Digging around some more I found another piece of F5 documentation that stated that their API for file downloads is capped at 1MB.  This feels like an intentional move on F5's part to push customer's towards buying their BIG-IQ backup appliance for managing these things.  Some members on their forum pointed out that F5's own Python SDK can handle downloading a UCS archive, and it's ultimately using the same API, so off I went to Github to read some Python.  Turns out they built a loop in to their file download function that downloads the files in .5MB chunks while streaming it to a file.  I also saw comments on Github that this is reportedly very slow.  
  
# Put the hammer down  
I was about two hours in to writing my own version of this particular Python method in Powershell when I took a break and explained to a friend what I was doing.  They looked at me as if I had told them I thought the CD-ROM tray was a cup holder.  Once I looked up from what I was doing for a moment I realized I shouldn't try to work within F5's constraints and instead just Secure Copy (SCP) the file off the box.  I've used the PoSH-SSH module before for SSH/SCP/SFTP functionality, and while I try to write scripts with little to no dependencies this seemed like a worthwhile inclusion.  
  
I put a "Sanity Checks" section near the beginning of my script and this is where I verify prerequisites.  Checking for PoSH-SSH looks like this:  
{% highlight Powershell %} 
if (-not(Test-path "C:\Program Files\WindowsPowerShell\Modules\posh-ssh")){
    Write-Host "Posh-SSH is needed to connect to SCP. Please install and try again" -ForegroundColor Red
    Exit
}
# import the Posh-SSH module for making an SCP connection
Import-Module -Name 'Posh-SSH' -Scope Local  
{% endhighlight %}  
Since this is a script and not a function I feel comfortable with using "Exit" to terminate script execution.  Another thing I import is my in-development logging module.  I've still been leveraging the module on the daily in conjunction with another module that's really just a collection of daily-use functions.  This script represented an opportunity to take advantage of good logging for auditing and troubleshooting purposes since this script could be ran daily or weekly.  I also wanted the script to log to a local file on my computer as well as a file on a network share, something I had envisioned from the onset of the WriteLog module.  
  
I decided to try importing the module via a literal path:  
{% highlight Powershell %} 
$PathToLogging = "C:\Scripts\Modules\WriteLog"
if (-not(Test-Path $PathToLogging)) {
    Write-Host "Logging Module not found at $PathToLogging" -ForegroundColor Red
    Write-Host "Please modify the PathToLogging variable and try again" -ForegroundColor Yellow
    Exit
} Else {
    Import-Module -Name $PathToLogging
}
{% endhighlight %}  
For brevity I've shown the variable and its use in one snippet.  With the logging available I can start using module functions to record and display output.  
{% highlight Powershell %} 
Set-WriteLogConfig -LogFilePath $LogDir -LogServerPath $NetworkLogPath -WriteHost
Start-Log -ScriptVersion "1.3"
Write-LogEntry "Task running as $env:username"
{% endhighlight %}  
This starts off by establishing a set of variables related to logging activities: The local file path, network file path, whether or not to display the information to the host, log to a global variable, and the logname. Some of this information is determined automatically and as you can see some is provided with parameters.  Then I can start the log entry with "Start-Log" which just puts a header of sorts in the log file and in this case includes the script version.  This way if I'm looking back through the logs and see a different behavior and notice that it was version "1.2" instead maybe that will help me correlate.  
  
For the rest of the script I'll use "Write-LogEntry" and "Write-LogObject" to log information as well as display it to the host.  What the console sees is exactly what gets logged to file.  
  
# Add-Content vs Out-File  
There's some pretty good pages out there that cover the difference between the Add-Content and Out-File cmdlets.  I can't honestly remember my decision process early on.  I was originally using Out-File in all my logging functions until I did a Get-Help on Add-Content and saw that the -Path parameter would accept an array of objects.  I thought this would be really handy for dynamically providing the destination for logging.  It could be a single local file, or as many local files and network files as you wanted to put in the array.  The actual code in Write-LogEntry for writing data to a file would only have to be one line in that case and you could manage the destination as a variable. You can see how this originally tuned out in [my previous post]({% post_url 2021-11-4-Logging-in-Powershell-scripts-cont %}) where I outlined how must of this functioned.  
  
I've been using it with Add-Content for months now without issue, but I've only been logging to a local file.  For this backup script I wanted to log locally and to a network file.  I immediately saw red text upon testing complaining that a "stream was unreadable" or something like that.  File locks appeared to be the issue and all of my Google-fu was telling me that Out-File had better file lock handling.  A quick refactor of my Write-Log functions and my errors were gone. Instead of a one-liner I came up with this instead for Write-LogEntry:  
{% highlight Powershell %} 
Foreach ($FileDestination in $Destination){
    Out-File -FilePath $FileDestination -Encoding 'UTF8' -Append -NoClobber -InputObject ('{0} {1}: {2}' -f $OutputObject.Timestamp,$OutputObject.Severity,$OutputObject.LogObject) -ErrorAction Stop
}
{% endhighlight %} 
The end result is the same on the console and in the text file, and at least so far, it doesn't seem to be much slower.  This was a good usecase for testing my logging module.  There were a couple of other little tweaks too but we don't need to go in to them in this post.  
  
# Execute  
Now that logging had been sorted out, and all the other functional pieces were in place we could execute the script. 
A quick Get-Help shows that there is only one parameter and it's an override to skip removing older backups.  
  
![UCS1]({{ "/assets/images/ucs_01.png" | absolute_url }}) 
  
The first thing that happens after executing the script is a request for credentials. This is for authenticating against the F5 BIG-IP, for both the web API and SCP.  
  
![UCS2]({{ "/assets/images/ucs_02.png" | absolute_url }})  
  
If I wanted to run this script as a scheduled task I'd need to [secure those F5 credentials]({% post_url 2021-04-25-Secure-Creds-In-Powershell %}) and make them available to the account executing the scheduled task.  
The script connects to the first F5 appliance we're backing up and shows a list of the current UCS backup files present on the machine. Then it sends the API call to create a new UCS backup. This can take a moment or so:  
  
![UCS3]({{ "/assets/images/ucs_03.png" | absolute_url }})  
  
Once the backup is created the script moves on to using SCP to copy the file off the F5 appliance and to a network share.  Thanks to the [developers](https://github.com/darkoperator/Posh-SSH) of the PoSH-SSH module there's a nice progress bar while you wait for this to complete. I also called the cmdlet in the script with the -Verbose switch for extra information:  
  
![UCS4]({{ "/assets/images/ucs_04.png" | absolute_url }})  
  
Loops through and does the next appliance:  
  
![UCS5]({{ "/assets/images/ucs_05.png" | absolute_url }})  
![UCS6]({{ "/assets/images/ucs_06.png" | absolute_url }})  
  
The tail end of the script deals with the backup destination directory.  It get's all of the *.ucs files, and references anything that's older than 90 days.  It then shows you these files (thereby logging it as well) and then removes them:  
  
![UCS7]({{ "/assets/images/ucs_07.png" | absolute_url }})  
  
Here's a snippet of one of the log files to show that it looks just like the console output:  
   
![UCS8]({{ "/assets/images/ucs_08.png" | absolute_url }})  

# Conclusion  
The end script is about 200 lines for something that could probably be done in less than 20 (not including the logging module). However, this should be fairly robust, transferable to other team mates, and includes really good logging so that I or others can audit the operation and troubleshoot any problems.  Also, I learned *why* people recommend Out-File over Add-Content so often.  Consequently, Out-File also outputs Powershell objects the way they are seen on the console when writing to a file, whereas the *-Content cmdlets do not.  So actually I've been using Out-File in my Write-LogObject function from the get-go to capture object output the same way it's seen on the console.  Maybe that should have been a clue. 
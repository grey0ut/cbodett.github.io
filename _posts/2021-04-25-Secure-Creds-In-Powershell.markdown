---
layout: posts
title: Secure Credentials in Powershell
classes: wide
date: 2021-04-25
---
A coworker from a neighboring department had an interesting request one day. They wanted a scheduled task to run on a server. Through whatever mechanism the task would look in a series of folders for PDF files. If it found PDFs it would then FTP upload them and when complete move the files to an archive server for safe keeping.  A good job for Powershell with a lot of fun components to it but I want to focus on one aspect: how to save the FTP credentials in the script securely. At some point everyone has probably come across this in a script somewhere:
{% highlight Powershell %}
$Username = "Admin"
$Password = "MySecretPassword"
{% endhighlight %}
This wouldn't fly here.  Typically I would handle this by having the script prompt the user for the credential at the execution of the script using Get-Credential:
![Get-Credential]({{ "/assets/images/PS_credential_01.png" | absolute_url }})  
![SecureString]({{ "/assets/images/PS_credential_02.png" | absolute_url }})  
The value for the provided password is then stored in that object as a SecureString. It can then be safely used throughout the script during execution and is cleared when the script ends.  If you want to save a SecureString to a file for use later you need to convert the object from a SecureString using ConvertFrom-SecureString.  I recommend reading Microsoft's documentation about the ConvertFrom and ConvertTo SecureString cmdlets.  
[ConvertFrom-SecureString](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/convertfrom-securestring?view=powershell-7.1)  
[ConvertTo-SecureString](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/convertto-securestring?view=powershell-7.1)  
By default ConvertFrom-SecureString will use the Windows Data Protection API (DPAPI) to encrypt the standard string contents, which looks like this:  
![DPAPI_String]({{ "/assets/images/PS_credential_03.png" | absolute_url }})  
What's convenient, and secure, about the use of DPAPI with ConvertFrom-SecureString is that the encryption key is derived from the current user on the current machine.  This means that if I saved that string to a file only my user account on that computer would be able to convert it back in to a SecureString object and use it in a script.  If your account got hacked then an attacker could reveal the plain text content of these saved SecureStrings.  
My first thought was that there should be a dedicated service account for use with this Scheduled Task. It would have no logon rights and a very complex secure password. This account, on the intended server/computer, would then be the one to take the FTP credentials, convert them from a SecureString and save them to a file for later use.  This way, any other administrator of that server would just see encrypted junk in the file and no amount of Powershell-fu could reveal it to them.  
The Powershell to collect the credentials and save them to a file is pretty simple:
{% highlight Powershell %}
$FTPCreds = Get-Credential -Message "Provide credentials for FTP connection"
{% endhighlight %}
It will then pop up the standard Get-Credential GUI prompt. Fill out the fields and hit enter.
Now we'll use a cmdlet called "Export-Clixml" to save the entire PScredential object to a file for ease of importation later.
{% highlight Powershell %}
$FTPCreds | Export-Clixml "C:\Users\Courtney\Documents\FTPCreds.xml"
{% endhighlight %}
If you look at this XML file in a text editor you can see that this cmdlet saves what type of Powershell object it is and contains both object properties: UserName and Password. You can see the password is a DPAPI encrypted string.
![Clixml]({{ "/assets/images/PS_credential_04.png" | absolute_url }})  
Using the companion cmdlet Import-Clixml you can easily create a PScredential object with these saved credentials.  
![ClixmlImport]({{ "/assets/images/PS_credential_05.png" | absolute_url }})  
Now we have the mechanics we need to capture credentials, save them to a file and import those credentials to Powershell all securely.  The trick is, we need the service account that will be running this script to store the credentials for import later.  That's tricky because I already said this service account won't have logon rights to the server. We need a script just for saving these credentials it turns out. A server administrator should be able to launch this script and provide the FTP credentials to then be saved to an XML file and that process has to be done as our service account user.  There may be another way to do this but I decided to use the "-Credential" parameter of the "Start-Job" cmdlet to execute a script block as the service account. Observe the behavior here:  
![Start-Job]({{ "/assets/images/PS_credential_06.png" | absolute_url }})  
First I run "whoami" so you can see the session is running under my account then I start a job that executes "whoami" as part of the script block and lastly I provide the "-Credential" parameter and specify a different user account (FTPuser). I receive the job, which would be the output of the executed "whoami" script block and as you can see it returned the username "FTPuser".  
The workflow would then be:  
- Administrator launches script
- Administrator is prompted for service account credentials
- Administrator is prompted for FTP credentials they wish to save
- XML file it outputted with the saved credentials  

Then in the primary script the FTP credentials can just be imported via Import-Clixml from that file.  It's pretty common for service account passwords to be set to never expire meaning that this whole exercise might seem pointless but nevertheless I wanted the end user to have the option to change the saved password if they wanted.  
Here's an example of what this script might look like:  

{% highlight Powershell %}  
# Location where the FTP credential file will be saved
$SavedFolder = "$PSScriptRoot\Credentials\"
$SavedFilename = "ftp_obj.xml"
# If the "Credentials" folder doesn't exist, create it
If (!(Test-Path $SavedFolder)){
    New-Item -Path $PSScriptRoot\Credentials -ItemType Directory | out-null
}

# Get credentials for service account that runs this task
$SvcCreds = Get-Credential -message "Provide credentials for Service Account" -UserName "Contoso\ServiceAcct7508"
$SvcUname = $SvcCreds.Username.Split('\')[-1]
$SvcPasswd = $SvcCreds.GetNetworkCredential().Password
# need to validate the SVC account credentials before starting the job
Add-Type -AssemblyName System.DirectoryServices.AccountManagement
$PC = New-Object System.DirectoryServices.AccountManagement.PrincipalContext([System.DirectoryServices.AccountManagement.ContextType]::Domain, 'Contoso')
# validate the credentials, and retry up to 3 times total before failing out and exiting the script. 
$Count = 1
While ((($Auth = $PC.ValidateCredentials($SvcUname,$SvcPasswd)) -eq $False) -and ($Count -le 2)){
    $Count++
    $SvcCreds = Get-Credential -message "Auth Failure. Try again. $(4 - $count) attempts left"
    $SvcUname = $SvcCreds.Username.split('\')[-1]
    $SvcPasswd = $SvcCreds.GetNetworkCredential().Password
}
If (!($Auth)){
    Write-Host "Failed to validate service account credentials. Please check and try again" -ForegroundColor Yellow
    Write-host "Hit enter to exit this script"
    Read-Host
    Exit
}

# Get credentials for the FTP connection. 
$FTPCreds = Get-Credential -message "Provide credentials for FTP connection" -UserName "FTPuser"

# Save the FTP credentials as a SecureString object that only that service account, on this computer, can decrypt for use later
Write-Host "Saving Credentials..." -ForegroundColor Yellow
$CredJob = Start-Job -ScriptBlock {$Using:FTPCreds | Export-Clixml -path ($Using:SavedFolder + $Using:SavedFilename) -Force} -Credential $SvcCreds
Wait-Job $CredJob | out-null
$t = Receive-Job -Job $CredJob

Write-Host "FTP credentials saved. Please hit Enter to exit this script" -ForegroundColor Yellow
Read-Host 
Exit
{% endhighlight %}  
The last three lines were added to support better execution of this script via a shortcut. The administrator then simply double-clicks a standard Windows shortcut and is greeted with a Powershell window, and then prompted for the credentials. The shortcut might look something like this:  
![Shortcut]({{ "/assets/images/PS_credential_07.png" | absolute_url }})  
The behavior at the end is simply that when the user hits "enter" (or really anything to satisfy the Read-Host cmdlet) the script moves to the next execution statement which is an "Exit" causing the window to close.  
You may have also noticed the lines regarding validating the service account credentials are correct.  This is just another sanity check to deal with the potential error of the administrator providing an incorrect password for the service account.  Had we not validated it there then the Start-Job cmdlet would have likely generated an error when the credentials didn't work.  Since I had written this little validation block before I thought I would recycle it.  

There you have it.  A secure way to leverage service accounts, Windows Task Scheduler, Powershell and saved credentials.  
# Update  
Sometime in the Spring of 2023 Microsoft pushed out a security update and I'm attributing the change in behavior to that.  What happened was the script that was being executed via a shortcut starting erroring out.  
![Error]({{ "/assets/images/PS_credential_08.png" | absolute_url }})  
The key part here is the "Access Denied".  When it first happened I stepped through the script in ISE, line by line, and did not encounter the error.  I chalked it up to "who knows" and moved on. But a few months later it was time to update the password and it happened again.  I quickly realized I had launched ISE as Administrator and that was the difference: double-clicking the shortcut would launch a standard Powershell session.  I ran the script from an Administrative Powershell session and it worked no problem.  
After some testing it appears that now if you want to use Start-Job with the -Credential parameter it needs to be ran with administrative rights.  To update the process on our end I edited the shortcut file. Under "Advanced..." there is a checkbox for "Run as administrator" which will cause the shortcut to always elevate to administrative rights and spawns a UAC prompt.  This caused the script to fail to run and I believe it's because the spawned Powershell session was starting in "C:\Windows\System32" regardless of what you configure in the "Start in" field.  Passing the -File parameter to Powershell.exe wasn't working either.  
To fix this I changed the "Target" statement in the shortcut to look like this:  
{% highlight Powershell %}
powershell.exe -exec bypass -command "& {pushd 'C:\scripts'; .\Save-FTPCredentials.ps1}"  
{% endhighlight %}  
That got us back up and running.  
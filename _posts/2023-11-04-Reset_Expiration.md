---
layout: posts
title: Reset Expiration Clock
classes: wide
date: 2023-11-04
---  
  
With more and more people working remotely there's been a huge uptick in VPN usage.  A lot of organizations have had to completely rethink some of their previous IT policies and procedures.  Some things that used to be simple are now slightly more complicated.  
  
One thing I wasn't aware of, being so far removed from front line customer support at work, was that a lot of our user's passwords were expiring while they were working remote.  With an expired password, they couldn't connect to the VPN, and without connecting to the VPN they couldn't update their password.  Unfortunately self-service password reset is not within our control because that's the obvious answer.  In some cases users were being told to come in to the nearest office so they could sign in to their computer on network, and then update their password.  In other cases the help desk was resetting their password and dictating it to them over the phone. But, more often than not the help desk was asking for the user's current password, and resetting it in AD to that.  Obviously this is all really bad (especially that last one), but there wasn't an available solution to stop this from happening.  I read that there was a way with Powershell to essentially reset the password expiration clock on a user account to push the date out.  If your password expired yesterday, and the domain policy was a 90 day password, then "resetting" it would change your expiration date to 90 days from now.  This would make the user's currently configured password valid again and prevent any form of password sharing.  Then the user could manually initiate a password change once they were up and running again.  
  
# The pwdLastSet Attribute  
  
The pwdLastSet Attribute in Active Directory contains a record of the last time the account's password was set.  Here is the definition from [Microsoft](https://learn.microsoft.com/en-us/windows/win32/adschema/a-pwdlastset):  
>"The date and time that the password for this account was last changed. This value is stored as a large integer that represents the number of 100 nanosecond intervals since January 1, 1601 (UTC). If this value is set to 0 and the User-Account-Control attribute does not contain the UF_DONT_EXPIRE_PASSWD flag, then the user must set the password at the next logon."  
   
There's also the PasswordLastSet attribute which is just the pwdLastSet attribute but converted in to a DateTime object which is a lot more readable.  But, if you want to make a change directly to an account's Password Last Set it's done via the pwdLastSet attribute.  Knowing that it's stored as a large integer number representing "file time" is important when we start making changes to it.  
  
# Updating the Attribute  
  
Making changes to an Active Directory user account is often done with Set-ADUser and this is no different.  If you look at the help info for [Set-ADUser](https://learn.microsoft.com/en-us/powershell/module/activedirectory/set-aduser?view=windowsserver2022-ps) we can see that there are a lot of parameters representing attributes/properties we can change.  The pwdLastSet attribute isn't on the list however. There are plenty of forum hits and examples that reveal that the parameter we need to use is -Replace.  The -Replace parameter accepts a hashtable as value so the syntax is pretty straight forward:  The property name you want to update, and the value you want to replace it with.  
  
Whether a user account's password is expired or not, if you replace the pwdLastSet value with a 0 it effectively expires their password immediately.  We're clearing the slate here.  The next step seems odd but we replace the pwdLastSet value with a -1. Since this is stored as a large integer value we're telling it to set it to the largest number that can be stored in a large integer value.  This would be some insane date out in the future *except* that it uses the domain password policy and caps it out at the default max password age. If that's 90 days for example, then setting it to -1 puts the expiration date as 90 days out in the future from the execution of the command.  The general consensus online is that both of these steps need to be taken: set it to 0, then -1.  I haven't done a deep dive on why, but if anyone has an explanation feel free to hit me up.  
  
# The Script  
  
Seems simple enough then right?  The script just needs to set the pwdLastSet attribute for a given user to 0 and then -1.  One of the things I always ask when I'm writing Powershell for someone else's consumption is "how" they want to be able to use this.  Do they want to manually launch Powershell and execute the script by calling out its path?  Do they want to be able to double-click a shortcut and have the script execute?  Do they just want a function they can run as a CLI tool in Powershell?  
In our case the help desk doesn't spend a lot of time with Powershell and would prefer to just double-click a shortcut.  I on the other hand prefer to run Powershell scripts from an open Powershell session, so I figured I would accommodate both.  
  
At its simplest the script really just needs to do this:  
{% highlight Powershell %}
$User = Read-Host -Prompt "Enter username you wish to reset"  
Set-ADUser -Identity $User -Replace @{pwdLastSet = 0} 
Set-ADUser -Identity $User -Replace @{pwdLastSet = -1} 
{% endhighlight %}  
  
However, I wanted the script to have some sanity checks, provide before and after info regarding the account's password expiration, allow for alternate credential use and to run in a loop in case there were multiple accounts to target.  I also wanted it to support running as the target of a shortcut, as well as an interactive script for users that would prefer to do it that way.  
[Script on Github](https://github.com/grey0ut/ActiveDirectoryPS)

{% highlight Powershell %}
<#
.Synopsis
Script to reset an Active Directory account's password timer to the current date and time.  
.Description
This script will reset the 'pwdLastSet' attribute in Active Directory to the current date and time.  Useful for when an account has an expired password, but the user is remote and has no way to sign in to change their password.
.Parameter Username
The Identity of the user you wish to reset in Active Directory. This should be their SamAccountName. 
If provided with a domain prepended the 'Server' variable will be set to that domain, e.g. 'contoso\jsmith'
.Parameter Credential
A PSCredential object you would like to use instead of the current running user to authenticate the change
.Parameter Server
The domain name you wish to perform the action against if different than the domain you're currently on. 
.Parameter Shortcut
A switch parameter to be used in conjunction with a Windows shortcut. When used it keeps the Powershell window open after script execution until the user hits 'enter'
.EXAMPLE
PS> .\Reset-PasswordClock.ps1
Please provide a username: jsmith
User's current info:

User            : jsmith
DisplayName     : Smith, John
PasswordLastSet : 12/12/2022 7:42:01 AM
ExpiryDate      : 3/12/2023 8:42:01 AM
Lockedout       : False

------------------------------------------------
Would you like to reset the 'PasswordLastSet' to the current time? (y/n):y
Resetting password clock.
User's current info:

User            : jsmith
DisplayName     : Smith, John
PasswordLastSet : 3/15/2023 9:32:01 AM
ExpiryDate      : 6/15/2023 10:32:01 AM
Lockedout       : False

------------------------------------------------
#>
#Requires -Modules 'ActiveDirectory'

Param (
    [CmdletBinding()]
    [Parameter(Position = 0)]
    [String]$Username,
    [Parameter(Position = 2)]
    [PSCredential]$Credential,
    [Parameter(Position = 1)]
    [String]$Server,
    [Switch]$Shortcut
)

If ($Shortcut) {
    $CurrentUser = $ENV:USERNAME
    Write-Host "Script running  as $CurrentUser"
    Do {
        Try {
        [ValidateSet('y','n')]$Answer = Read-Host -Prompt "Would you like to continue? Say 'n' to be prompted for different credentials (y/n)"
        $Continue = $true
        } Catch {
            Write-Host "Please answer with 'y' or 'n'" -ForegroundColor Red
            $Continue = $false
        }
    } Until ($Continue)
    If ($Answer -eq 'n') {
        $Credential = Get-Credential -Message "Provide credentials for executing Reset-PasswordClock"
        Write-Host "Continuing execution as $($Credential.Username)"
    }
}

$Loop = "y"
Do {
    # prompt for a userid to query if not provided at the command line
    If (-not $Username) {
        $Username = Read-Host -Prompt "Please provide a username"
    }

    # get our current domain if not provided by the -Server parameter
    If (-not $Server -and $Username -notmatch '\\') {
        $Server = Get-CimInstance -ClassName win32_computersystem | Select-Object -ExpandProperty Domain
    } ElseIf (-not $Server -and $Username -match '\\') {
        $Server = $Username.Split('\')[0]
    }

    # if username was provided with domain prepended, remove it at this point
    If ($Username -match '\\') {
        $Username = $Username.Split('\')[1]
    }

    # define our 'Select-Object' properties to make the command easier to read down below
    $SelObjArgs = [Ordered]@{
        Property = @(@{Name="User";Expression={$_.SamAccountName}},
                    "DisplayName",
                    "PasswordLastSet",
                    @{Name="ExpiryDate";Expression={[datetime]::fromfiletime($_."msds-userpasswordexpirytimecomputed")}},
                    "Lockedout"
        )
    }

    # our command parameters, defined ahead of time for easier reading down below
    $GetADUserArgs = [Ordered]@{
        Identity = $Username
        Server = $Server
        Properties = @('Displayname','Passwordlastset','msDS-userpasswordexpirytimecomputed','lockedout')
    }
    If ($Credential) {
        $GetADUserArgs.Add('Credential',$Credential)
    }

    # Check AD to see if the supplied username exists and then provide the current state of the account.
    Try {
        $ADInfo = Get-ADUser @GetADUserArgs -ErrorAction Stop | Select-Object  @SelObjArgs
    } Catch [Microsoft.ActiveDirectory.Management.ADIdentityNotFoundException]{
        Write-Warning "$Username not found in domain: $Server"
        Exit
    } Catch{
        $Error[0].Exception
        Exit
    }

    Write-Host "User's current info:"
    $ADInfo
    Write-Host ('-'*48)

    Do {
        Try {
        [ValidateSet('y','n')]$Answer = Read-Host -Prompt "Would you like to reset the 'PasswordLastSet' to the current time? (y/n)"
        $Continue = $true
        } Catch {
            Write-Host "Please answer with 'y' or 'n'" -ForegroundColor Red
            $Continue = $false
        }
    } Until ($Continue)
    Remove-Variable Continue

    If ($Answer -eq "n") {
        Write-Host "Exiting..." -ForegroundColor Yellow
        Exit
    }

    <# Assigning a 0 to the 'pwdLastSet' attribute immediately expires the password, and is a prerequisite to the next step.
    Followed by assigning a -1. Because of the way 64-bit integers are saved, this is the largest possible value that
    can be saved in a LargeInteger attribute. It corresponds to a date far in the future. But the system will assign a 
    value corresponding to the current datetime the next time the user logs on. The password will then expire according 
    to the maximum password age policy that applies to the user.
    #>
    Write-Host "Resetting password clock."
    $SetADUserArgs = [Ordered]@{
        Identity = $Username
        Server = $Server
        ErrorAction = 'Stop'
    }
    If ($Credential) {
        $SetADUserArgs.Add('Credential',$Credential)
    }

    Try {
        Set-ADUser @SetADUserArgs -Replace @{pwdLastSet = 0} 
        Set-ADUser @SetADUserArgs -Replace @{pwdLastSet = -1} 
    } Catch {
        Write-Warning "Encountered an error. Unable to reset password expiration."
        $Error[0].Exception
    }

    # Re-Check AD
    Try {
        $ADInfo = Get-ADUser @GetADUserArgs -ErrorAction Stop | Select-Object  @SelObjArgs
    } Catch [Microsoft.ActiveDirectory.Management.ADIdentityNotFoundException]{
        Write-Warning "$Username not found in domain: $Server"
        Exit
    } Catch{
        $Error[0].Exception
        Exit
    }

    Write-Host "User's current info:"
    $ADInfo
    Write-Host ('-'*48)

    # clear variables in case of loop
    Remove-Variable ADInfo,Username,Server
    # ask if the user would like to run the action again against a different user.
    Do {
        Try {
        [ValidateSet('y','n')]$Loop = Read-Host -Prompt "Would you like to work on another user (y/n)?"
        $Continue = $true
        } Catch {
            Write-Host "Please answer with 'y' or 'n'" -ForegroundColor Red
            $Continue = $false
        }
    } Until ($Continue)

} While ($Loop -eq "y")

If ($Shortcut){
    Write-Host "Press enter to close this window..." -ForegroundColor Yellow
    Read-Host
    Exit
}
{% endhighlight %}
  
That's a lot, I know.  But let's look at what that might look like in action.  You've got a shortcut made for this stored somewhere accessible, and you simply double click it:  
![Shortcut]({{ "/assets/images/Reset_Expiration_01.png" | absolute_url }})  
Then let's say the account you log in to your computer with doesn't have delegated permissions to make these changes so you need to provide credentials. This is collected securely via a Read-Host prompt:  
![Step1]({{ "/assets/images/Reset_Expiration_03.png" | absolute_url }})  
Provide the account name that you want to target and it starts by providing the current state of the account.  Seeing the ExpiryDate property represents an expired password I say "y" to the prompt to reset the clock and it pulls the account from AD again to show that now the PasswordLastSet and ExpiryDate properties have updated.  
![Step2]({{ "/assets/images/Reset_Expiration_04.png" | absolute_url }})  
If there are more accounts to be done you can answer "y" at the end and the process starts over.  If we say "n" then it will prompt that pressing "enter" will close the window, and that's it.  If running the script from the CLI directly it's very similar except that you can provide a PSCredential object as a parameter or simply specify a username and it will securely prompt for a password.  The loop is the same, just without the prompt at the end to "close this window".  

# Conclusion  
  
In the end the script has helped with a task that was previously being performed with poor security practices.  It should be noted that both NIST and ISO have moved away from recommending password expiration as a security control, and as organizations catch up this will likely become less and less of an issue.  Also, Powershell is absolutely **not** the right solution for this problem.  The right solution would be to have a self-service password reset portal available where the users can authenticate with their expired password, and securely update their password and have it update in all required systems.  In absence of that, Powershell turned out to be a pretty good solution.
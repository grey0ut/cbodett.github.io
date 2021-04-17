---
layout: posts
title: Get-ADPasswordInfo
classes: wide
---
When I first started getting in to Powershell I was working in an IT Security position and was sifting through a lot of "noise" in the SIEM alerts.  The main offender was account lockouts.  Typically, if I looked up the user in Active Directory I'd find out that they had recently changed their password, and so it wasn't anomalous behavior for them to have locked their account.  But, getting this information from the AD/UC snapin was very slow, and some of the information was more easily gleaned through Powershell.  One of the Sys Admins had given me a script they wrote that kind of did what I wanted, but I decided to write my own.  

Running the following command got me to a good starting place:
{% highlight Powershell %}
Get-ADuser -Identity JohnSmith -Properties * | Select-Object *
{% endhighlight %}
This gave me all of the possible properties a user object from AD might contain and what their values were. A little bit of searching and I had my list of properties I wanted to query for a specific user.
{% highlight Powershell %}
Get-ADuser -Identity JohnSmith -Properties Displayname,Passwordlastset,Badpasswordtime,msDS-userpasswordexpirytimecomputed,lockedout,accountlockouttime,LastBadPasswordAttempt
{% endhighlight %}
That's kind of long to look at and as it turns out, the value stored within "msDS-userpasswordexpirytimecomputed" isn't very friendly so you need to convert it to something more human readable:
{% highlight Powershell %}
Get-ADuser -Identity JohnSmith -Properties Displayname,Passwordlastset,Badpasswordtime,msDS-userpasswordexpirytimecomputed,lockedout,accountlockouttime,LastBadPasswordAttempt | Select-Object @{Name="ExpiryDate";Expression={[datetime]::fromfiletime($_."msds-userpasswordexpirytimecomputed")}}
{% endhighlight %}
Great, not it's even longer. Obviously there's no way I'm going to remember all of that every time I want to get this information, so this is a perfect opportunity for a Powershell function.  When I have a lot of object properties to deal with I like to set up hashtables so I can splat them at the cmdlet.
{% highlight Powershell %}
$GetADUserArgs = [ordered]@{
    Identity = $User
    Server = $Server
    Properties = @(
        'Displayname',
        'Passwordlastset',
        'Badpasswordtime',
        'msDS-userpasswordexpirytimecomputed',
        'lockedout',
        'accountlockouttime',
        'LastBadPasswordAttempt')
}
{% endhighlight %}
I've got 3 parameters for the Get-ADuser cmdlet I want to provide values for and one of them (Properties) I've got 7 values to provide.  Creating this ahead of time in the above format makes it much easier to read and better for the next person who comes along to edit it.  I also need to convert that one time property to a human readable version in a Select-Object statement so I might as well set up a similar block for that.
{% highlight Powershell %}
$SelObjArgs = [ordered]@{
    Property = @("Displayname",
                "Passwordlastset",
                @{Name="ExpiryDate";Expression={[datetime]::fromfiletime($_."msds-userpasswordexpirytimecomputed")}},
                "Lockedout",
                @{Name="LockoutTime";Expression={ $_.accountlockouttime }},
                @{Name="LastFailedAuth";Expression={ $_.lastbadpasswordattempt}}
    )
}
{% endhighlight %}
The last two are just to rename the properties "AccountLockoutTime" and "LastBadPasswordAttempt" so that my results are a little shorter and easier to look at.  With those two variables defined my cmdlet execution get's to look pretty concise.
{% highlight Powershell %}
Get-ADUser @GetADUserArgs | Select-Object  @SelObjArgs
{% endhighlight %}
Now we just need to build the function around this and we're almost there
{% highlight Powershell %}
Function Get-ADPasswordInfo {
    Param(
    [cmdletbinding()]
    [Parameter(Position = 0, ValueFromPipeline,Mandatory=$true)]
    [Alias('Username')]
    $User,
    [string]$Server = 'contoso.com' #put your AD domain here
)

Begin {
    # stuff to set up before processing the request
    # check to make sure the AD module is loaded
    if (!(Get-module -name ActiveDirectory)){
        Import-Module -Name ActiveDirectory
    }
    # define our 'select-object' properties to make the command easier to read down below
    $SelObjArgs = [ordered]@{
        Property = @("Displayname",
                    "Passwordlastset",
                    @{Name="ExpiryDate";Expression={[datetime]::fromfiletime($_."msds-userpasswordexpirytimecomputed")}},
                    "Lockedout",
                    @{Name="LockoutTime";Expression={ $_.accountlockouttime }},
                    @{Name="LastFailedAuth";Expression={ $_.lastbadpasswordattempt}}
        )
    }
    # an array to store our results in
    $results = @()
} # end Begin block

Process {
    # our command parameters, defined ahead of time for easier reading down below. This needs to be in the 'process' block so that the $user variable can be defined/updated from pipelineinput
    $GetADUserArgs = [ordered]@{
        Identity = $User
        Server = $server
        Properties = @('Displayname','Passwordlastset','Badpasswordtime','msDS-userpasswordexpirytimecomputed','lockedout','accountlockouttime','LastBadPasswordAttempt')
    }
    # do the query and then append the info to the results array
    Try{
        $adinfo = Get-ADUser @GetADUserArgs -ErrorAction SilentlyContinue | Select-Object  @SelObjArgs
    }Catch [Microsoft.ActiveDirectory.Management.ADIdentityNotFoundException]{
        Write-Host "$User not found in domain: $Server" -ForegroundColor Yellow
    }Catch{
        $Error[0].Exception
    }
    $results+=$adinfo
} # end Process block

End {
    $Results | Format-Table
} # end of End block
} # end of function

{% endhighlight %}
I omitted the Get-Help info at the top, but I always recommend writing this so other people can understand how to use your code. Execution of this function might look something like this.
{% highlight Powershell %}
Get-ADPasswordInfo -User cbodett
Displayname            Passwordlastset      ExpiryDate           Lockedout LockoutTime LastFailedAuth
-----------            ---------------      ----------           --------- ----------- --------------
Bodett, Courtney        2/13/2020 3:44:03 PM 4/13/2020 4:44:03 PM     False             3/16/2020 7:16:02 AM
{% endhighlight %}
Since it's an advanced function it supports pipeline input, and the Begin and Process blocks are built around this idea too.  This means you can pipe a bunch of usernames to the function, it will process them all, and then spit out a big table with all of your results.  I would often just do this:
{% highlight Powershell %}
Search-ADaccount -Lockedout | Select-Object -Expand samaccountname | Get-ADPasswordInfo
{% endhighlight %}
This would get me the pertinent password information about every account that was currently locked out.  A quick glance at when their password expired and when they last changed it would usually let me know whether or not this required much further investigation.
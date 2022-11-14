---
layout: posts
title: Get-GeoLocation
classes: wide
date: 2022-11-13
---  
  
## Getting GPS Coordinates From A Windows Machine  
Since 2020 a lot of organizations have ended up with a more distributed workforce than they previously had.  This means a lot of cloud services, VPNs, and company assets out in the wild.  Some tools will let you build a "geo fence" around your infrastructure and block access to resources if the source is from a country other than your approved list.  Let's say you only have employees in the United States, you could specify in your cloud services that if anyone attempts to access your email from a country other than the United States, the authentication attempt would be refused.  
  
This is generally accomplished through IP-based geolocation.  We've all seen TV and movies where they get a person's IP address and then magically pinpoint their location down to a couple of feet.  In reality, that's not true. If you want to see for yourself I used this website quite a bit during testing: [IPlocation.net](https://www.iplocation.net/)  
The site will determine your apparent public IP address (note that a VPN could change this) and then get some publicly available information about the IP from a WHOis lookup.  It will also query several Geo-IP databases and come up with GPS coordinates for your IP.  Using the site above it comes up with a location that's about 36 miles off.  That's certainly good enough to determine what country I am in, and maybe even what state I'm in, depending, but that's about it.  For preventing out-of-country login attempts that's probably fine, but if I fire up NordVPN and specify Germany as my destination, IPlocation.net will now say I'm in Germany.  That's how most cloud destinations will view it as well.  
  
For the sake of argument, let's say you work for an organization that allows remote work within the United States but you want to take a trip out of country and do some remote work.  Maybe a VPN would be enough to convince all of your work resources that you were still in the United States and not throw any alarms.  I wanted to more accurately determine a computer's location and found that there really aren't a lot of options out there, except for the Location Services that exist on most Windows computers. One good way to leverage this service is through Powershell and a .NET namespace.  
  
## GeoCoordinateWatcher  
While searching the internet for how to get GPS coordinates out of a computer I kept running across the same code, or slight variations of it.  
From [StackOverflow](https://stackoverflow.com/questions/46287792/powershell-getting-gps-coordinates-in-windows-10-using-windows-location-api)  
{% highlight Powershell %} 
Add-Type -AssemblyName System.Device #Required to access System.Device.Location namespace
$GeoWatcher = New-Object System.Device.Location.GeoCoordinateWatcher #Create the required object
$GeoWatcher.Start() #Begin resolving current locaton

while (($GeoWatcher.Status -ne 'Ready') -and ($GeoWatcher.Permission -ne 'Denied')) {
    Start-Sleep -Milliseconds 100 #Wait for discovery.
}  

if ($GeoWatcher.Permission -eq 'Denied'){
    Write-Error 'Access Denied for Location Information'
} else {
    $GeoWatcher.Position.Location | Select Latitude,Longitude #Select the relevent results.
}
{% endhighlight %} 
From [Microsoft](https://techcommunity.microsoft.com/t5/windows-powershell/accuracy-issues-when-calling-geocoordinatewatcher/m-p/1519830)  
{% highlight Powershell %} 
# Time to see if we can get some location information
# Required to access System.Device.Location namespace
Add-Type -AssemblyName System.Device

# Create the required object
$GeoWatcher = New-Object System.Device.Location.GeoCoordinateWatcher

# Begin resolving current locaton
$GeoWatcher.Start()

while (($GeoWatcher.Status -ne 'Ready') -and ($GeoWatcher.Permission -ne 'Denied')) 
{
#Wait for discovery
Start-Sleep -Seconds 15
}
#Select the relevent results.
$LocationArray = $GeoWatcher.Position.Location
{% endhighlight %} 

From [Github](https://github.com/I-Am-Jakoby/PowerShell-for-Hackers/blob/main/Functions/Get-GeoLocation.md)  
{% highlight Powershell %} 
function Get-GeoLocation{
	try {
	Add-Type -AssemblyName System.Device #Required to access System.Device.Location namespace
	$GeoWatcher = New-Object System.Device.Location.GeoCoordinateWatcher #Create the required object
	$GeoWatcher.Start() #Begin resolving current locaton

	while (($GeoWatcher.Status -ne 'Ready') -and ($GeoWatcher.Permission -ne 'Denied')) {
		Start-Sleep -Milliseconds 100 #Wait for discovery.
	}  

	if ($GeoWatcher.Permission -eq 'Denied'){
		Write-Error 'Access Denied for Location Information'
	} else {
		$GL = $GeoWatcher.Position.Location | Select Latitude,Longitude #Select the relevent results.
		$GL = $GL -split " "
		$Lat = $GL[0].Substring(11) -replace ".$"
		$Lon = $GL[1].Substring(10) -replace ".$" 
		return $Lat, $Lon


	}
	}
    # Write Error is just for troubleshooting
    catch {Write-Error "No coordinates found" 
    return "No Coordinates found"
    -ErrorAction SilentlyContinue
    } 

}

$Lat, $Lon = Get-GeoLocation
{% endhighlight %} 
  
In my preferred editor (Visual Studio Code) I copied over some of these and started experimenting.  It's clear they're using a .NET namespace "System.Device" in order to create an instance of a "GeoCoordinateWatcher" object.  I looked this object class up so I could read more about it straight from [Microsoft](https://learn.microsoft.com/en-us/dotnet/api/system.device.location.geocoordinatewatcher?view=netframework-4.8).  
  
I always like stepping through code line by line when I'm writing it and exploring what properties and methods the objects I'm dealing with have.  If we execute the first couple lines we've seen in all of these examples we'll have an object we can play with:  
{% highlight Powershell %} 
Add-Type -AssemblyName System.Device 
$GeoWatcher = New-Object System.Device.Location.GeoCoordinateWatcher
{% endhighlight %} 
Then simply call the new object and just see what it says:  
![Geo1]({{ "/assets/images/Get_GeoLocation_01.png" | absolute_url }})  
From this we can see that my "Permission" property is "Granted", "Status" is "NoData" and "Position" looks like it contains some additional objects.  I can infer from the code examples I pointed out that there must be some occasions where "Permission" is actually "Denied" but nobody seems to talk about that so for now I'll just be thankful I'm not in that boat and move on.  What's in the "Position" property?  
![Geo2]({{ "/assets/images/Get_GeoLocation_02.png" | absolute_url }})  
The next step seems like calling the "Start" method on the object.  
![Geo3]({{ "/assets/images/Get_GeoLocation_03.png" | absolute_url }})  
I only waited a second or two before calling the object again and as you can see the "Status" was "Ready" pretty quickly.  If I look at the "Position" property again we see that it actually has value now:  
![Geo4]({{ "/assets/images/Get_GeoLocation_04.png" | absolute_url }})  
Wow, awesome. GPS coordinates and a timestamp.  The GPS coordinates are returned in decimal degrees, which can be copy and pasted right in to Google maps to show you the location.  
  
## Accuracy  
Where is Location Services getting this information from?  Well, it's not 100% clear from just this object class alone. The "System.Device" namespace [page](https://learn.microsoft.com/en-us/dotnet/api/system.device.location?view=netframework-4.8) has this to say about it:  
```  
Location information may come from multiple providers, such as GPS, Wi-Fi triangulation, and cell phone tower triangulation.  
```  
I've read similar remarks on forums regarding this service, but that's about as deep as it goes. Through my own testing it seems that if there are no radios (WiFi, GPS, Cellular) on the computer, it will do some type of geolocation look up based on the apparent public IP address.  However, if even a USB WiFi dongle is plugged in the accuracy of the returned GPS coordinates can get as high as within a few yards.  I haven't gotten to test on a computer with a cellular card in it but I assume it would be similarly accurate.  
  
## String Manipulation: The Results  
One thing I noticed in most of the examples of the code was that people were specifically calling the longitude and latitude of the "Position" property.  Remember above that when calling the "Position" property it returned a location and a timestamp, and the location was shown as decimal degrees.  Call the "Location" property of the "Position" property and you get a bigger picture:  
![Geo5]({{ "/assets/images/Get_GeoLocation_05.png" | absolute_url }})  
Now we can see that when viewing "Position" property there's some object formatting taking place to show us the latitude and longitude as comma separated numbers.  In actuality the "Location" property itself has 8 properties and we're really only interested in the "Latitude" and "Longitude" ones.  There's examples out there of different ways to manipulate these to get what you want out of them, but I'm always a big fan of piping an object to Get-Member to see what's available:  
![Geo6]({{ "/assets/images/Get_GeoLocation_06.png" | absolute_url }}) 
I see a "ToString" method, I wonder what that looks like:  
![Geo7]({{ "/assets/images/Get_GeoLocation_07.png" | absolute_url }})   
Great, I'm done.  They did all the work for me and all those other examples out there of using Select-Object, or splitting, or whatever can be ignored.  
  
## Permissions  
One thing you see in every example is a reference to the "Permission" property possibly being listed as "Denied".  I was able to find a couple of computers where this was the case and wanted to understand what was controlling that and how I could possibly overcome it since no one seemed to talk about it in the posts surrounding the above code examples.  
The short story is that it depends on whether or not Location Services is being allowed, at both the computer configuration and user configuration level.  I make that distinction because there's a registry key in both the LocalMachine and CurrentUser hive that applies to this. The path is here:  
```  
Software\Microsoft\Windows\CurrentVersion\CapabilityAccessManager\ConsentStore\location\Value  
```  
![Geo8]({{ "/assets/images/Get_GeoLocation_08.png" | absolute_url }})  
At the computer configuration level if this is set to "Deny" then Location Services won't work and you'll need admin rights to change that registry key.  If the computer configuration is set to "Allow" but the current user configuration is set to "Deny" then the registry key can be changed without administrative privilege.  
To show you what I mean, I've set my current user location registry key to "Deny" and recreated my GeoCoordinateWatcher object:  
![Geo9]({{ "/assets/images/Get_GeoLocation_09.png" | absolute_url }}) 
As you can see the "Permission" property shows "Denied". Calling the "Start" method on the object does not throw an error, and the "Status" property never changes from "NoData".  
![Geo10]({{ "/assets/images/Get_GeoLocation_10.png" | absolute_url }})  
The while loop in all of the code examples is reliant on "Permission" being equal to something other than "Denied", so in my current state the script would flow right through the while loop and move on, possibly writing some kind of error to host.  
  
What if instead we checked to see what the registry key's value was before trying to start the process?  Then if we have the appropriate permissions, change the value, do the work, and change it back when we're done.  
Let's look at what I would do instead and then talk about it.  
  
## Code  
{% highlight Powershell %} 
$IsAdmin = [Security.Principal.WindowsIdentity]::GetCurrent().Groups -contains 'S-1-5-32-544'
$RegPath = "Software\Microsoft\Windows\CurrentVersion\CapabilityAccessManager\ConsentStore\location\"
$CompValue = Get-ItemProperty ("HKLM:\" + $RegPath) -Name "Value" | Select-Object -ExpandProperty Value

if ($CompValue -ne "Allow" -and $IsAdmin) {
    Write-Verbose "Admin rights present. Editing registry to allow location services"
    Set-ItemProperty ("HKLM:\" + $RegPath) -Name "Value" -Value "Allow"
    $ChangedHKLM = $true
    $Continue = $true
} elseif ($CompValue -eq "Allow") {
    Write-Verbose "Local machine allows for location services"
    $Continue = $true
} else {
    Write-Verbose "No admin rights and location services denied at machine level"
    $Location = "Permission"
    $Continue = $false
}

if ($Continue) {
    $UserValue = Get-ItemProperty ("HKCU:\" + $RegPath) -Name "Value" -ErrorAction SilentlyContinue | Select-Object -ExpandProperty Value
    if ($UserValue -eq "Deny") {
        Write-Verbose "User config: location services denied"
        Write-Verbose "Updating registry for current user to allow location services"
        Set-ItemProperty ("HKCU:\" + $RegPath) -Name "Value" -Value "Allow"
    }
    Add-Type -AssemblyName System.Device
    $GeoWatcher = New-Object System.Device.Location.GeoCoordinateWatcher(1)
    Write-Verbose "Starting GeoCoordinateWatcher"
    $GeoWatcher.Start()
    $C = 0
    while (($GeoWatcher.Status -ne 'Ready') -and ($Geowatcher.Permission -ne 'Denied') -and ($C -le 15)) {
        Write-Verbose "[$C] Waiting 2 seconds for results"
        Start-Sleep -Seconds 2
        $C++
    }
    # need to wait a little longer to allow for more accurate data. 
    Start-Sleep -Seconds 2
    $Location = ($GeoWatcher.Position.Location).ToString()
    $GeoWatcher.Dispose()
    if ($UserValue -eq "Deny") {
        Write-Verbose "Updating registry for current user to revert changes"
        Set-ItemProperty ("HKCU:\" + $RegPath) -Name "Value" -Value "Deny"
    }
}
if ($ChangedHKLM) {
    Write-Verbose "Updating registry for local machine to revert changes"
    Set-ItemProperty ("HKLM:\" + $RegPath) -Name "Value" -Value "Deny"
}
[PSCustomObject]@{
    Computer = $Env:COMPUTERNAME
    Location = $Location
    NetAdapter = (Get-NetAdapter -Physical | Where-Object {$_.Status -eq "Up"} | Select-Object -ExpandProperty Name) -Join ','
}
{% endhighlight %}  
  
Let's talk through this.  
![GeoCode1]({{ "/assets/images/Get_GeoLocation_Code_01.png" | absolute_url }})  
The first line is a method for determining if the current running user is in the local Administrators group.  Then we save the bulk of a registry path for use later, and then check the HKLM value in the registry to see if Location Services are allowed.  
When I tested this on several Domain joined computers it worked great (and was new to me) but as I write this on my personal computer I see some interesting behavior.  My local account (a Microsoft account) **is** in the local Administrators group, but that first line returns false.  Checking manually in the GUI it shows that I **am** in fact in that group, but the "Groups" property from that code doesn't show that I'm in that group.  So I flipped the logic around and instead got the members of that group to see if it contains the SID of the current user, and that returns true.  
![GeoCode2]({{ "/assets/images/Get_GeoLocation_Code_02.png" | absolute_url }})  
Ok, next section.  
![GeoCode3]({{ "/assets/images/Get_GeoLocation_Code_03.png" | absolute_url }})  
A little if/elseif/else action here.  If Location Services is currently not being allowed at the computer level, *and* we're running as admin, then change the registry and define a couple of variables. Else, if the value is already  set to "Allow" then just define a variable.  Else, finally, if we're not running as admin then define that same variable as "False".  If that were the case, then this next section that checks that variable first, would be skipped.  
![GeoCode4]({{ "/assets/images/Get_GeoLocation_Code_04.png" | absolute_url }})  
This is where we do the actual work.  We've checked if Location Services is allowed at the computer level, and if that worked then "Continue" will be true. We start off by essentially doing the same registry check but for the Current User hive.  If Location Services isn't being allowed then we'll change it to allow.  Then we add our .NET namespace and create out GeoCoordinateWatcher object. Note the "(1)" at the end of that. This denotes that it should be created with "High Accuracy" mode.  Since we're only going to be leveraging it for a few seconds, I see no downside to this.  
  
Then we start the process and I also start a counter by defining "$C" as zero.  No one else had this in their code, but I wasn't sure what the maximum potential amount of time this process might take was, and I didn't want to accidentally create an infinite loop so my while loop has 3 conditions.  The final condition, the counter, must be less than or equal to 15.  With the Start-Sleep statement within the loop set to 2 seconds this means the maximum amount of time this loop could go on for is approximately 30 seconds.   
  
I then found through some testing that if you just immediately go from "Ready" to checking for the location that it may not actually be ready.  Unclear what's happening in the background, but if you just wait 2 more seconds it seems to allow for enough time.  Then I take the positional data and use its own "ToString" method to save the GPS coordinates to a variable.  Dispose of the GeoWatcher object and if the user's registry value was changed by this script, change it back.  The next section does the same thing for the computer registry hive.  
![GeoCode5]({{ "/assets/images/Get_GeoLocation_Code_05.png" | absolute_url }})  
Then finally the last section.  
![GeoCode6]({{ "/assets/images/Get_GeoLocation_Code_06.png" | absolute_url }})  
This just creates, and returns, a PSCustomObject with three properties: The current computer name, the GPS coordinates, and the name of the connected network adapters.  Let's execute all of this and see what that looks like.  
![GeoCode7]({{ "/assets/images/Get_GeoLocation_Code_07.png" | absolute_url }})  
Great! Now we have the name of the computer, some pretty accurate looking GPS coordinates, and the network adapter(s) that were present at the time.  I included this because the presence of WiFi seems to be a pretty big factor in how accurate the GPS coordinates are and I thought it might be nice for reference.  
  
The reason it was written the way it was is because I figured more often than not I'm going to be running this against a remote computer and might want to pass this code as a script block.  With that in mind, this is all collected together as a function called Get-GeoLocation, which has a parameter called "ComputerName" for specifying a remote computer you wish to run it against.  This has only been tested in one Active Directory environment so far, but the code is available on Github in case you want to play around with it on your own.  
[GitHub Get-GeoLocation](https://github.com/grey0ut/Powershell-General/blob/main/Get-GeoLocation.ps1)  
  
## Closing Thoughts  
When I started down this rabbit hole of trying to reliably determine a computer's location I really did *not* want to use Powershell to do it.  I know I have a tendency to use Powershell for everything and I wanted to use tools that we already had available to us.  Unfortunately everything seems to rely on Geo-IP databases which return fairly inaccurate results.  This was also a fun exercise in taking "found in the wild" code a step further and hydrating it with some more error handling, and features.  
Remember to check out Microsoft's documentation when you can, pipe to Get-Member, and just explore in general.  It's interesting what you'll find.  
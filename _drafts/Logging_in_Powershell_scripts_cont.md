---
layout: posts
title: Logging in Powershell scripts; Continued
classes: wide
date: 2021-11-4
---
In [my previous post]({% post_url 2021-09-09-Logging-in-Powershell_scripts %}) I explained a bit about some of my justifications for logging in Powershell.  My interest in logging has continued since then and I spent some time exploring Github reading other people's functions and modules. I saw some really neat features amongst all of the code out there and began to think about how I might use some of them in my daily work life.  I have a small module I built and maintain at work, internally, that's just a collection of some tools (like Get-ADPasswordInfo) to help streamline some tasks.  I don't particularly have a need for logging in my module, but there are other departments adjacent to mine that run a lot of Powershell scripts within the organization and they definitely log throughout their scripts.  I decided that I wanted to try purpose-building a module from the ground up for logging.  The idea would be to develop it, integrate it with my daily use module for testing, and ultimately publish it to the Powershell Gallery for other people to use if they like.
# Getting Started  
The first step was to write down all of the things I would want the module to do as each one of those would represent a function.  I also needed to think about how it might do these things.  With some inspiration from Github I decided to approach it like this; from the perspective of a script that's going to be logging, what needs to happen?  
  
Similar to the logging function in my previous post each script would need to know some settings about logging before it could continue.  Where are we logging to? Are we displaying the log info to console? Should we also keep track of current session logs?  
  
I wanted a function that would handle creating a script scope variable that would contain the logging settings.  These settings could be defined via the same function when executed, using parameters, or if executed with no parameters it would look for global saved settings.  Global settings themselves would need two functions: one to save them to an environmental variable, and one to retrieve them.  
A function to save logging preferences globally, a function to retrieve those preferences, and a function to set those preferences as script scope variables.  For sure we'll need a function to actually write a log entry and based on one example I saw in Github I want a pair of functions for starting a log and stopping a log.  
  
Now I had an idea of some functions with some possible names that just needed to be paired up with the appropriate verbs.
- Save-WriteLogConfig  
- Get-WriteLogConfig  
- Set-WriteLogConfig  
- Start-Log  
- Stop-Log  
- Write-LogEntry  

This would be enough to get me started.  
# Laying The Ground Work  
Not as sexy, but just as important, is to layout the structure for our module.  There are plenty of good blog posts, including [this one](http://ramblingcookiemonster.github.io/Building-A-PowerShell-Module/) from Warren F , that dive in to creating modules so I won't spend too much time on this.  
  
The way I like to write and maintain functions for a module is in individual .ps1 files.  There's also a chance that there will be functions that a user of the module should be aware of, and use, and then there will be functions that are internal to the function of the module itself that a user does not need to interact with.  I like the terms "Public" and "Private" for separating these function.  
One of the things I knew I wanted to play with in this module was a custom class for creating a "log object" as well as a custom format file for controlling the appears of these objects.  In addition to the standard module manifest file and .psm1 I'll create the following folders:  
- Classes  
- Formats  
- Private  
- Public  

My .psm1 file contents would then look like this:  
{% highlight Powershell %}
$Public  = Get-ChildItem -Path $PSScriptRoot\Public\*.ps1 -ErrorAction SilentlyContinue
$Private = Get-ChildItem -Path $PSScriptRoot\Private\*.ps1 -ErrorAction SilentlyContinue
$Classes = Get-ChildItem -Path $PSScriptRoot\Classes\*.ps1 -ErrorAction SilentlyContinue

$AllManifestItems = $Public + $Private + $Classes

#Dot source the files
Foreach($Item in $AllManifestItems){
    Try{
        . $Item.fullname
    }
    Catch{
        Write-Error -Message "Failed to import Item $($Item.fullname): $_"
    }
} 
{% endhighlight %}  
Simple enough right?  It just gets all of the .ps1 files from the folders that contain them, and then loops through and dot sources them.  There are other ways to do this, perhaps better, but this is how I've been doing it thus far.  
  
Working down my list of functions I needed to start with I created the files and began writing.  "Save-WriteLogconfig" was probably the simplest as it just needed to save information in an environment variable.  This can be accomplished pretty succinctly with a hashtable:  
{% highlight Powershell %}
$Config = @{
    LogFilePath         = $LogFilePath
    LogServerPath       = $LogServerPath
    CurrentSessionLogs  = $CurrentSessionLogs
    WriteHost           = $WriteHost
}
[Environment]::SetEnvironmentVariable("PSLogging", ($Config | ConvertTo-Json -Compress), "User")
{% endhighlight %}
Then it just needs a good parameter block:  
{% highlight Powershell %}
Param(
    [validatescript({
        if( -not ($_ | test-path) ){
            throw "Folder does not exist"
            }
        if(-not ( $_ | test-path -pathtype Container) ){
            throw "The -path argument must be a folder"
            }
            return $true
    })]
    [Parameter(Mandatory = $true)]
    $LogFilePath,
    [validatescript({
        if( -not ($_ | test-path) ){
            throw "Folder does not exist"
            }
        if(-not ( $_ | test-path -pathtype Container) ){
            throw "The -path argument must be a folder"
            }
            return $true
    })]
    $LogServerPath,
    [Parameter(ParameterSetName = "Switch")]
    [Alias("CSL")]
    [Switch]$CurrentSessionLogs,
    [Parameter(ParameterSetName = "Switch")]
    [Alias("WH")]
    [Switch]$WriteHost
)
{% endhighlight %}  
Then the companion function "Get-WriteLogConfig" to retrieve these settings:  
{% highlight Powershell %}
Function Get-WriteLogConfig {
    [cmdletbinding()]
    Param(
        [Parameter(Mandatory = $false, ParameterSetName = "Env")]
        [String]$PSLogConfig = ([environment]::GetEnvironmentVariable("PSLogging", "User"))
    )

    try {
        $Settings = $PSLogconfig | ConvertFrom-Json
    } catch {
        Throw "Failed to import existing Json: $($_.Exception)"
    }

    if ($PSLogConfig){
        $PSLogPowershellVariables = @{
            LogFilePath         = $Settings.LogFilePath
            LogServerPath       = $Settings.LogServerPath
            LogName             = $Settings.LogName
            CurrentSessionLogs  = $Settings.CurrentSessionLogs.IsPresent
            WriteHost           = $Settings.WriteHost.IsPresent
        }
        $PSLogPowershellVariables
    } else {
        Write-Host "Could not find any settings. Save settings with Save-WriteLogConfig" -ForegroundColor Yellow
    }
}
{% endhighlight %}
Testing these was simple enough as I just needed to be able to provide settings and verify that I could recall them in the current session, or a new session.  Next up, I want to be able to retrieve these settings within a script, or provide the settings. 
# Set-WriteLogConfig  
"Set-WriteLogConfig" accomplishes this:  
{% highlight Powershell %}
Function Set-WriteLogConfig {
    [cmdletbinding(DefaultParameterSetName = 'Env')]
    Param(
        [Parameter(Mandatory = $true, ParameterSetName = "Manual")]
        [ValidateScript({
            if( -not ($_ | test-path) ){
                throw "Folder does not exist"
                }
            if(-not ( $_ | test-path -pathtype Container) ){
                throw "The -path argument must be a folder"
                }
                return $true
        })]
        $LogFilePath,
        [Parameter(Mandatory = $false, ParameterSetName = "Manual")]
        [ValidateScript({
            if( -not ($_ | test-path) ){
                throw "Folder does not exist"
                }
            if(-not ( $_ | test-path -pathtype Container) ){
                throw "The -path argument must be a folder"
                }
                return $true
        })]
        $LogServerPath,
        [Parameter(Mandatory = $false, ParameterSetName = "Manual")]
        [String]$LogName,
        [Parameter(Mandatory = $false, ParameterSetName = "Manual")]
        [Parameter(ParameterSetName = "Switch")]
        [Alias("CSL")]
        [Switch]$CurrentSessionLogs,
        [Parameter(Mandatory = $false, ParameterSetName = "Manual")]
        [Parameter(ParameterSetName = "Switch")]
        [Alias("WH")]
        [Switch]$WriteHost,
        [Parameter(Mandatory = $false, ParameterSetName = "Env")]
        [String]$PSLogConfig = ([environment]::GetEnvironmentVariable("PSLogging", "User"))
    )

    $ScopeLevel = Get-LogScopeLevel
    If ($null -eq $ScopeLevel) {
        $ScopeLevel = 1
    }

    If (-not ($LogName)) {
        $Callstack = Get-PSCallStack 
        $Source = $Callstack[$ScopeLevel].Command
    } Else {
        $Source = $Logname
    }

    switch ($PSCmdLet.ParameterSetName) {
        'Env' {
            try {
                $Settings = $PSLogconfig | ConvertFrom-Json
            } catch {
                Throw "Failed to import existing Json: $($_.Exception)"
            }

            if ($null -eq $Settings -or [boolean]($null -eq $Settings.LogFilePath)) {
                Write-Verbose "No loggings settings provided. Using defaults"
                $Settings = @{
                    LogFilePath         = ${env:temp}
                    LogServerPath       = $null
                    LogName             = [System.Io.Path]::GetFileNameWithoutExtension($Source)
                    CurrentSessionLogs  = $false
                    WriteHost           = $false
                    Scope               = $ScopeLevel
                }
            }

            $PSLogPowershellVariables = @{
                LogFilePath         = $Settings.LogFilePath
                LogServerPath       = $Settings.LogServerPath
                LogName             = [System.Io.Path]::GetFileNameWithoutExtension($Source)
                CurrentSessionLogs  = $Settings.CurrentSessionLogs.IsPresent
                WriteHost           = $Settings.WriteHost.IsPresent
                Scope               = $ScopeLevel
            }
        }
        'Manual' {
            $PSLogPowershellVariables = @{
                LogFilePath         = $LogFilePath
                LogServerPath       = $LogServerPath
                LogName             = [System.Io.Path]::GetFileNameWithoutExtension($Source)
                CurrentSessionLogs  = $CurrentSessionLogs
                WriteHost           = $WriteHost
                Scope               = $ScopeLevel
            }
        }
    }
    Initialize-WriteLogConfig @PSLogPowershellVariables
}
{% endhighlight %}
There's kind of a lot happening here, and if you read through that you may have noticed a couple of new functions.  As I was writing this module, I realized that there was a need for more functions than I originally planned.  I also continued to look through Github for inspiration for how others had handled similar setups.  
  
**Param**  
  
The first thing of note is actually the parameter block, and more specifically that I've arranged the parameters in to two sets by name.  The "Env" parameter set and the "Manual" parameter set.  The former implying that the logging settings will be retrieved from the environment variable created by "Save-WriteLogConfig". The latter more plainly stating that these settings will be provided manually via the parameters of this function.  More on this in a bit.  
  
**Logging Scope**  
  
The next thing of note is "Get-LogScopeLevel" right at the beginning of the script.  I was inspired a lot by EsOsO's "Logging" module on [Github](https://github.com/EsOsO/Logging) when I first started my research.  They actually a few functions built around this idea of "scope" but I wasn't sure I understood it at first glance.  As I started testing my module in use with functions  I noticed some behavior that made my realize why this was necessary.  At first I was getting the calling script name through other means to use as the name of the log file.  I.e. if the script was called "Get-AllUsers" and this logging module was used inside, it would automatically create a logfile named "Get-AllUsers.txt" without any input saying so.  Where this got messed up was when I called a function within a function and both of them were leveraging the logging module.  The logs were start off being written to a file for function A, and then after function B was executed the remaining logs would all be written to it.  This is because the "Set-WriteLogConfig" function is called at the beginning of any participating script and would overwrite the script scope variable with those new settings.  
  
I needed a function to get the current scope level as well as one to set the scope level.  The idea being that if I knew I was about to call a function within a function that's already logging I could manually set the scope level with "Set-LogScopeLevel" to direct the logs to all continue within the scope of the parent script/function.  Just another script scope variable to add to the list:   
{% highlight Powershell %}
Function Set-LogScopeLevel {
    [CmdletBinding()]
    Param(
        [Parameter(Mandatory = $false, Position = 0)]
        [Int]$ScopeLevel = 1
    )

    $Script:PSLogging.Scope = $ScopeLevel
}
{% endhighlight %}  
The companion "Get-LogScopeLevel" basically just retrieving the numeric value stored in that variable.  
Moving down the "Set-WriteLogConfig" function a little further you can see where this comes in to play:  
{% highlight Powershell %}
    $ScopeLevel = Get-LogScopeLevel
    If ($null -eq $ScopeLevel) {
        $ScopeLevel = 1
    }

    If (-not ($LogName)) {
        $Callstack = Get-PSCallStack 
        $Source = $Callstack[$ScopeLevel].Command
    } Else {
        $Source = $Logname
    }
{% endhighlight %}
The method I settled on for getting the script name is a cmdlet I hadn't seen before but stumbled across on one of my searches. If you were to open Powershell and just type "Get-PSCallStack" it would output this:  
{% highlight Powershell %}

Command       Arguments Location 
-------       --------- -------- 
<ScriptBlock> {}        <No file>

{% endhighlight %}
Now, write a function called "Test" that just contains "Get-PSCallStack" and execute "test." Your output will look like this:  
{% highlight Powershell %}

Command       Arguments Location
-------       --------- --------
test          {}        <No file>
<ScriptBlock> {}        <No file>

{% endhighlight %}
By capturing the output of "Get-PSCallStack" in to a variable I essentially create an array.  Since arrays are 0-indexed in Powershell that means if my "LogScopeLevel" is 1, it would be the second thing in this array which would always be the script/function that the logging functions were called within.  If the script is called "Get-AllUsers" and "Set-WriteLogConfig" is called within that, it will pull "Get-AllUsers" as the name of the second object returned from "Get-PSCallStack".  The "Logname" can also be provided manually but it is part of the parameter set "Manual" which means all of the other settings would also be required.  
  
**Switch block**  
  
Moving along in the body of the function I use a switch block off of '$PSCmdLet.ParameterSetName' variable to load up a hashtable named "$PSLogPowershellVariables."  Whichever one the switch block uses the end result is the same; the "Initialize-WriteLogConfig" function takes that hastable as splatted parameters.  
  
**Initialize that config**  
  
"Initialize-WriteLogConfig" is the other new function I decided I needed **and** there's no need for it to be publicly accessible so it gets to be our first "private" function.  Its job is simple.  It takes each one of the logging settings it's fed via the named parameters, and creates a new variable in the script scope containing them.  That way any other logging functions that need to leverage those settings can retrieve them from the script scope variable.  
{% highlight Powershell %}
Function Initialize-WriteLogConfig {
    [cmdletbinding()]
    Param(
        [Parameter(Mandatory = $true)]
        [String]$LogFilePath,
        [Parameter(Mandatory = $false)]
        [String]$LogServerPath,
        [Parameter(Mandatory = $true)]
        [String]$LogName,
        [Switch]$CurrentSessionLogs,
        [Switch]$WriteHost,
        [Int]$Scope
    )

    $Script:PSLogging = @{
        LogFilePath         = $LogFilePath.TrimEnd('\')
        LogServerPath       = $LogServerPath.TrimEnd('\')
        LogName             = $LogName
        CurrentSessionLogs  = $CurrentSessionLogs
        WriteHost           = $WriteHost
        Scope               = $Scope
    }

}
{% endhighlight %}  
Seems like a lot so far and we haven't even gotten to anything resembling logging.  
# Custom Class  
I mentioned in the beginning that I wanted to create a custom class for a log object in this module and also a formats file.  Let's look at these before we get in to "Write-LogEntry."  
  
I use "PSCustomObject" in my scripts a lot as a way to control the output from loops, or to store info in arrays for easier formatting as tables, or output to CSV files.  A powershell class is basically just an object definition.  There is **way** more depth to these then I got in to with mine.  I just needed to define an object so I could write a formats file for it.  The class looks like this:  
{% highlight Powershell %}
Class LogEntryObject {
    [Int32]   $Index
    [String]  $Timestamp
    [String]  $Source
    [String]  $Severity
    $LogObject
    LogEntryObject ([DateTime]$Timestamp, [String]$Source, [String]$Severity, $LogObject) {
        $this.Timestamp = Get-Date $Timestamp -Format 'MM/dd/yy HH:mm:ss'
        $this.Source = $Source
        $this.Severity = '[{0}]' -f $Severity.ToUpper()
        $this.LogObject = $LogObject
    }
}
{% endhighlight %}
This has changed a bit since the initial iteration, and may change still.  But the core of a log entry for me was going to be a timestamp, the source name for where this log is from, the severity level and then the actual log message itself ("$LogObject").  Tyler Muir's post on [AdamTheAutomator.com](https://adamtheautomator.com/powershell-classes/) is where I got a lot of my info for this.  
  
I'm using a class constructor to control the formatting of the timestamp and severity properties.  Then to further control the creation of a LogEntryObject I created another private function called "New-LogEntryObject."  
{% highlight Powershell %}
Function New-LogEntryObject {
    [cmdletbinding()]
    Param (
        [Parameter(Mandatory = $false, Position = 2)]
        [DateTime]$Time = (Get-Date),
        [Parameter(Mandatory = $true, Position = 0)]
        [String]$Source,
        [Parameter(Mandatory = $true, Position = 1)]
        [String]$Severity,
        [Parameter(Mandatory = $false, Position = 3)]
        $LogObject
    )

    [LogEntryObject]::New($Time, $Source, $Severity, $LogObject)
}
{% endhighlight %}
Then the .ps1xml format file that accompanies this custom class is how I control the color output of the "Severity" property.  I saw an example on Reddit, and borrowed most of the methodology from [this post](https://www.reddit.com/r/PowerShell/comments/o7vkr0/change_output_colors_on_pscustomobject_based_on/). The only part of this code with any much significance is the section regarding the "Severity" property:  
{% highlight Powershell %}
<?xml version="1.0" encoding="utf-8"?>
<Configuration>
  <ViewDefinitions>
    <View>
      <Name>LogEntryObject</Name>
      <ViewSelectedBy>
        <TypeName>LogEntryObject</TypeName>
      </ViewSelectedBy>
      <TableControl>
        <TableHeaders>
            <TableColumnHeader>
                <Label>Index</Label>
            </TableColumnHeader>
            <TableColumnHeader>
                <Label>Timestamp</Label>
            </TableColumnHeader>
            <TableColumnHeader>
                <Label>Source</Label>
            </TableColumnHeader>
            <TableColumnHeader>
                <Label>Severity</Label>
            </TableColumnHeader>
            <TableColumnHeader>
                <Label>LogObject</Label>
            </TableColumnHeader>
        </TableHeaders>
        <TableRowEntries>
          <TableRowEntry>
            <TableColumnItems>
            <TableColumnItem>
                <PropertyName>Index</PropertyName>
            </TableColumnItem>
            <TableColumnItem>
                <PropertyName>Timestamp</PropertyName>
            </TableColumnItem>
            <TableColumnItem>
                <PropertyName>Source</PropertyName>
            </TableColumnItem>
            <TableColumnItem>
                <ScriptBlock>
                  $Esc = [char]27
                  $Reset = [char]27 + "[0m"
                  $Color = switch ($_.Severity){
                    {$_ -eq '[INFO]'} {"96"}
                    {$_ -eq '[ERROR]'} {"91"}
                    {$_ -eq '[WARNING]'} {"93"}
                    }
                  "$($Esc)[$($color)m$($_.Severity)$($Esc)[39m"    
                </ScriptBlock>
            </TableColumnItem>
              <TableColumnItem>
                <PropertyName>LogObject</PropertyName>
              </TableColumnItem>
            </TableColumnItems>
          </TableRowEntry>
        </TableRowEntries>
      </TableControl>
    </View>
  </ViewDefinitions>
</Configuration>
{% endhighlight %}
A fairly simple to understand switch block.  Info severity's are blue, Error's are red, and Warning's are yellow.  With those pieces in place I could move on to writing log entries.  
# Are we logging yet?  
"Write-LogEntry" could now be written more effectively since these other building blocks were in place:  
{% highlight Powershell %}
Function Write-LogEntry {
    [cmdletbinding()]
    [Alias("Write-Log")]
    Param(
        [Parameter(Mandatory = $false, Position = 0)]
        [String]$Message,
        [Parameter(Mandatory = $false, Position = 1)]
        [ValidateSet("Info","Warning","Error")]
        [String]$Severity = "Info",
        [Parameter(Mandatory = $false, Position = 2)]
        [Switch]$SuppressOutput
    )

    Begin {
        if (-not ($Script:PSLogging)) {
            try {
                Set-WriteLogConfig
            } catch {
                Throw [System.Management.Automation.ParameterBindingException] 'PSLogging is not configured yet. Run Set-WriteLogConfig first'
            }
        }

        If (-not ($Global:CurrentSessionLogs)) {
            $Global:CurrentSessionLogs  = [System.Collections.ArrayList]@()
        }

        $Destination = @()

        If ($PSLogging.LogFilePath -and $PSLogging.LogName) {
            $LogFile = '{0}\{1}.txt' -f $PSLogging.Logfilepath,$PSLogging.Logname
            $Destination += $Logfile
        }

        If ($PSLogging.LogServerPath -and $PSLogging.LogName) {
            $ServerLogFile = '{0}\{1}.txt' -f $PSLogging.LogServerPath,$PSLogging.Logname
            $Destination += $ServerLogFile
        }
        $OutputObject = New-LogEntryObject -Source $PSLogging.LogName -Severity $Severity -LogObject $Message
    }

    Process {
        Add-Content -Path $Destination -Value ('{0} {1}: {2}' -f $OutputObject.Timestamp,$OutputObject.Severity,$OutputObject.LogObject)
        If ($PSLogging.WriteHost -and (-not $SuppressOutput)){
            Switch ($OutputObject.Severity){
                '[INFO]' {Write-Host ('{0} {1}: {2}' -f $OutputObject.Timestamp,$OutputObject.Severity,$OutputObject.LogObject) -ForegroundColor Cyan}
                '[WARNING]' {Write-Host ('{0} {1}: {2}' -f $OutputObject.Timestamp,$OutputObject.Severity,$OutputObject.LogObject) -ForegroundColor Yellow}
                '[ERROR]' {Write-Host ('{0} {1}: {2}' -f $OutputObject.Timestamp,$OutputObject.Severity,$OutputObject.LogObject) -ForegroundColor Red}
            }
        }
        If ($PSLogging.CurrentSessionLogs) {
            [void]$Global:CurrentSessionLogs.Add($OutputObject)
        }
    }
}
{% endhighlight %}  
Let's step through this a bit with the previous functions in mind.  Starting right off with the "Severity" parameter you can see that I've created a set of valid values to ultimately control what gets sent to "New-LogEntryObject."  This is also where I default to "Info" severity so that "Write-LogEntry" can be called without specifying this parameter.  
  
First check in the beginning of the script is to see if "$PSLogging" exists in the script scope. If it doesn't exist then someone hasn't been following directions and didn't run "Set-WriteLogConfig". We'll attempt to run it ourselves and hope for saved settings via "Save-WriteLogConfig."  
  
The next step is to check and see if there's a global scope variable called "CurrentSessionLogs" and if not, create an array list of that name.  An array list offers an important distinction compared to regular arrays: it is **not** fixed size so you can add objects to it individually without having to tear it down and build it again using something like "+=".  In addition to logging to a file, or files, I wanted to log to a global variable so that, within a given session, you could retrieve logs from scripts you've executed.  
  
Then we set up our destinations.  This could be a single local file and/or a file located on a network share.  The log name will be taken from the script scope "$PSLogging" variable.  
  
Last bit of set up is creating our "LogEntryObject" using the "New-LogEntryObject" private function.  It takes whatever value was provided to the "$Message" parameter of this function and uses it to satisfy the "LogObject" parameter of "New-LogEntryObject."  
  
On to processing.  One line with "Add-Content" handles the actual writing to a file(s) since the "Path" parameter will accept an array of values.  I may need to change this later if I decide I want to incorporate Mutex in to my logging module.  
  
A switchblock handles the console output, if the logging settings deem to do so.  I use "Write-Host" so I can colorize the output to match the colors I used in the format file .ps1xml.  
  
The last piece is adding the same object to the global "CurrentSessionLogs" variable.  
  
# What about logging objects?  
  
I knew I wanted to log pretty much anything coming out of my scripts, but I hadn't thought far enough ahead to realize that if I wanted to log, to a file, the output from scripts I wouldn't be able to use "Add-Content" and maintain the way output looks.  To preserve, for instance, the way an array of PSCustomObjects looks in the console when written to a file I would need to use "Out-File" instead.  Since this is a different task needed when logging I decided there should be a "Write-LogObject" function as well:  
{% highlight Powershell %}
Function Write-LogObject {
    [cmdletbinding()]
    [Alias("Write-LogObj")]
    Param(
        [Parameter(Mandatory = $false, Position = 0)]
        $InputObject,
        [Parameter(Mandatory = $false, Position = 1)]
        [ValidateSet("Info","Warning","Error")]
        [String]$Severity = "Info",
        [Parameter(Mandatory = $false)]
        [Switch]$Passthru,
        [Parameter(Mandatory = $false)]
        [Switch]$SuppressWriteHost
    )

    Begin {
        if (-not ($Script:PSLogging)) {
            try {
                Set-WriteLogConfig
            } catch {
                Throw [System.Management.Automation.ParameterBindingException] 'PSLogging is not configured yet. Run Set-WriteLogConfig first'
            }
        }

        If (-not ($Global:CurrentSessionLogs)) {
            New-Variable -Name "CurrentSessionLogs" -Scope Global -Value ([System.Collections.ArrayList]@())
        }

        $Destination = @()

        If ($PSLogging.LogFilePath -and $PSLogging.LogName) {
            $LogFile = '{0}\{1}.txt' -f $PSLogging.Logfilepath,$PSLogging.Logname
            $Destination += $Logfile
        }

        If ($PSLogging.LogServerPath -and $PSLogging.LogName) {
            $ServerLogFile = '{0}\{1}.txt' -f $PSLogging.LogServerPath,$PSLogging.Logname
            $Destination += $ServerLogFile
        }
        $OutputObject = New-LogEntryObject -Source $PSLogging.LogName -Severity $Severity -LogObject $InputObject
    }

    Process {
        Add-Content -Path $Destination -Value ('{0} {1}: {2}' -f $OutputObject.Timestamp,$OutputObject.Severity,"Object Output")
        Foreach ($FileDestination in $Destination){
            Out-File -FilePath $FileDestination -InputObject $OutputObject.Logobject -Encoding ASCII -Append
        }
        
        If (-not $SuppressWriteHost -and -not $Passthru) {
            $InputObject | Out-Default
        } Else {
            $InputObject
        }
        
        If ($PSLogging.CurrentSessionLogs) {
            [void]$Global:CurrentSessionLogs.Add($OutputObject)
        }
    }
}
{% endhighlight %}
You can see the similarities.  The big differences are in the "Process" block where it handles the first two tasks differently.  First it adds a line to log files that says "Object Output" to signify that the next lines contain that.  Then it loops through the destinations and uses "Out-File" to write the info in ASCII.  
  
For outputting to the console I actually needed an "If" statement depending on circumstances.  For instance, if I wanted to output the results of a script to the console using "Write-LogObject" but I also wanted to pipe them to "Format-Table" I needed to pipe the log object to "Out-Default."  This was necessary to get things to output to the console in the order expected.  Without this I was having script results output on the screen in an unexpected order relative to other operations.  [This blog post](https://get-powershellblog.blogspot.com/2017/04/out-default-secrets-revealed.html) goes in to some really good detail about that.  
  
Lastly the same log object is added to the "$CurrentSessionLogs" variable globally for retrieval later.  
  
# Get the current session logs  
The global variable full of current session logs was honestly the part I wanted to use the most, while I pictured other people might have more use for the actual logging to a file aspect.  I was comfortable with just calling the variable "$CurrentSessionLogs" and then piping to "Where-Object" to get just the things I wanted, but I decided recently that there should be one more public function.  
  
"Get-CurrentSessionLogs" or "GCSL" for short will retrieve the logs from the global variable, and also provides filtering options for retrieving specifics entries.  Let's take a look:  
{% highlight Powershell %}
Function Get-CurrentSessionLogs {
    [cmdletbinding()]
    [Alias("GCSL")]
    Param(
        [DateTime]$After,
        [DateTime]$Before,
        [DateTime]$Timestamp,
        [String]$Source,
        [ValidateSet("Info","Warning","Error")]
        [String]$Severity,
        [String]$Contains,
        [ValidateRange(1,999999)]
        [Int32]$Index,
        [Switch]$LogOnly
    )

    Begin {
        $WhereArray = [System.Collections.ArrayList]@()

        If ($After) {
            [void]$WhereArray.Add('(Get-Date $_.Timestamp) -gt $After')
        }
        If ($Before) {
            [void]$WhereArray.Add('(Get-Date $_.Timestamp) -lt $Before')
        }
        If ($Timestamp) {
            [void]$WhereArray.Add('(Get-Date $_.Timestamp) -eq $Timestamp')
        }
        If ($Source) {
            [void]$WhereArray.Add('$_.Source -Match $Source')
        }
        If ($Severity) {
            [void]$WhereArray.Add('$_.Severity -Match $Severity')
        }
        If ($Contains) {
            [void]$WhereArray.Add('$_.LogObject -Match $Contains')
        }

        $WhereString = $WhereArray -join " -and "
        $WhereBlock = [ScriptBlock]::Create($WhereString)

        $StartIndex = 1
        $CurrentSessionLogs | Foreach-Object {$_.index = $StartIndex; $StartIndex++}

        If ($Index) {
            $IndexNum = $Index - 1
        }
    }

    Process {
        If ($WhereString) {
            $Results = $CurrentSessionLogs | Where-Object $WhereBlock
        } Else {
            $Results = $CurrentSessionLogs
        }
        If ($PSBoundParameters.keys -contains 'Index') {
            $Results = $Results[$IndexNum]
        }   
        If ($LogOnly) {
            $Results = $Results.LogObject
        }
    }

    End {
        $Results
    }
}
{% endhighlight %}
This was pretty fun to work on.  I wrote down a list of all the ways you might want to filter the logs entries by: time, source, severity, keyword. I also wanted to be able to look on the screen, see a specific log, and be able to call it by its index number position in the array.  With 20+ objects in the array this was a little hard when manually indexing in to the array with "$CurrentSessionLogs[14]" as an example.  This was actually when I went back and edited the "LogEntryObject.ps1" class file to add the "Index" property.  

**Filtering**  
  
For "time" I actually decided that using Powershell's "Get-Date" cmdlet I wanted to be able to filter on entries "Before" and "After" a given time, as well as providing a specific timestamp. "Source" and "Severity" are pretty straight forward as is "Contains" for keyword searching.  
The interesting task was figuring out how to dynamically create a "Where-Object" statement.  I wanted to be able to provide no parameters, or combinations of parameters, and still have it function.  Writing each "Where-Object" statement is simple enough and I knew that I could chain them together with "-and" but it took some looking around to figure out the next part.  If you do "Get-Help" on "Where-Object" there's actually a lot in there, and admittedly I hadn't really looked at it before.  I always use "Where-Object" similar to this:  
{% highlight Powershell %}
Get-Process | Where-Object {$_.Name -match "Firefox"}
{% endhighlight %}  
Or I'll use the alias for "Where-Object", "?" for brevity.  However, upon reading the help info I saw that the parameter that occupies position 0 is a Filter Script:  
{% highlight Powershell %}
    -FilterScript <System.Management.Automation.ScriptBlock>
        Specifies the script block that is used to filter the objects. Enclose the script block in braces (`{}`).
        

        Required?                    true
        Position?                    0
        Default value                None
        Accept pipeline input?       False
        Accept wildcard characters?  false
{% endhighlight %}  
This means I could technically pass it a variable as long as that variable is of object type "ScriptBlock."  This  makes the operation pretty straightforward then and could be done with "If" statements or a "Switchblock."  
- Make an array to store our filter script statements  
- Make a string out of all of the objects in that array and join them together with "-and"  
- Create a script block object using that string  

And we're done.  Now in the process block we show filtered results, unfiltered results, a specific entry by index number, or just the actual logged info.  If "GCSL" is executed once it will show all of the logs on the screen like so:  
![GCSL1]({{ "/assets/images/GCSL_02.jpg" | absolute_url }})  
Then if you provide a specific index number and re-run "GCSL" it will return only that entry:  
![GCSL2]({{ "/assets/images/GCSL_03.jpg" | absolute_url }})  
Then if you just want the original output, or "LogObject" from a specific entry you can add that parameter:  
![GCSL3]({{ "/assets/images/GCSL_04.jpg" | absolute_url }})  
  
# The logs  
Now you've seen the current session logs aspect, which is admittedly my favorite part.  But this is about logging, and it wouldn't be logging without something being written to disk.  To incorporate WriteLog in to my existing module's functions I went through and replaced every instance of "Write-Host" with "Write-LogEntry".  Anywhere where a variable's output is being returned directly to the console I replaced that with "Write-LogObject".  In some cases I added some extra logging and used the "SuppressOutput" flag to specific that this only be written to the log file.  With my preferred settings saved using "Save-WriteLogConfig" I could just call "Set-WriteLogconfig" at the beginning of each script file.  Settings:  
![Config]({{ "/assets/images/config.png" | absolute_url }})  
Each script file really just needs to contain 3 lines like this:  
{% highlight Powershell %}
Set-WriteLogConfig
Start-Log -ScriptVersion "1.1"
...
Stop-Log
{% endhighlight %}  
Establish the settings, start the log file (providing a script version is optional), and ultimately stop the log.  How many times you use "Write-LogEntry" or "Write-LogObject" within is up to you.  Here's an example of the target folder's log files:  
![LogFiles]({{ "/assets/images/LogFiles.png" | absolute_url }})  
And the contents of the "Test-Password" log file:  
![Test-Password]({{ "/assets/images/Test-Password Log.png" | absolute_url }})  

# Wrapping up  
This is still very much in development, but I have been using it for the last month or so to debug it.  There are a lot of great turn-key logging modules already on Github and some of them may work better for you.  My intent in writing this module wasn't to make the most widely consumable logging module for Powershell.  I set out to write my first purpose built module, rather than just a collection of things thrown together.  I planned to use it for my own purposes but hoped that maybe it would find use elsewhere in my organization.  If nothing else it was a good thought exercise in how to approach writing a module and I had a lot of fun so far. 
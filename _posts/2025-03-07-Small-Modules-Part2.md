---
layout: posts
title: Small Modules Part 2
classes: wide
date: 2025-12-08
---  

## Natural Language Passphrases  
Previously I talked about making small modules as an excuse to practice making tools and focus on making something that works well.  I ended by saying I might try turning my ['New-NaturalLanguagePassword Script']('https://www.powershellgallery.com/packages/New-NaturalLanguagePassword') in to a module.  
I followed through on this not long after making that post with the first version being published in March and the most recent version being from May.  If you want to skip right to checking it out I invite you to visit the Github page for it: ['PSPhrase']('https://github.com/grey0ut/PSPhrase')  
I wanted it to work on PowerShell v5.1 or newer, as well as Windows, Linux or Mac.  I also wanted to leverage the ['Sampler Module']('https://github.com/gaelcolas/Sampler') for the development.   
## The Checklist  
The first thing I like to do before I even open an IDE is to write down the goals for what this thing will do.  
```
- Retain all existing configuration options from the script  
- Store word lists outside of the ps1 files  
- Allow for saving parameters/values as 'configuration'
```  
It would be easy enough to retain all of the existing parameters from the script as I already wrote them.  In the script the adjective and noun wordlists are stored directly in the script as a hashtable with numbers as keys corresponding to dice rolls.  This is an artifact of originally being based on diceware passwords.  Since I'm doing my own thing now I decided that words can just be stored in simple text files with one word per line.  This simplifies maintaining the wordlists and doesn't constrain us to number ranges that align with 6 sided dice rolls.  
Lastly, I wanted the end user to be able to save parameters and values as their default preference and have it persist between sessions.  I know that ['Paramter Default Values']('https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_parameters_default_values?view=powershell-7.5') exist but I wanted to tailor this specifically for use with the module.  
## What's In a Name?  
![MyNameIs]({{"/assets/images/Small_Modules_02.png" | absolute_url }})  
After listening to [James Brundage]('https://startautomating.com/') talk about how important a good name/branding is for a module at the '24' PowerShell Summit I really like to start with that.  The script name is very long and quite literal.  This module would be generating passwords.  Not just passwords, but passphrases specifically.  I like the idea of trying to incorporate 'PS' in to the name of the module if it works and it didn't take very long to think of 'PSPhrase'.  This would mean I could name the primary function `Get-PSPhrase`.  That's a lot easier to type than `New-NaturalLanguagePassword`.  

## Starting with Sampler Module  
The first thing was to start a new project with SamplerModule.  I'm not going to go in to great detail here, but the resulting structure will look something like this:  
```
.
├── Assets
│   └── PSPhrase.svg
├── build.ps1
├── build.yaml
├── CHANGELOG.md
├── LICENSE
├── output
│   ├── CHANGELOG.md
│   ├── module
│   ├── ReleaseNotes.md
│   ├── RequiredModules
│   └── testResults
├── README.md
├── RequiredModules.psd1
├── Resolve-Dependency.ps1
├── Resolve-Dependency.psd1
├── source
│   ├── Data
│   ├── Private
│   ├── PSPhrase.psd1
│   ├── PSPhrase.psm1
│   └── Public
└── tests
    ├── QA
    └── Unit
```  
Most of my work will be happening in the `source` directory.  
```
.
├── Data
│   ├── Adjectives.txt
│   └── Nouns.txt
├── Private
│   ├── Get-RandomInt.ps1
│   └── Initialize-Dictionary.ps1
├── PSPhrase.psd1
├── PSPhrase.psm1
└── Public
    ├── Get-PSPhrase.ps1
    ├── Get-PSPhraseSetting.ps1
    └── Set-PSPhraseSetting.ps1
```  
My `Data` directory will house the txt files for the dictionary words and will get shipped with the module in the same way.  The `Private` and `Public` directories containing functions will all get built in to a single monolithic .psm1 file with only the Public functions getting exported.  
## Existing Configuration Options  
The parameters present in the `New-NaturalLanaugePassword` script were pretty much taken word for word for the new `Get-PSPhrase` function.  The `Shortcut` parameter was dropped since that was a script specific thing, and the `Pairs` parameter now has a ValidateRange set for between 1 and 100.  
Pretty simple.  
{% highlight Powershell %}
param (
    [ValidateRange(1,100)]
    [Int32]$Pairs = 2,
    [Switch]$TitleCase,
    [Switch]$Substitution,
    [String]$Append,
    [String]$Prepend,
    [String]$Delimiter = ' ',
    [ValidateRange(1,500)]
    [Int32]$Count = 1,
    [Switch]$IncludeNumber,
    [Switch]$IncludeSymbol
)
{% endhighlight %}  
##  Store Wordlists Outside of ps1 Files  
Making txt files containing the words was easy enough, and it allowed me to search for some more words to include so that technically this is more word options than are available in the script.  I really liked the use of hashtables in the original script because it made retrieving words super fast.  Since the wordlists would be stored outside of the actual module (.ps1/.psm1) I'm now introducing a file read operation plus hashtable creation.  I experimented with this a bit using Get-Content and after some testing I found that using the .NET `System.IO.File` class was much faster at reading in the file.  A quick loop to assign a number to the word starting at 1 and going as high as there are words and we've got the first private function.  
{% highlight Powershell %}
function Initialize-Dictionary {
    <#
    .SYNOPSIS
    Creates ordered hashtables of wordlists for faster retrieval with random number generator
    .DESCRIPTION
    Reads in a text file containing one word per line and creates an ordered dictionary starting the keys with '1' and incrementing up from there
    for each word added.  Then using a RNG a corresponding key can be called and the associated value (word) can be retrieved very quickly
    .PARAMETER Type
    Whether to load the Nouns or Adjectives list
    .EXAMPLE
    $Nouns = Initialize-Dictionary -Type Nouns

    will turn $Nouns in to a hashtable containing all the words from the Nouns.txt file
    #>
    [Cmdletbinding()]
    [OutputType([System.Collections.Specialized.OrderedDictionary])]
    param (
        [ValidateSet("Nouns","Adjectives")]
        [String]$Type
    )

    $File = switch ($Type) {
        "Nouns" {"Nouns.txt"}
        "Adjectives" {"Adjectives.txt"}
    }

    $WordListPath = Join-Path -Path ($PSScriptRoot) -ChildPath "Data/$File"

    $Words = [System.IO.File]::ReadAllLines($WordListPath)
    $Dictionary = [Ordered]@{}
    $Number = 1
    foreach ($Word in $Words) {
        $Dictionary.Add($Number, $Word)
        $Number++
    }
    $Dictionary
}
{% endhighlight %}  
The `Type` parameter just helps dictate which txt file to read in. This way the same function works for both the list of adjectives and nouns.  Skipping ahead a bit, but for reference it takes just a handful of milliseconds for Get-PSPhrase to generate a single passphrase which involves reading in both of these txt files, creating hashtables, generating random numbers, and combining words. I think that's plenty fast enough.  
  
Now that we have  way to create our hashtables we also need a way to randomly select a word from the hashtables.  Since I'm not strictly sticking to the idea of dice rolls anymore I can focus on just generating a random number bewteen 1 and however many words there are. Rather than using the provided `Get-Random` cmdlet I wanted to make sure this was a really good RNG to ensure high entropy.  Again we're going with a .NET method for this.  
{% highlight Powershell %}
Function Get-RandomInt {
    <#
    .SYNOPSIS
    More robust method of getting a random number than Get-Random
    .DESCRIPTION
    Leverages the .NET RNGCryptoServiceProvider to retrieve a random number
    .PARAMETER Minimum
    Minimum number for range of random number generation
    .PARAMETER Maximum
    Maximum number for range of random number generation
    .EXAMPLE
    PS> Get-RanomInt -Minimum 1 -Maximum 1000
    838

    will return a random number from between 1 and 1000
    #>
    param (
        [UInt32]$Minimum,
        [UInt32]$Maximum
    )

    $Difference = $Maximum-$Minimum+1
    [Byte[]]$Bytes = 1..4
    [System.Security.Cryptography.RNGCryptoServiceProvider]::Create().GetBytes($Bytes)
    [Int32]$Integer = [System.BitConverter]::ToUInt32(($Bytes),0) % $Difference + 1
    $Integer
}

{% endhighlight %}  
This creates a 4 byte array to feed to the RNDCryptoServiceProvider for randomization and then converts the byte array in to an integer to work with.  Now the relevant section of Get-PSPhrase can look like this:  
{% highlight Powershell %}
$NounsHash = Initialize-Dictionary -Type Nouns
$AdjectivesHash = Initialize-Dictionary -Type Adjectives

$Passphrases = 1..$Settings.Count | ForEach-Object {
    [System.Collections.ArrayList]$WordArray = 1..$Settings.Pairs | ForEach-Object {
        $Number = Get-RandomInt -Minimum 1 -Maximum $AdjectivesHash.Count
        $AdjectivesHash.$Number
        $Number = Get-RandomInt -Minimum 1 -Maximum $NounsHash.Count
        $NounsHash.$Number
    }
{% endhighlight %}  
Read the text files and create the wordlist hashtables.  The first section is a little confusing but we create a number sequence using the `1..5` syntax.  If the 'Count' from settings is 10 this will create the numbers 1 through 10.  These are piped to `ForEach-Object` but rather than do anything with the objects themselves I'm just using it as a way to repeat next operation however many times is dicated by 'Count'.  
The same technique is used for the number of pairs of words to be generated for the passphrase.  Then it's simply generate a random number, get an adjective, generate a random number, get a noun.  These are output and captured by an ArrayList called `$WordArray`.  That's it, that's the meat and potatos.  
## Allow Saving Preferences  
There's quite a few parameters that allow you to control the output of `Get-PSPhrase` and if you use every single one it can be quite a bit to type. Example:  
{% highlight Powershell %}
PS> Get-PSPhrase -Count 10 -Pairs 2 -TitleCase -Substitution -Append '1' -Prepend '!' -Delimiter '-' -IncludeNumber -IncludeSymbol
{% endhighlight %}
```
!-L0c0-3Fl3$h-#P3rf3ct-T3nt-1
!-1Unt1dy$-R3j3ct-$udd3n-Clump-1
!-V3n3r@t3d7-C0mm3nt$-*Cl0$3-T1ngl3-1
!-$n1v3l1ng%-3ng1n38-W3ll-D0cum3nt3d-Pur1t@n-1
!-#B1g-Pr@nk3r-Bubbly5-G0bl1n$-1
!-$Pr3w@r-Bl0@t-Prud3nt-20c3@n-1
!-@n0th3r*-B00m-0bl0ng-6H0m3-1
!-!Up$3t-W0m3n-1Gr@nd10$3-$ph3r3$-1
!-Bl0@t3d$-N0t3$-N0t3d8-R@nch-1
!-%L3th@l2-Cr3w$-Ch@rr3d-G1rl$-1
```  
Imagine you want to run Get-PSPhrase like that every single time because that's your preference.  You could copy that out somewhere and paste it everytime you want use it.  You could wrap it in a function definition you keep in your profile.  You could even put it in a script and then execute that.  I wanted people to be able to save their preference for parameters and values **and** still overwrite that or use other combinations.  
  
The first step was figuring out where to store these preferences, with consideration that I wanted this to be cross platform.  The module could also be installed system wide for all users, or just for a single user.  What I focused on was storing the preferences somewhere specific to the current user's 'Home'.  Detecting if the current computer is Windows, Mac or Linux was the first step and dictates where the settings file will be written:  
{% highlight Powershell %}
# check to see if we're on Windows or not
if ($IsWindows -or $ENV:OS) {
    $Windows = $true
} else {
    $Windows = $false
}
if ($Windows) {
    $SettingsPath = Join-Path -Path $Env:APPDATA -ChildPath "PSPhrase\Settings.json"
} else {
    $SettingsPath = Join-Path -Path ([Environment]::GetEnvironmentVariable("HOME")) -ChildPath ".local/share/powershell/Modules/PSPhrase/Settings.json"
}
{% endhighlight %}  
I decided to go with a json file as it made it easy to write to and read from and still deal with objects.  The function `Set-PSPhraseSetting` would then have the same parameter options as `Get-PSPhrase` but would instead save these parameters to disk.  The called parameters and values are stored in a hashtable called '$Settings' with each parameter name being a key, and the provided value the corresponding value.  The settings hashtable is then converted to json and written to disk.  A switch parameter of `Defaults` is also included which will actually just delete the json file so that Get-PSPhrase goes back to its defaults.  
The `Get-PSPhraseSetting` function is pretty straight forward.  It does the same check for platform type to determine where the json file should be, checks for it, and if found it reads it in, converts it from json and spits out the saved settings.  
{% highlight Powershell %}
PS> Get-PSPhraseSetting

Count IncludeNumber
----- -------------
   20          True

{% endhighlight %}  
Standard PowerShell formatting rules apply; if 4 or less properties it will return a table.  More than that and it will return a list.  
  
Now the tricky part was making it so you could call `Get-PSPhrase` by itself, leveraging the saved preferences, *or* with additional (or the same) parameters and still have it work. I should have started writing this blog post closer to when I was authoring the module just so I could tell you all the ways that **don't** work.  Instead here's what ended up working for me:  
{% highlight Powershell %}
$Settings = [Ordered]@{
    Pairs = $Pairs
    Count = $Count
    Delimiter = $Delimiter
}
if ($DefaultSettings = Get-PSPhraseSetting) {
    foreach ($Setting in $DefaultSettings.PSObject.Properties.Name) {
        if (-not($PSBoundParameters.ContainsKey($Setting))) {
            $Settings.$Setting = $DefaultSettings.$Setting
        }
    }
}
{% endhighlight %}  
Taken from inside Get-PSPhrase.  The first thing we do is start our `$Settings` hashtable to store settings.  Since 'Pairs', 'Count', and 'Delimeter' all have default values in the parameter statement we establish those right off the bat.  Then, leveraging our existing public function that returns a PSCustomObject of any saved settings we combine a check for saved settings with the definition of a variable called `$DefaultSettings` should they exist.  Then the logic after that is something like this:  For each setting found in saved settings, **if** that setting was not explicitly called during the execution of `Get-PSPhrase` then update our `$Settings` hashtable with that setting name and its value.  This works to overwrite those existing default settings if necessary, while also creating new settings entries in the hashtable if they don't exist.  
  
After that it's more or less the same logic I used before for string manipulation.  I use a switch block on the keys in the `$Settings` hashtable and it executes on whatever is present.  
{% highlight Powershell %}
$CultureObj = (Get-Culture).TextInfo

switch ($Settings.Keys) {
    'TitleCase' {
        $WordArray = $WordArray | ForEach-Object {
            $CultureObj.ToTitleCase($_)
        }
    }
    'Substitution' {
        $WordArray = $WordArray -replace "e","3" -replace "a","@" -replace "o","0" -replace "s","$" -replace "i","1"
    }
    'IncludeNumber' {
        $RandomNumber = Get-Random -Minimum 0 -Maximum 9
        $WordIndex = $WordArray.IndexOf(($WordArray | Get-Random))
        $Position = 0,($WordArray[$WordIndex].Length) | Get-Random
        $WordArray[$WordIndex] = $WordArray[$WordIndex].Insert($Position, $RandomNumber)
    }
    'IncludeSymbol' {
        $RandomSymbol = '!', '@', '#', '$', '%', '*', '?' | Get-Random
        $WordIndex = $WordArray.IndexOf(($WordArray | Get-Random))
        $Position = 0,($WordArray[$WordIndex].Length) | Get-Random
        $WordArray[$WordIndex] = $WordArray[$WordIndex].Insert($Position, $RandomSymbol)
    }
    'Append' {
        [Void]$WordArray.Add($Settings.Append)
    }
    'Prepend' {
        $WordArray.Insert(0, $Settings.Prepend)
    }
}
$WordArray -join $Settings.Delimiter
}
$Passphrases
{% endhighlight %}  
The order is somewhat important here as well, and is why the `$Settings` hashtable is ordered. We can't perform the TitleCase or Substitution operations after the Append and Prepend operations as it could take affect on those values, which are intended to be static.  
## Conclusion  
I had fun getting more comfortable with the SamplerModule.  Writing QA and Unit tests was new to me and I felt like I spent a lot of time just figuring out how tests work.  But in the end I'm happy with how the module turned out and I've got a pretty good flow for any changes that need to take place in the future.  

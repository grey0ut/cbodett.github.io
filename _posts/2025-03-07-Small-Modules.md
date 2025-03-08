---
layout: posts
title: Small Modules
classes: wide
date: 2025-03-01
---  

## Size Doesn't Matter   
When I made my first PowerShell module it was mostly an experiment to see if I could understand how they worked.  It was simply `tools.psm1` and it lived in `$HOME\Documents\PowerShell\Modules\Tools`.  That was all it took to move some function definitions out of my $Profile and in to a place where they would automatically load when called.  
This would later grow in to a bigger collection of daily use functions for me and my team.  I think at its peak there were around 44 functions, a couple of class definitions, and a few format files.  Some of the modules you may be familiar with (e.g. ImportExcel, Posh-SSH, PowerCLI) can have upwards of 70 public functions.  These modules represent tremendous effort and provide huge value to the greater PowerShell community.  But not every module needs to be a monolith of code. 
  
Something that Kevin Marquette wrote about in [his blog]("https://powershellexplained.com/2019-04-11-Powershell-Building-Micro-Modules/") that I found very enouraging was the concept of building "micro modules."  Just because a module exports 1 or 2 functions doesn't mean that it's not valuable.  
What I really like about a module is the ability to package up PowerShell functions, and any requirements, in to something that's easy to install and use from the CLI.  You can define classes and format files to go along with your module and the process of getting the module doesn't change one bit.  I could write a script to accomplish a lot of the same end result but here's my general philosphy on PowerShell code:  
- Cmdlets are used in the CLI interactively to perform tasks
- Custom functions are used in the CLI interactively to perform tasks. They expand on existing cmdlets or fill a more custom need.  
- Scripts are an orchestration/automation tool used to perform a set of actions, sometimes unattended  
- Modules are a deliverable way to install functions (and functionality) on to a system  
  
Maybe you've written a logging function that you want to incorporate in to a bunch of scripts you've written that might run unattended.  It could be just 40 lines of code or so, and you could include it at the top of all your scripts. Maybe it lives in a separate .ps1 file and you dot source it at script execution.  By comparison the latter option would make it easier to write changes to your logging module.  **OR** you turn your logging function in to a module and publish it in a repository.  Now it's installable on any machine that you need it on, and you can use built in cmdlets for module management like `Update-Module` to handle changes.  
  
If you're writing PowerShell functions, and you're putting them in to modules, you're making tools.  If that tool happens to do one thing really well that sounds like a pretty good tool to me.  I don't need a screwdriver that's also a flashlight, a radio, and a spatula.  I need a screwdriver.  Preferably one that's so good I never have to think about it. Go ahead and turn that function, or two, in to a shippable module.  

## Some Examples  
I wrote about my [ProtectStrings]({{ "/ProtectStrings" | absolute_url }}) module before and while it took me a lot of work to get it to where I want, it's ultimately two functions: Protect-String, and Unprotect-String.  The other 7 functions are just about supporting the master password in some way.  
  
I also had a weird scenario at work where we needed to change the network profile of a WiFi network from "Public" to "Private" while *not* connected to that network.  This required editing the registry.  To make it easier to do I wrote a function called Set-NetworkProfile that would handle the edits.  If you're making a Set-* function I think it's usually appropriate to make a Get-* one as well and think about how they might be used together.  
Pretty quickly this turned in to a small module with 3 public functions, 1 private, and a format file for forcing the output to be a table.  The module does one thing, and does it well and because of the fact that it's a module it's easy to deploy to machines, and allowed me to hide a private function from the end-user and control the output.  You can see more about it [here]("https://github.com/grey0ut/NetworkProfile").  
  
Another recent one was born from a very silly use case.  A peer wanted help putting an easter egg in his large script the involved some ASCII art and some other stuff.  He didn't want people to be able to open the script in an editor and see this so the goal was obfuscation.  I suggested base64 at first but the resulting block of text was quite large.  We did a little bit of research and ended up using the [deflatestream]("https://learn.microsoft.com/en-us/dotnet/api/system.io.compression.deflatestream?view=net-9.0") .NET class to compress the data and then expand it during script execution when the easter egg was triggered.  I got back to him a little bit later to show him I had written `Compress-String` and `Expand-String` and was working on making Compress-String accept pipeline input.  At the time neither of us could think of a single thing to do with these functions so I stowed them away and forgot about them.  
Months later I found them again and decided that they should be in a module and spent way too much time trying to think of a clever name instead of doing anything with the code. I recall one of the names I picked already belonged to an existing project on Github but I noticed that they hadn't properly handled pipeline input so this told me I still had something to offer.  
[ComPrS]("https://github.com/grey0ut/ComPrS/") is live now after finding another legitimate use case for it.  Microsoft published a detection script for a vulnerability and the script calls out to Github to download a CSV of hashes and check against them.  The script was appreciated but we had a lot of endpoints to run it on and many of them didn't have the ability to connect to Github.  In the interest of portability I put the data right in the script, adding over 3700 lines to it.  As an exercise I included a copy of my Expand-String function in the script and compressed the reference data using Compress-String and ConvertTo-HereString.  This shortened the addition to the script significantly but not anythign earth shattering.  
  
## OK so what?  
![AustinPowers]({{"/assets/images/Small_Modules_01.png" | absolute_url }})  
Building a small module is a great way to focus on making a tool that does something well, and to get you in the headspace of a tool maker.  Asking yourself questions about how other people might use this tool or what functionality they might want out of it.  Try to make it as applicable as possible for its use case.  
  
Also, it's a great way to keep learning PowerShell and get in to module making without feeling intimidated.

Now I think I want to turn my script "New-NaturalLanguagePassword" in to a module so that it's easier to install and use from the command line.
![Ihr1]({{"/assets/images/Invoke_History_01.png" | absolute_url }})  

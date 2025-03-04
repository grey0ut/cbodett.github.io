---
layout: posts
title: Small Modules
classes: wide
date: 2025-03-01
---  

## Size Doesn't Matter   
When I made my first PowerShell module it was mostly an experiment to see if I could understand how they worked.  It was simply `tools.psm1` and it lived in `$HOME\Documents\PowerShell\Modules\Tools`.  That was all it took to move some function definitions out of my $Profile and in to a place where they would automatically load when called.  
This would later grow in to a bigger collection of daily use functions for me and my team.  I think at its peak there were around 44 functions, a couple of class definitions, and a few format files.  Some of the modules you may be familiar with (e.g. ImportExcel, Posh-SSH, PowerCLI) can have upwards of 70 public functions.  These modules represent tremendous effort and provide huge value to the greater PowerShell community.  
  
Something that Kevin Marquette wrote about in [his blog]("https://powershellexplained.com/2019-04-11-Powershell-Building-Micro-Modules/") that I found very enouraging was the concept of building "micro modules."  Just because a module exports 1 or 2 functions doesn't mean that it's not valuable.  What I really like about a module is the ability to package up PowerShell functions, and any requirements, in to something that's easy to install and use from the CLI.  You can define classes and format files to go along with your module and the process of getting the module doesn't change one bit.  I could write a script to accomplish a lot of the same end result but here's my general philosphy on PowerShell code:  
- Cmdlets are used in the CLI interactively to perform tasks
- Custom functions are used in the CLI interactively to perform tasks
- Scripts are an orchestration/automation tool used to perform a set of actions, sometimes unattended  
  
With that in mind, packaging up functions in to modules 

![Ihr1]({{"/assets/images/Invoke_History_01.png" | absolute_url }})  
{% highlight Powershell %} 
# code goes here

Get-ChildItem -Path $Stuff
{% endhighlight %} 

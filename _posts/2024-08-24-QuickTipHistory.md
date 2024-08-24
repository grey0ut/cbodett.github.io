---
layout: posts
title: Quick Tip Invoke-History
classes: wide
date: 2024-8-24
---  

The more time I spend living in the CLI the more I appreciate learning and adopting shorthand for operations.  In Powershell the aliases for Where-Object and ForEach-Object have become second nature.  Using up arrow to repeat the previous command and add more to it a near constant occurence.  One situation I find myself in quite a bit however is running a command in Powershell, and then finding that based on the output I'd actually like to re-run that command an get the value from a property instead.  On the keyboard I've been using for the past couple of years I would typically just hit up-arrow to repeat the last command, `Home` to put my cursor on the beginning of the line, type a `(` then `End` a closing `)` then use dot notation to call the property I wanted the value of.  
  
As an example let's say I run a script and see what the output looks like:
![Ihr1]({{ "/assets/images/Invoke_History_01.png" | absolute_url }})  
From this I observe the output and decide that I'd like to add some parameters.  No problem, I'll just up-arrow to repeat the command and add the parameters to the end:  
![Ihr2]({{ "/assets/images/Invoke_History_02.png" | absolute_url }})  
Great, but if I wanted to call a property/method on that output object I'd either have to pipe it to another cmdlet or wrap it in parenthesis and then call the property I want with the dot shortcut. I.e.:  
![Ihr3]({{ "/assets/images/Invoke_History_03.png" | absolute_url }})  
Seems simple enough. Home key, parenthesis, End key, parenthesis, dot, property name.  But, my new keyboard is a 60% and doesn't have dedicated arrow keys requiring that I hold another key to access a layer that has arrow keys on it.  
# Invoke-History Has Entered the Chat  
I remebered that along with Get-History there was Invoke-History and its alias of 'r'.  I've previously used this similarly to repeating commands in bash.  Get-History, find the number of the command, then use `r <num>` to repeat:  
![Ihr4]({{ "/assets/images/Invoke_History_04.png" | absolute_url }})  
Through experimentation I found that calling `r` by itself repeats the previous command by default.  
![Ihr5]({{ "/assets/images/Invoke_History_05.png" | absolute_url }})  
The next thing I tried was wrapping 'r' in an expression to see if I could then use the dot shortcut to retrieve a property or method:  
![Ihr6]({{ "/assets/images/Invoke_History_06.png" | absolute_url }})  
And shazam! Now instead of needing arrow keys or Home/End keys I can start from a fresh prompt and type `(r)` following by whatever it was I wanted to do on the previous operation.  
![MindBlown]({{ "/assets/images/mind-blown.gif" | absolute_url }})  


---
layout: posts
title: Status Update
classes: wide
date: 2023-08-20
---  
Hi all.  Just wanted to provide a brief status update.  It's been a while since my last post and while I have been busy, and making frequent use of Powershell, I haven't had anything novel that I felt like sharing.  
  
I've still been using the Get-GeoLocation function quite a bit as well as another function I wrote called Get-WhoIsIP.  It's nothing crazy and primarily leverages "http://ipwho.is" API for results.  I spent a lot of time using Powershell as a CLI and want a way to quickly look up IP addresses to determine ownership.  Sometimes lots of IP addresses.  
  
Primarily I would say that I've had a lot of occasion to help other people with their Powershell related needs.  Here are some highlight topics I can think of:  
- modified a Domain Join script to handle adding a computer to groups as part of the process.   
- A script as part of a Scheduled Task that emails a CSV of specific types of accounts that need to change their password (based on age) 
- A script for someone that needs to rename hundreds of files based on string text found within.  Was previously a manual process, now thanks to Powershell (and Regex) something that used to take hours and hours each week takes a couple of seconds.  
- A couple different Active Directory off-boarding scripts to consistently handle removing accounts.  
- A script that resets an Active Directory user's password expiry clock.  Effectively changing the "Password Last Set" time to that of script execution.  
- A series of scripts with Scheduled Task setups/executions, data output, and data collection. Heavily relying on DPAPI and AES encryption for data protection.  The "psuedo code" is basically; execute script, save data, acquire data, alert on data.  This also involves build scripts, deployment scripts, removal scripts and a few Scheduled Tasks.  Was a lot of fun to write.  
- Helped on a CTF that involved a lot of deobfuscating Powershell and finding flags within the code.  
  
That's about it.  I'm still looking for the idea that's going to inspire me to write another Powershell module. For now I'll keep maintaining my team's internal module, and my publicly available ProtectStrings module.  
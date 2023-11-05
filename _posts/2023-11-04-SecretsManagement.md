---
layout: posts
title: SecretStore Module
classes: wide
date: 2023-11-04
---   

SecretManagement module is a Powershell module intended to make it easier to store and retrieve secrets.  
> The secrets are stored in SecretManagement extension vaults. An extension vault is a PowerShell module that has been registered to SecretManagement, and exports five module functions required by SecretManagement. An extension vault can store secrets locally or remotely. Extension vaults are registered to the current logged in user context, and will be available only to that user (unless also registered to other users).  
  
[SecretManagement Module on Github](https://github.com/PowerShell/SecretManagement)  
  
This is a really cool project and an awesome tool that Microsoft created.  I see it get referred to a lot in different Powershell communities as a recommended solution for dealing with secrets in automation.  I haven't had any occasion to use it myself but I had often thought about writing a Powershell based password manager (until SecretManagement was released).  
  
Relevant to my interests then if you want to just store secrets locally on your computer for use in scripts you'll want to look at the SecretStore module.  
[SecretStore Module on Github](https://github.com/PowerShell/SecretStore)  
> It stores secrets locally on file for the current user account context, and uses .NET crypto APIs to encrypt file contents. Secrets remain encrypted in-memory, and are only decrypted when retrieved and passed to the user. This module works over all supported PowerShell platforms on Windows, Linux, and macOS.  
  
Since theirs is cross platform and mine isn't it's probably different .NET in the backend, but the class is likely the same.  For reference, in .NET it's referred to by its RFC "Rfc2898DeriveBytes".  Since this is all publicly available on Github I thought I would search through the relevant CS code and try to understand how they did it differently.  Here is the file I found where I believe PBKDF2 is happening:  
[Utils.cs](https://github.com/PowerShell/SecretStore/blob/master/src/code/Utils.cs)  
  
There's two sections that drew my attention.  The first:  
{% highlight Powershell %}
private static byte[] DeriveKeyFromPassword(
    byte[] passwordData,
    int keyLength)
{
    try
    {
        using (var derivedBytes = new Rfc2898DeriveBytes(
            password: passwordData, 
            salt: salt, 
            iterations: 1000))
        {
            return derivedBytes.GetBytes(keyLength);
        }
    }
    finally
    {
        ZeroOutData(passwordData);
    }
}
{% endhighlight %}  
  
And the second:  

{% highlight Powershell %}
private static AesKey DeriveKeyFromKeyAndPasswordOrUser(
        SecureString passWord,
        AesKey key,
        bool useOrigUserNameCasing = false)
    {            
        var passWordData = GetPasswordOrUserData(passWord, useOrigUserNameCasing);
        try
        {
            byte[] newKey;
            using (var derivedBytes = new Rfc2898DeriveBytes(
                password: passWordData, 
                salt: key.Key, 
                iterations: 1000))
            {
                newKey = derivedBytes.GetBytes(key.Key.Length);
            }

            byte[] newIV;
            using (var derivedBytes = new Rfc2898DeriveBytes(
                password: passWordData,
                salt: key.IV,
                iterations: 1000))
            {
                newIV = derivedBytes.GetBytes(key.IV.Length);
            }

            return new AesKey(
                key: newKey,
                iv: newIV);
        }
        finally
        {
            ZeroOutData(passWordData);
        }
    }
{% endhighlight %}  

# Backstory  
  
Remember how quickly everyone bailed on LastPass after their breach?  It was revealed that they were using PBKDF2 with SHA256 and only 100,001 iterations ([Article from The Verge](https://www.theverge.com/2022/12/28/23529547/lastpass-vault-breach-disclosure-encryption-cybersecurity-rebuttal)).  In some cases, if the account was older, it was even worse than that at only 5000 iterations.  For comparison, OWASP has been recommending 310,000 iterations (SHA256) since at least 2021 and as of today recommends 600,000.  
  
Essentially, LastPass left people's vaults more vulnerable to brute force attack (after the breach) because it wasn't computationally costly to iterate through millions of attempted master passwords.  
  
# The Problem  
  
Having spent time in Powershell leveraging the Rfc2898DeriveBytes class (PBKDF2) quite a bit with my ProtectStrings module I spent a bit of time reading up on the available hash algorithms, recommended salt lengths and iteration counts and made sure to pick values that I thought would make it not worth while to try to brute force any keys generated using my code.  
  
I was trying to find a deep dive article anywhere online that got in to how the SecretStore module functioned at a cryptographic level but couldn't find anything.  I'm still not 100% sure at what point these implementations of PBKDF2 are being leveraged but from what I can tell the way they're being called leverages the default constructors.  
Reading the actual RFC it appears that the default constructor ([from 2000](https://www.rfc-editor.org/rfc/rfc2898)) leverages SHA1 and 1000 iterations.  
  
Now, it would certainly not be an easy task to brute force this, but this is production code being advertised for essentially the same purpose as LastPass: protect your secrets.  In this module's case the data would be stored locally on your computer, as compared to someone else's server, but the fact remains that this code is leveraging 23 year old constructors that *even* Microsoft says are deprecated.  
![PBKDF2Constructors]({{ "/assets/images/PBKDF2_01.png" | absolute_url }})  
![PBKDF2]({{ "/assets/images/PBKDF2_02.png" | absolute_url }})  
  
Seems like poor form for Microsoft to call out all these deprecated constructors as obsolete and insecure and then go ahead and use them in their Powershell module specifically made for storing secrets.  
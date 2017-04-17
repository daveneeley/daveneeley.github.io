---
layout: post
title:  "FamilySearch GEDCOM Export"
date:   2017-04-17 02:01:00
categories: family-history 
---
Today I was able to export my ancestry from [Family Search](https://familysearch.org) into GEDCOM format, and then import it into [Family Tree DNA](https://www.familytreedna.com/). I only exported seven generations back, which was far enough to encompass both my last confidently known paternal and maternal ancestors. I am so grateful for the countless hours of work done by many people to make the family tree available, as well as to make the export possible. Truly, we stand on the shoulders of giants. 

According to [wikipedia.org](https://en.wikipedia.org/wiki/GEDCOM), "GEDCOM (an acronymn for Genealogical Data Communication) is an open de facto specification for exchanging genealogical data between different genealogy software. GEDCOM was developed by the Church of Jesus Christ of Latter-day Saints (LDS Church) as an aid to genealogical research." It's pronounced JED-com, like Uncle Jed from The Beverly Hillbillies. Even though it's an old-school sequential text format, it works! In particular, it's the only import format supported by Family Tree DNA.

It's not possible to export directly to GEDCOM from Family Search today. But attention to detail, a healthy helping of perseverance, and perhaps a little programmer laziness can do wonders for finding solutions.  It's super-easy to export from [Family Search](https://familysearch.org) to [Ancestry](https://www.ancestry.com). When I learned that GEDCOM export was not supported, I started digging around. I read about lots of old niche software that supports GEDCOM. But none of this stuff seems to have kept up with the web 2.0 revolution. Finally, in some obscure help article, a kind soul posted a link to a python script. Aha! We're in business. A GitHub user by the handle 'freeseek' took the time to learn the Family Search API *and* the GEDCOM format *and* shared the work with the world, *for free*. I love open-source!

The script is called [getmyancestors.py](https://github.com/freeseek/getmyancestors), and it's well documented. However, python probably isn't yet a mainstream tool for most family history researchers. My goal is to provide a walkthrough that modern Windows PC users can follow to use this awesome tool. There's a suggested python distribution in the script's README, but it was really large (422MB). I'm not sure my walkthrough is any better, honestly.

The first thing to leverage is the command prompt. It's been around since Windows 3.1, and many people still call it the DOS prompt. The next tool is called Chocolatey, which is a package manager for installing windows-based utilities. The third tool is Git, which is a source-code management system (the getmyancestors.py script is source-code). The next tool is python, a programming language that can be used on many different operating systems such as Windows, Mac, and Linux. And finally, the getmyancestors python script itself.

## Adminstrative Command Prompt

The Command Prompt is already on your Windows PC. We'll need to run it as an dminstrator in order to install the other tools. Just search for cmd in the windows start menu, then right-click it, and select Run As Administrator. 

## Chocolatey

Go to [chocolatey.org/install](https://chocolatey.org/install) to copy the one-line string that will automatically download and install chocolatey for you to your clipboard. As of this writing, that string is:

{% highlight powershell %}
@powershell -NoProfile -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"
{% endhighlight %}

The way to paste this into a command prompt is to right-click anywhere inside the command prompt window and select 'paste'. I promise, this is the hardest part.

## Other Tools

With chocolatey installed, you can run simpler commands to install everything else. In the same command-prompt window, type the next three lines, hitting enter after each.

{% highlight powershell %}
choco install git
choco install python3
python -m pip install requests
{% endhighlight %}

This will install git, python3, and an extra python script that is required to run getmyancestors on your PC.

## Download the script

Now, close the command prompt you just opened, and open a new one. You don't need to be an administrator anymore. But more importantly, in your new window the folder location will be different. Typically, it's set to c:\users\<your user name>. From here you can change the directory to inside your documents folder and make a copy of the script.

{% highlight powershell %}
cd documents
git clone https://github.com/freeseek/getmyancestors.git
{% endhighlight %}

This will make a new folder called getmyancestors inside your documents folder. 

## Export to GEDCOM

At this point, you are ready to run the first example from the README. You'll want to change directory into the getmyancestors folder, and then run the script.

{% highlight powershell %}
cd getmyancestors
python .\getmyancestors.py
{% endhighlight %}

This will prompt you to login to Family Search, and then, after a minute or two, will print out a GEDCOM file on the screen. Just do this to check that everything works. The second example will do the same, but save the results to a file. You can then take the file and import it into any system that supports GEDCOM.

{% highlight powershell %}
python .\getmyancestors.py -o out.ged
{% endhighlight %}

And that's it. You can upload the out.ged file to the system of your choice. I'm sure I glossed over some things that might be a bit foreign to the average windows user. Feel free to contact me for help. Maybe I'll write a web-service to do this if there's interest.

Here's the whole thing, which you can try in a windows .bat file.
{% highlight powershell %}
@powershell -NoProfile -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"
choco install git
choco install python3
python -m pip install requests
cd %USERPROFILE%\Documents
git clone https://github.com/freeseek/getmyancestors.git
python .\getmyancestors.py -o out.ged
{% endhighlight %}

---
layout: post
title: "Azure Command Line Tools: Cross Platform CLI aka xplat-cli"
excerpt: "Azure management and automation"
categories: articles
tags: [Azure, Tools]
comments: true
share: true
---

If you are launching an occasional VM, administering a small environment on Azure, the management console does the job but if you need to deal with multiple subscriptions and get things done pretty quick, cli tools are must. Fortunately, you have multiple choices with Azure here - 

1. Obviously [Powershell](http://azure.microsoft.com/en-in/documentation/articles/install-configure-powershell/). [Resistance is futile](http://www.infoworld.com/article/2614339/windows-server/resistance-is-futile--you-will-use-powershell.html).  
2. For non windows environments [Azure cross Platform CLI](http://azure.microsoft.com/en-in/documentation/articles/xplat-cli/). 
3. And then there is [Python](http://azure.microsoft.com/en-in/documentation/articles/cloud-services-python-how-to-use-service-management) via Azure SDK for Python. 

I still haven't tried enough to figure out the extent to which you can manage using Python but the other two options are pretty powerful. Though Microsoft has excellent documentation I thought I'd write down the steps to configure these tools, for my clarity. 


## Setting up Azure cross platform CLI

I prefer this over Powershell because your scripts can run anywhere as long as you have NodeJs. I work predominantly on a windows machine so the steps below are applicable for windows platform. 

If you are running a windows machine, do yourself a favor and install [Chocolatey](https://chocolatey.org/). Chocolatey is `yum install` equivalent for Windows crowd and it is awesome. Microsoft has actually shipped a package manager called [OneGet](http://blogs.msdn.com/b/garretts/archive/2014/04/01/my-little-secret-windows-powershell-oneget.aspx) on Windows 10 Technical Preview and Chocolatey is one of the package sources. Installing it is ridiculously easy. Open an elevated command prompt and paste the following

{% highlight bash %}
@powershell -NoProfile -ExecutionPolicy unrestricted -Command "iex ((new-object net.webclient).DownloadString('https://chocolatey.org/install.ps1'))" && SET PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin
{% endhighlight %}

Once its installed, you can use `choco install myawesomepackage` to install wide variety of software. Coming back to installation of NodeJs, just enter this

{% highlight bash %}
choco install nodejs
{% endhighlight %}

Its that simple. Once done, use `npm` to install Azurecli

{% highlight bash %}
npm install azure-cli -g
{% endhighlight %}

After installation, if you just type `azure` you'll see info and help, including awesome ASCII art :)

{% highlight text %}
C:\WINDOWS\system32>azure
info:             _    _____   _ ___ ___
info:            /_\  |_  / | | | _ \ __|
info:      _ ___/ _ \__/ /| |_| |   / _|___ _ _
info:    (___  /_/ \_\/___|\___/|_|_\___| _____)
info:       (_______ _ _)         _ ______ _)_ _
info:              (______________ _ )   (___ _ _)
info:
info:    Microsoft Azure: Microsoft's Cloud Platform
info:
info:    Tool version 0.8.12
help:
help:    Display help for a given command
help:      help [options] [command]
help:
help:    Log in to an Azure subscription using Active Directory. Currently, the user can login only via Microsoft organizational account
{% endhighlight %}

## Connect to Azure

Now that we have installed cli, lets set it up and play around a bit. There are two ways you could login - Using an organizational account (Azure AD Login) or a Microsoft live account. If you have Azure AD login, you can connect using `azure login -u username -p password`. I have a live account to which multiple subscriptions are attached so I first downloaded the azure publish settings file.

{% highlight bash %}
azure account download
{% endhighlight %}

This will launch your default browser. Signing with your live credentials and download the settings file
{% highlight bash %}
azure account import "E:\Azure Pass-1-25-2015-credentials.publishsettings"
{% endhighlight %}

If your account has multiple subscriptions attached to it, you might want to make sure that you are connected to the right subscription before doing anything
{% highlight bash %}
azure account list
{% endhighlight %}

Set the account you choose to use
{% highlight bash %}
azure account set "Azure Pass"
{% endhighlight %}

Azurecli has two modes - *Service Management Mode* and *Resource Management Mode*. Resource Manager is a relatively recent addition to Azure and its available on new portal. It allows you to manage a group of resources as a logical unit. Example - a bunch of servers on SQL Server AlwaysOn setup. There are a bunch of templates available which can be downloaded and modified. At the time of this writing, these two cli modes are exclusive and you can switch between them using commands below

{% highlight bash %}
azure config mode arm
azure config mode asm
{% endhighlight %}








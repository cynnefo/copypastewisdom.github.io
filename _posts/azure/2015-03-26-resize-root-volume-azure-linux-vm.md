---
layout: post
title: "Azure Linux VM - Resize root volume"
excerpt: "How do you resize an Azure root volume?"
categories: articles
tags: [Azure]
comments: true
share: true
---

So you are stuck. You provisioned an Azure linux VM back when the maximum size of OS drive was 127 GB and for whatever reason, your root volume is filling up and you would like to extend it. Microsoft has [recently removed this restriction](http://azure.microsoft.com/blog/2015/03/25/azure-vm-os-drive-limit-octupled/) and it can now be 1023 GB. I have been there too and naturally, I took to Google and find a way around. All these proposed solutions were rather elaborate and required deletion of the VM without deleting attached disks, extending the disk using a tool like cloud explorer etc. I was almost tempted to try these steps on [LessThanDot Blog](http://blogs.lessthandot.com/index.php/enterprisedev/cloud/azure/expanding-an-existing-azure-vm-system-drive/) but the prospect of deleting the VM frightened me. 

I was stuck till I stumbled upon this azure powershell cmdlet - `Update-AzureDisk`. It looks like you can shutdown the VM, and using `ResizedSizeInGB` switch, you can extend the volume. To test this theory, I quickly launched a VM. 

In order to follow this tutorial, you need to have [Azure Powershell](http://azure.microsoft.com/en-us/documentation/articles/powershell-install-configure/) installed on your machine.

Use `Get-AzurePublishSettingsFile` to download your settings file and import

{% highlight powershell %}
Import-AzurePublishSettingsFile C:\Users\RK\Downloads\Pay-As-You-Go-3-26-2015-credentials.publishsettings
{% endhighlight %}
Once you are in, create a test linux VM to play around with. Mine was an `A1` Ubuntu server. As you can see below, the root volume is 30 GB

<figure>
	<img src="/images/resizenix1.png" alt="image">
</figure>  

To resize this, first shutdown the vm:

{% highlight powershell %}
Get-AzureVM -ServiceName "foundation" -Name "Trantor" | Stop-AzureVM -Force
{% endhighlight %}

Get the OS disk attacked to this VM:

{% highlight powershell %}
Get-AzureVM -ServiceName "foundation" -Name "Trantor" | get-AzureOSDisk
{% endhighlight %}

<figure>
	<img src="/images/resizenix-diskname.png" alt="image">
</figure> 

Run `Update-AzureDisk` to extend the root volume from 30 GB to 160 GB (or your desired size which is < 1023 GB)

{% highlight powershell %}
Update-AzureDisk –DiskName "Trantor-Trantor-0-201503261541080008" -Label "ResiZedOS" -ResizedSizeInGB 160
{% endhighlight %}

<figure>
	<img src="/images/resizenix-succeeded.png" alt="image">
</figure> 

And boom! it looks like it succeeded. Lets check inside the server and see if it reflects. 

<figure>
	<img src="/images/resizenix-succeeded2.png" alt="image">
</figure> 

It sure does!

**Watch out!** I didn't find documentation on MSDN supporting usage of this cmdlet on OS Disk. While it worked perfectly, I am still not sure whether Microsoft officially supports this way. So Please don't blame if you servers get lost into a galaxy far far away! :) 
{: .notice} 


 
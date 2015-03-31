---
layout: post
title: "Virtual Lab: Multi node Hadoop cluster on windows - HortonWorks, HDInsight"
categories: articles
excerpt: "a step by step guide to deploying a hadoop cluster on windows"
tags: [bigdata]
comments: true
share: true
---
<section id="table-of-contents" class="toc">
  <header>
    <h3>Overview</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->

Microsoft recently [announced HDInsight](http://blogs.technet.com/b/serverandtools/archive/2012/10/24/microsoft-hdinsight-server-for-windows-and-windows-azure-hdinsight-service-announced.aspx), built on top of[HortonWorks DataPlatform](http://hortonworks.com/products/hdp/) (a distribution of Hadoop similar to Cloudera). There is a[developer preview of HDInsight](http://social.technet.microsoft.com/wiki/contents/articles/14141.installing-the-developer-preview-of-hdinsight-services-on-windows.aspx) available as a free download from Microsoft, which is very easy to setup. However,  the [developer preview is a single box solution](http://social.msdn.microsoft.com/Forums/windowsazure/en-US/a30e726a-c907-4642-a8da-4188b4c60c6d/installaling-hdinsight-on-multi-nodes) and it is not possible to simulate a real world deployment with multi node clusters. You can create a 4 node hadoop cluster on Azure with a few clicks but it is [prohibitively costly](http://www.windowsazure.com/en-us/pricing/details/hdinsight/) (and the cluster will be shutdown after free tier usage, if your account is a trial one). This is a step by step guide to setup a multi node cluster for free on your laptop using HortonWorks Data Platform. HDInsight is actually built on top of Hortonworks solution, so there won’t be many differences apart from some usability features and nice looking clients.  So let’s get started with the setup:

### What will we build?

We will build a Virtualbox Base VM with  all the necessary software installed, sysprep it and spin linked clones from the base for better use of available resources, We will build a domain controller and add 4 VMs to the domain. You can achieve similar setup using HyperV too, using differential disks. We should end up with something like this:

**Domain: TESLA.COM**

</div><table border="0" cellpadding="2" cellspacing="0" style="text-align: justify; width: 657px;"><tbody><tr><td valign="top" width="118"><div style="text-align: left;">**Name of the VM**</div></td><td valign="top" width="334"><div style="text-align: left;">**Description**</div></td><td valign="top" width="91"><div style="text-align: left;">**IP Address**</div></td><td valign="top" width="112"><div style="text-align: left;">RAM</div></td></tr><tr><td valign="top" width="118">HDDC</td><td valign="top" width="334">Domain Controller and Database host for Metastore</td><td valign="top" width="91">192.168.80.1</td><td valign="top" width="112">1024</td></tr><tr><td valign="top" width="118">HDMaster</td><td valign="top" width="334">Name node, Secondary Name node, Hive, Oozie, Templeton</td><td valign="top" width="91">192.168.80.2</td><td valign="top" width="112">2048</td></tr><tr><td valign="top" width="118">HDSlave01</td><td valign="top" width="334">Slave/Data Node</td><td valign="top" width="91">192.168.80.3</td><td valign="top" width="112">2048</td></tr><tr><td valign="top" width="118">HDSlave02</td><td valign="top" width="334">Slave/Data Node</td><td valign="top" width="91">192.168.80.4</td><td valign="top" width="112">2048</td></tr><tr><td valign="top" width="118">HDSlave03</td><td valign="top" width="334">Slave/Data Node</td><td valign="top" width="91">192.168.80.5</td><td valign="top" width="112">1024</td></tr></tbody></table><div style="text-align: justify;">In prod environments, HIVE, OOZIE and TEMPLETON usually have separate servers running the services, but for the sake of simplicity, I chose to run all of them on a single VM.  If you don’t have enough resources to run 3 data /slave nodes, then you can go with 2\. Usually 2 slave nodes and 1 master is necessary to get a feel of real Hadoop Cluster.

### System Requirements:

You need a decent laptop/desktop with more than 12 GB of RAM and a decent processor like core i5\. If you have less RAM, you might want to run a bare minimum of 3 node cluster (1 master and 2 slaves) and adjust the available RAM accordingly.  For this demo, check the way I assigned RAM in the table above. You can get away with assigning lesser memory to data nodes.

### Software Requirements:

 - [Windows Server 2008 R2 or Windows Server 2012 Evaluation edition, 64 bit](http://www.microsoft.com/en-in/download/details.aspx?id=11093)
 - [Microsoft .NET Framework 4.0](http://www.microsoft.com/en-in/download/details.aspx?id=17718)
 - [Microsoft Visual C++  2010 redistributable, 64 bit](http://www.microsoft.com/en-in/download/details.aspx?id=14632)
 - [Java JDK 6u31 or higher](http://www.oracle.com/technetwork/java/javase/downloads/jdk-6u31-download-1501634.html)
 - [Python 2.7](http://www.python.org/getit/)
 - [Microsoft SQL Server 2008 R2](http://www.microsoft.com/en-in/download/details.aspx?id=8158) (if you would like to use it for Metastore)
 - [VirtualBox for virtualization](https://www.virtualbox.org/wiki/Downloads) (Or HyperV, if you prefer)[HortonWorks DataPlatform for Windows](http://hortonworks.com/partners/microsoft/)
 
### Building a Base VM:

This guide assumes that you are using VirtualBox for virtualization. The steps are similar even if you are using HyperV with minor differences like creating differencing disks before creating VM clones from it. Hadoop cluster setup involves installing a bunch of software and setting environment variables on each server. Creating a base VM and spinning linked clones from it makes it remarkably easier and saves lot of time and it helps creating more VMs within the available limits of your disk space compared to stand alone VMs.

 -  Download and Install VirtualBox
 -  Create a new VM for a base VM. I called mine `MOTHERSHIP`. Select 25 GB of initial space, allowing dynamic expansion. We will do a lot of installations on this machine and to make them run quick, assign maximum amount of RAM temporarily. At the end of the setup, we can adjust the RAM to a lesser value.
 -  Enable two network adapters on your VM. `Adapter1`connected to NAT (if you care about internet access on VMs).`Adapter2` for Internal only Domain Network.
 -  [![image](http://lh6.ggpht.com/-aP0oBj5IqNY/UfEfr0jVkFI/AAAAAAAASXw/JOZcU21R2Cs/image_thumb.png?imgmax=800 "image")](http://lh6.ggpht.com/-Xp0HrP8-FV4/UfEfrJCY5sI/AAAAAAAASXo/t9o-0EsMNa4/s1600-h/image%25255B2%25255D.png) 
 - [![image](http://lh3.ggpht.com/-c3z0iTLoxfE/UfEftWq4MKI/AAAAAAAASYA/cjOiFKwXnBs/image_thumb%25255B1%25255D.png?imgmax=800 "image")](http://lh5.ggpht.com/-0P2J5pEQqEU/UfEfsi0pjLI/AAAAAAAASX4/U38za_CytQU/s1600-h/image%25255B5%25255D.png)
 - Start the VM and boot from the iso. Perform Windows Installation. This step is straight forward. 
 -  Once the installation is done, set password to `P@$$w0rd1` and login.
 - Install guest additions for better graphics, shared folders etc.
 - You will move a lot of files from your Host machine to the VM, so configuring a shared folder will help.
 - [![image](http://lh6.ggpht.com/-3rnzCIG2yFw/UfEfu1BdToI/AAAAAAAASYQ/LlJNILftKXI/image_thumb%25255B2%25255D.png?imgmax=800 "image")](http://lh3.ggpht.com/-mf8SkZKQa58/UfEfuMr5VpI/AAAAAAAASYI/dRfIW7xYQ6Q/s1600-h/image%25255B8%25255D.png)
 
### Preparing the BaseVM:
 -  Login to the base VM you built above and install `Microsoft .NET Framework 4.0`
 -  Install `Microsoft 2010 VC ++ redistributable package`
 - Install `Java JDK 6u31` or higher. Make sure that you select a destination directory that doesn’t have any spaces in the address during installation. For example, I installed mine on `C:\Java\jdk1.6.0_31` 
 - Add an environment variable for Java. If you installed Java on ` C:\Java\jdk1.6.0_31` you should add that path as a new system variable.  Open a Powershell windows as an admin and execute the below command: `[Environment]::SetEnvironmentVariable("JAVA_HOME","c:\java\jdk1.6.0_31","Machine"`
 - Install Python. Add Python to Path. Use the command below in a powershell shell launched as Administrator `$env:Path += ";C:\Python27"`
 - Confirm that the environment variables have been set properly by running `%JAVA_HOME%` from Run, it should open the directory where JDK was installed. Also, run `Python` from Run and it should open Python shell
 - We are almost done with the Base VM. We can now generalize this image using `sysprep`. Go to `Run`, type `sysprep` and enter. It should open /system32/sysprep directory. Run `sysprep.exe` as an Admin. Select `System Out of Box Experience`, check `Generalize` and select **Shutdown.**
 - [![image](http://lh6.ggpht.com/-wvonh7kHuZw/UfIRgPSH7yI/AAAAAAAASYo/0xJzHA7RksU/image_thumb3.png?imgmax=800 "image")](http://lh4.ggpht.com/-tgaJbcGU5po/UfIRff7I8uI/AAAAAAAASYg/6QvCVl8EpOQ/s1600-h/image11.png)
 
### Create Linked Clones:
 -  Go to VirtualBox, Right click on the Base VM you created above and select clone.
 - [![image](http://lh5.ggpht.com/-KoHYQe0fewQ/UfIRh3Vv65I/AAAAAAAASY4/G8gD499anbw/image_thumb4.png?imgmax=800 "image")](http://lh6.ggpht.com/-Axp_FrhLnFc/UfIRg-DEhII/AAAAAAAASYw/NSOcyZ4XdA4/s1600-h/image14.png)
 - Give the VM an appropriate name. In my case, its `HDDC` for domain controller. Make sure you select**Re-Initialize MAC Addresses of all network cards**
 - In the**Clone Type**Dialog, select**Linked Clone,**click**Clone.**
 - [![image](http://lh5.ggpht.com/-S8DoH3rBRbY/UfIRjP-ikQI/AAAAAAAASZI/ECQ2gopGqQs/image_thumb5.png?imgmax=800 "image")](http://lh4.ggpht.com/-iVb-NpWnWSU/UfIRiRRIchI/AAAAAAAASZA/wmESIVsrvYQ/s1600-h/image17.png)
 - **Repeat** steps # 1 – 5 for creating other VMs that will be part of your cluster. You should end up with `HDDC` , `HDMaster` and slave nodes - `HDSlave01`, `HDSlave02`, `HDSlave03` (depending on whether your resources allow you create these many nodes)
 - Since we created these clones from the same base VM, it will inherit all settings, including memory. You might want to go to the properties of each VM and assign memory as you see fit.
 
### Configuring Domain on HDDC
 We will not go into details of setting up active directory. Google is your friend. But below are some steps specific to our setup.
 
 - Start HDDC VM and login.
 - VirtualBox creates funny hostnames names, even though your VM name is HDDC. Change the computer name to `HDDC` and reboot.
 - Go to the network settings and you should see two network adapters enabled. Usually `Network Adapter2` is your domain only internal network. To confirm, you can go to VM settings while the VM is running, go to**Network**tab and temporarily simulate cable disconnection by unchecking**cable connected.**The Adapter attached to NAT is for internet and the one created as Internal is for Domain Network.
 - [![image](http://lh3.ggpht.com/-IBk2lZ5w5LQ/UfIn9K0gITI/AAAAAAAASZk/61nTWJx8W5s/image_thumb%25255B2%25255D.png?imgmax=800 "image")](http://lh3.ggpht.com/-GPtAVIcBu5A/UfIn8T2HXWI/AAAAAAAASZc/9W5CfZy67kw/s1600-h/image%25255B6%25255D.png)
 - [![image](http://lh4.ggpht.com/-kTDule0Xa9w/UfIn-rCroKI/AAAAAAAASZ0/rJAX81tXx5c/image_thumb%25255B3%25255D.png?imgmax=800 "image")](http://lh5.ggpht.com/-hxZJAsuq9AY/UfIn9o1kIXI/AAAAAAAASZs/-IQ267SZpzQ/s1600-h/image%25255B9%25255D.png)
 - Enable Active Directory services role and reboot. Use dcpromo.exe to create a new domain. I chose my domain name as TESLA.COM( the greatest scientist and inventor ever). If you need help configuring AD Domain, check[this page](http://www.elmajdal.net/win2k8/setting_up_your_first_domain_controller_with_windows_server_2008.aspx).
 -  Post configuration and reboot, logon to the Domain Controller as `TESLA\Administrator` and create an active directory account and make it a Domain Admin, this will make things easier for you for the rest of setup.
 - Optionally, if you are planning on using this VM as Metastore for Hadoop cluster, you can install SQL Server 2008 R2. We will not go into the details of that setup. It is just a straight forward installation. Make sure you create a SQL Login which has sysadmin privileges. My login is`hadoop`**I have also created a couple of blank databases on it, namely `hivedb, ooziedb`
 
### Network Configuration and Joining other VMs to Domain
 - Log on to HDMaster as an Administrator
 - Find the Domain network Adapter (refer to step #3 in Configuring Domain section). Make sure your HDDC (Domain controller) VM is running.
 - Change the adapter settings: **IP - 192.168.80.2, subnet mast – 255.255.255.0, Default Gateway – Leave it blank, Preferred DNS – 192.168.80.1**
 - Right click on My Computer > Setting > Computer name, domain, workgroup settings > Settings and Change the computer name to `HDMaster` and enter the Domain name `TESLA.COM` Enter Domain Admin login and password when prompted and you’ll be greeted with a welcome message. Reboot.
 - Logon to each of the remaining VMs and repeat steps #1 to 4 – the only difference is the IP address. Refer to the list of IPs under ‘What will we build’ section of this article. Preferred DNS remains same for all the VMs i.e, `192.168.80.1`
 
### Preparing The environment for Hadoop Cluster

We now have a domain and all the VMs added to it. We also installed all the necessary software to the base VM so all the VMs created from it will inherit those. It is now time to prepare the environment for hadoop installation. Before we begin, make sure ALL VMs have same date, time settings.

**Disable IPV6 on ALL VMs**
Log on to each VM, open each network adapter and uncheck IPV6 protocol from adapter settings.

**Add firewall exceptions:**
Hadoop uses multiple ports for communicating between clients and other services so we need to create firewall exceptions on all the servers involved in the cluster. Execute the code below on**each VM** of the cluster on a powershell prompt launched as Administrator. If you want to make it simpler, you can just turn off Firewall at the domain level altogether on all VMs. But we don’t normally do that on production.
{% highlight powershell %}
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=50470 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=50070 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=8020-9000 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=50075 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=50475 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=50010 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=50020 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=50030 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=8021 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=50060 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=51111 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=10000 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=9083 
netsh advfirewall firewall add rule name=AllowRPCCommunication dir=in action=allow protocol=TCP localport=50111
{% endhighlight %}

#### Networking Configurations and Edit Group Policies:

 - Logon to your Domain Controller VM,**HDDC** with a Domain Administrator account (ex: TESLA\admin)
 - Go to start and type**Group Policy Management**in the search bar. Open Group Policy Management
 - If you don’t see your Forest TESLA.COM already, add a Forest and provide your domain name
 - Expand Forest –> Domain –> Name of your Domain –> right click on default domain policy –> edit
 - [![image](http://lh4.ggpht.com/-LKc1_vXRjlg/UfJJCZHsrHI/AAAAAAAASaM/_y9ZNHHdfRY/image_thumb%25255B4%25255D.png?imgmax=800 "image")](http://lh3.ggpht.com/-rieANp5wFHo/UfJJBs5m-vI/AAAAAAAASaE/RuPQ_9iiMQM/s1600-h/image%25255B12%25255D.png)
 - Group Policy Management Editor (lets call it**GPM Editor**) will open up and**note that ALL the steps following below are done through this editor**
 - In GPM Editor, navigate to `Computer Configuration -> Policies -> Windows Settings -> Security Settings ->System Services -> Windows Remote Management (WS-Management)`Change the startup mode to **Automatic**
 - Go to `Computer Configuration -> Policies -> Windows Settings -> Security Settings -> Windows Firewall with Advanced Security`. Right click on it and create a new**inbound**firewall rule. For Rule Type, choose**pre-defined,**and select Windows Remote Management from the dropdown. It will automatically create 2 rules. Let them be.  click**Next,**Check**Allow The Connection.**
 - This step is for enabling Remote Execution Policy for Powershell. Go to `Computer Configuration -> Policies -> Administrative Templates -> Windows Components -> Windows PowerShell`.**Enable** Turn on Script Execution. You’ll see**Execution Policy**dropdown under options, select **Allow All Scripts**
 - Go to `Computer Configuration -> Policies -> Administrative Templates -> Windows Components -> Windows Windows Remote Management (WinRM) -> WinRM Service`. Enable “**Allow Automatic Configuration of Listeners**” . Under Options, enter*** (asterisk symbol)**  for**IPV4 filter.**  Also change “**Allow CredSSP Authentication”**to**Enabled.**
 - We configured WinRM Service so far, now it’s WinRM Client’s turn.  Go to`Computer Configuration -> Policies -> Administrative Templates -> Windows Components -> Windows Windows Remote Management (WinRM) -> WinRM Client.` Set**Trusted Hosts**to**Enabled**and under Options, set**TrustedHostsList**to*** (asterisk symbol) .**  In the same dialog, look for**Allow CredSSP Authentication** and set it to**Enabled.**
 - Go to`Computer Configuration -> Policies -> Administrative Templates -> System -> Credentials Delegation` Set**Allow Delegating Fresh Credentials**  to**Enabled.**Under options, click on**Show**next to**Add Servers To The List**  and set WSMAN to*****  as shown below: (make a new entry like so:**WSMAN/*.TESLA.COM**, replace it with your domain here)
 - [![image](http://lh5.ggpht.com/-J04cpV__VNY/UfJMOlF82VI/AAAAAAAASak/ZGI_D3tfnXE/image_thumb%25255B5%25255D.png?imgmax=800 "image")](http://lh5.ggpht.com/-S-hjekrak5Q/UfJMNxaj-wI/AAAAAAAASac/Uf-FnGPCDSE/s1600-h/image%25255B15%25255D.png)
 - Repeat instructions above step, this time for property**NTLM-only server authentication**
 - Tired? We are almost done. Just a couple more steps before we launch the installation. Go to**Run**  and type**ADSIEdit.msc**and enter. Expand**OU=Domain Controllers** menu item and select**CN=HDDC (controller hostname).** Go to `Properties -> Security -> Advanced –> Add.` Enter**NETWORK SERVICE**, click**Check Names**, then Ok. In the Permission Entry select**Validated write to service principal name**. Click**Allow** and**OK** to save your changes.
 - On the Domain controller VM, launch a powershell windows an an administrator and run the following:**Restart-Service WinRM**
 - Force update gpupdate on the other VMs in the environment. Run this command: `gpupdate /force`
 
### Define Cluster Configuration File

When you download HortonWorks Dataplatform for windows, the zip contains a sample  `clusterproperties.txt` file which can be used as a template during installation. The installation msi depends on clusterproperties file for cluster layout definition. Below is my Cluster Configuration file. Make changes to yours accordingly.
{% highlight text %}
#Log directory
HDP_LOG_DIR=c:\hadoop\logs

#Data directory
HDP_DATA_DIR=c:\hdp\data

#Hosts
NAMENODE_HOST=HDMaster.tesla.com
SECONDARY_NAMENODE_HOST=HDMaster.tesla.com
JOBTRACKER_HOST=HDMaster.tesla.com
HIVE_SERVER_HOST=HDMaster.tesla.com
OOZIE_SERVER_HOST=HDMaster.tesla.com
TEMPLETON_HOST=HDMaster.tesla.com
SLAVE_HOSTS=HDSlave01.tesla.com, HDSlave02.tesla.com

#Database host
DB_FLAVOR=mssql
DB_HOSTNAME=HDDC.tesla.com

#Hive properties
HIVE_DB_NAME=hivedb
HIVE_DB_USERNAME=sa
HIVE_DB_PASSWORD=P@$$word!

#Oozie properties
OOZIE_DB_NAME=ooziedb
OOZIE_DB_USERNAME=sa
OOZIE_DB_PASSWORD=P@$$word!
{% endhighlight %}

### Installation

 -  Keep all of your VMs running and logon to**HDMaster,**with domain admin account.
 - Create a folder**C:\setup** and copy the contents of HortonWorks Data Platform installation media you downloaded at the beginning of this guide.
 - Replace the clusterproperties.txt with the one you created under**Define Cluster Config File**section. This file is same for ALL the participant servers in the cluster.
 - Open a command prompt as administrator and cd to `C:\setup`.
 - Type the below command to kick off installation. Make sure you dont leave white spaces spaces in the command. It took me a lot of time to figure out the space was making the installation fail. Edit the folder path according to your setup. There is no space between HDP_LAYOUT and ‘=’. `msiexec /i "c:\setup\hdp-1.1.0-GA.winpkg.msi" /lv "hdp.log" HDP_LAYOUT="C:\setup\clusterProperties.txt" HDP_DIR="C:\hdp\Hadoop" DESTROY_DATA="no"`
 - [![image](http://lh5.ggpht.com/-hcO_7BaEBHA/UfJT-Ow6VUI/AAAAAAAASa8/_BuKQbbANUc/image_thumb%25255B6%25255D.png?imgmax=800 "image")](http://lh4.ggpht.com/-2oYxpeCaFkc/UfJT9MU53VI/AAAAAAAASa0/H6PX_CVMiY4/s1600-h/image%25255B18%25255D.png)
 - Log on to each of the other VMs, and**repeat installation steps**.
 - **Congratulations**, you have now successfully created your first Hadoop Cluster. You’ll see shortcuts like below on your desktop on each VM. Before starting Hadoop exploration, we need to start services on both Local and remote machines. Log on to the HDMaster server, go to hadoop installation directory and look for \hadoop folder. You’ll see a couple of batch files to start/stop services.
 - [![image](http://lh5.ggpht.com/-wZN2HD6Vj2E/UfJW1mTsWdI/AAAAAAAASbU/k1ZiWJtoZds/image_thumb%25255B7%25255D.png?imgmax=800 "image")](http://lh5.ggpht.com/-YDUp1kZ4S5s/UfJW02yfkvI/AAAAAAAASbM/fq2fwK6yqLw/s1600-h/image%25255B21%25255D.png)
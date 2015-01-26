---
title: Provisioning a Windows Server Vagrant box with IIS, .NET 4.5 and Octopus Deploy
date: 2015-01-21 10:00
type: blog
layout: post
tags: ['.NET','Automation', 'How To', 'IIS', 'Octopus Deploy', 'Vagrant', 'Windows Server']
---

As part of testing for our new Continuous Integration set-up, I needed to pull together a bunch of machines for testing deployments. I had planned on using [Octopus Deploy](https://octopusdeploy.com/) to do this so [Vagrant](https://www.vagrantup.com/) seemed like the best choice for getting new machines set up.

## Provisioning
The entry point for creating a Vagrant box is through the [Vagrantfile](https://docs.vagrantup.com/v2/vagrantfile/). Once you have Vagrant installed, you can create a new file at the command line using the command:

<pre class="brush: bash">
vagrant init
</pre>

As well as giving instructions for setting up the box, the Vagrant file also allows [provisioning](https://docs.vagrantup.com/v2/getting-started/provisioning.html) using various scripts.

We'll set up three boxes for testing - Dev (for local, ongoing development), FAT (Factory Acceptance Testing) and SAT (Site Acceptance Testing).

Our goal for each box will be:

- Create a new Windows Server machine with a valid name
- Give it a local network IP
- Set up IIS
- Delete the default IIS website
- Install .NET 4.5.2
- Install Octopus Tentacle for deployment
- Configure Octopus Tentacle for use with our installation

## Base Box
There are a lot of [Vagrant base boxes available online](http://www.vagrantbox.es/). For our mini network, we'll be using [ferventcoder/win2008r2-x64-nocm](https://vagrantcloud.com/ferventcoder/boxes/win2008r2-x64-nocm) from [Atlas](https://vagrantcloud.com).

This 64-bit Windows 2008 installation will suit just fine for running our .NET apps.

## The Vagrant File
First we need to set up our box. We'll be using [VirtualBox](https://www.virtualbox.org/) since it's multi-platform and supports hardware virtualisation (that we'll need).

Here's our basic file:

<pre class="brush: ruby">
Vagrant.configure(2) do |config|

  config.vm.box = "ferventcoder/win2008r2-x64-nocm"
  config.vm.guest = :windows
  config.vm.communicator = "winrm"

  config.vm.boot_timeout = 600

  config.vm.define "dev" do |dev|
    dev.vm.network "private_network", ip: "192.168.100.10"
    dev.vm.host_name = "vagranttests.dev"
    dev.vm.network :forwarded_port, guest: 5985, host: 5985, id: "winrm", auto_correct: true
  end

  config.vm.define "fat" do |fat|
    fat.vm.network "private_network", ip: "192.168.100.11"
    fat.vm.host_name = "vagranttests.fat"
    fat.vm.network :forwarded_port, guest: 5985, host: 5986, id: "winrm", auto_correct: true
  end

  config.vm.define "sat" do |sat|
    sat.vm.network "private_network", ip: "192.168.100.12"
    sat.vm.host_name = "vagranttests.sat"
    sat.vm.network :forwarded_port, guest: 5985, host: 5987, id: "winrm", auto_correct: true
  end

  config.vm.provider "virtualbox" do |vb|
    # Display the VirtualBox GUI when booting the machine
    # vb.gui = true
    
    # Customize the amount of memory on the VM:
    vb.cpus = 2
    vb.memory = 2048
  end

end
</pre>

There are a few things in here to cover.

<pre class="brush: ruby; first-line: 3">
  config.vm.box = "ferventcoder/win2008r2-x64-nocm"
  config.vm.guest = :windows
  config.vm.communicator = "winrm"
</pre>

Lines 3-5 specify our base box, tell Vagrant that the guest machine will be Windows, and that we want to use [winrm](https://docs.vagrantup.com/v2/vagrantfile/winrm_settings.html) for communication with the box rather than SSH.

<pre class="brush: ruby; first-line: 7">
  config.vm.boot_timeout = 600
</pre>

Line 7 bumps up the default timeout for the box to respond. You might not have issues with this but I found that Windows Server was taking a while to boot on some occasions.

The next few sections set up each machine. They're pretty similar so we'll just take a look at the Dev one.

<pre class="brush: ruby; first-line: 9">
  config.vm.define "dev" do |dev|
    dev.vm.network "private_network", ip: "192.168.100.10"
    dev.vm.host_name = "vagranttest.dev"
    dev.vm.network :forwarded_port, guest: 5985, host: 5985, id: "winrm", auto_correct: true
  end
</pre>

Line 9 uses the [multi-machine syntax](https://docs.vagrantup.com/v2/multi-machine/) to specify that this setting is just for a machine we'll refer to as **dev**.

Line 10 sets up a local private network and assigns the IP *192.168.100.10*. We'll be able to access the guest box from our OS on this IP.

Line 11 sets the name for the machine - **vagranttest.dev**.

Line 12 sets up a forwarding port to pass *winrm* through to the box.

The last section of the Vagrant file sets up some settings specific to [VirtualBox](https://www.virtualbox.org/).

<pre class="brush: ruby; first-line: 27">
  config.vm.provider "virtualbox" do |vb|
    # Display the VirtualBox GUI when booting the machine
    # vb.gui = true
    
    # Customize the amount of memory on the VM:
    vb.cpus = 2
    vb.memory = 2048
  end
</pre>

Line 29 toggles the VirtualBox GUI being launched along with a machine. This is generally not needed so we'll comment it out but keep it in the file so it's easy to enable if we need to in future.

Lines 32 and 33 specify that we want 2 CPUs and 2Gb memory per machine.

And that's all we need to spin up a default box. Next we need to write scripts to provision it.

## Provisioning
Here's a reminder of our goals and how we've progressed so far:

- <s>Create a new Windows Server machine with a valid name</s>
- <s>Give it a local network IP</s>
- Set up IIS
- Delete the default IIS website
- Install .NET 4.5
- Install Octopus Tentacle for deployment
- Configure Octopus Tentacle for use with our installation

We'll write scripts for each of these actions and save them in a *./scripts* sub-folder to our Vagrant file.

### Set up IIS
On Windows Server, we can install IIS using the Web Server Role. We'll also want to add in some output so we can see what's happening from the Vagrant command line.

So the CMD file for this would be:

#### install-iis.cmd
<pre class="brush: bash">
@echo off

echo "Installing IIS 7.5; it will take a while..."
CMD /C START /w PKGMGR.EXE /l:log.etw /iu:IIS-WebServerRole
echo "Done installing IIS."
</pre>

### Delete the default IIS website
We can use the WebAdministration PowerShell module to control the local IIS set-up.

#### delete-default-iis-website.ps1
<pre class="brush: bash">
$ErrorActionPreference = "Stop"
echo "Deleting the default IIS website..."

import-module webadministration
Stop-WebSite 'Default Web Site'
Remove-Website 'Default Web Site'

echo "Default website deleted."
</pre>

Line 1 here is telling the script to [stop if any error occurs](http://blogs.technet.com/b/heyscriptingguy/archive/2010/03/08/hey-scripting-guy-march-8-2010.aspx). The rest is self-explanatory.

### Install .NET
We'll install .NET in two parts. Firstly, we'll install the [.NET Windows Feature](http://blogs.msdn.com/b/sqlblog/archive/2010/01/08/how-to-install-net-framework-3-5-sp1-on-windows-server-2008-r2-environments.aspx) and then we'll run the [offline installer for .NET 4.5.2](http://www.microsoft.com/en-gb/download/details.aspx?id=42642).

Installing the features is done using the [Server Manager PowerShell module](http://technet.microsoft.com/en-us/magazine/ff476071.aspx) like so:

#### install-dot-net.ps1
<pre class="brush: bash">
$ErrorActionPreference = "Stop"

import-module servermanager
echo "Enabling .NET Framework"
add-windowsfeature as-net-framework
</pre>

To run the offline 4.5.2 installer, we'll need to get a copy of the file.

We could write a script to grab this from the internet once a machine has loaded but this would require every machine to have internet access (which we may or may not want) and will mean that the file is downloaded once per machine provision. This is not a big problem with our cluster of 3 machines, but could slow things down dramatically if we have dozens of machines or want to run the provisioning many times.

Vagrant supports [synced folders](https://docs.vagrantup.com/v2/synced-folders/basic_usage.html) on each machine, syncing our local directory containing the Vagrant file with (on Windows) **C:\Vagrant**. So, we can download a local copy of the installer and then launch that from within the machine. This has the added benefit of ensuring that every machine runs with the same resources.

So we can grab the [offline installer executable from Microsoft](http://download.microsoft.com/download/E/2/1/E21644B5-2DF2-47C2-91BD-63C560427900/NDP452-KB2901907-x86-x64-AllOS-ENU.exe) and put it into a **./resources** folder.

Then we just need a PowerShell script to quietly install this from our synced folder:

#### install-dot-net-45.cmd
<pre class="brush: bash">
@echo off

echo "Installing .NET Framework 4.5.2"
C:\vagrant\resources\NDP452-KB2901907-x86-x64-AllOS-ENU.exe /q
echo "Done!"
</pre>

### Install Octopus Tentacle for deployment
Octupus Tentacle comes as an MSI so we can install this in much the same way at the offline .NET installer.

We'll grab the installer from the [Octopus Deploy downloads page](http://octopusdeploy.com/downloads) and put it into our **./resources** folder.

Octopus have [very good docs on automating installation](http://docs.octopusdeploy.com/display/OD/Automating+Tentacle+installation) so have a read through those if you want detailed information on this or need to customise it. For our purposes, we'll just be installing with the defaults:

#### install-octopus-tentacle.cmd
<pre class="brush: bash">
@echo off

echo "Installing Octopus Tentacle..."
msiexec /i C:\vagrant\resources\Octopus.Tentacle.2.6.0.778-x64.msi /quiet
echo "Done!"
</pre>

### Configure Octopus Tentacle for use with our installation
From the same [good docs on automating installation](http://docs.octopusdeploy.com/display/OD/Automating+Tentacle+installation), Octopus have guides on setting up a new Tentacle instance. The easist way to get this information is to run locally using the GUI then choose **Show script** at the last stage.

We'll use a script to set up app locations for our instance, set ports, specify the trust key of our Octopus server and add a firewall rule to allow access:

#### configure-octopus-tentacle.cmd
<pre class="brush: bash; highlight: [9]">
echo "Configuring Octopus tentacle instance..."
cd "C:\Program Files\Octopus Deploy\Tentacle"
 
Tentacle.exe create-instance --instance "Tentacle" --config "C:\Octopus\Tentacle\Tentacle.config" --console
Tentacle.exe new-certificate --instance "Tentacle" --if-blank --console
Tentacle.exe new-squid --instance "Tentacle" --console
Tentacle.exe configure --instance "Tentacle" --reset-trust --console
Tentacle.exe configure --instance "Tentacle" --home "C:\Octopus" --app "C:\Octopus\Applications" --port "10933" --console
Tentacle.exe configure --instance "Tentacle" --trust "YOUROCTOPUSKEYGOESHERE" --console

"netsh" advfirewall firewall add rule "name=Octopus Deploy Tentacle" dir=in action=allow protocol=TCP localport=10933
Tentacle.exe service --instance "Tentacle" --install --start --console

echo "Done!"
</pre>

Remember to add your key from your Octopus Server in line 9.

And that's our completed scripts. After creating all of these, your file structure should look something like this:

<pre>
- /resources
   - NDP452-KB2901907-x86-x64-AllOS-ENU.exe
   - Octopus.Tentacle.2.6.0.778-x64.msi
- /scripts
   - configure-octopus-tentacle.cmd
   - delete-default-iis-website.ps1
   - install-dot-net.ps1
   - install-dot-net-45.cmd
   - install-iis.cmd
   - install-octopus-tentacle.cmd
- Vagrantfile
</pre>

We can now update our Vagrant file to run these scripts.

## Running Provisioning
Firstly we'll add in some checks for our resources so that the script will stop until these are available.

<pre class="brush: ruby; first-line: 4">
if ! File.exists?('./resources/NDP452-KB2901907-x86-x64-AllOS-ENU.exe')
  puts '.Net 4.5.2 installer could not be found!'
  puts "Please run:\n  wget http://download.microsoft.com/download/E/2/1/E21644B5-2DF2-47C2-91BD-63C560427900/NDP452-KB2901907-x86-x64-AllOS-ENU.exe"
  exit 1
end

if ! File.exists?('./resources/Octopus.Tentacle.2.6.0.778-x64.msi')
  puts 'Octopus Tentacle installer could not be found!'
  puts "Please run:\n  wget http://download.octopusdeploy.com/octopus/Octopus.Tentacle.2.6.0.778-x64.msi"
  exit 1
end
</pre>

Doing this at the start means that feedback on the missing files will be immediate to anyone using the Vagrant file. If we put these checks during provisioning, there would be no warning until the boxes had already been created.

If those checks pass, we can run the provisioning scripts.

As we'll be running the same provisioning on each machine, this can be done outside of the box config blocks.

Adding these in under our existing generic configs now gives us this:

<pre class="brush: ruby; first-line: 18; highlight: [22,23,24,25,26,27]">
  config.vm.box = "ferventcoder/win2008r2-x64-nocm"
  config.vm.guest = :windows
  config.vm.communicator = "winrm"

  config.vm.provision :shell, path: "scripts/install-iis.cmd"
  config.vm.provision :shell, path: "scripts/delete-default-iis-website.ps1"
  config.vm.provision :shell, path: "scripts/install-dot-net.ps1"
  config.vm.provision :shell, path: "scripts/install-dot-net-45.cmd"
  config.vm.provision :shell, path: "scripts/install-octopus-tentacle.cmd"
  config.vm.provision :shell, path: "scripts/configure-octopus-tentacle.cmd"

  config.vm.boot_timeout = 600
</pre>

And that's it.

## The Final Vagrant File
Here's what our final file will look like:

<pre class="brush: ruby">
# -*- mode: ruby -*-
# vi: set ft=ruby :

if ! File.exists?('./resources/NDP452-KB2901907-x86-x64-AllOS-ENU.exe')
  puts '.Net 4.5.2 installer could not be found!'
  puts "Please run:\n  wget http://download.microsoft.com/download/E/2/1/E21644B5-2DF2-47C2-91BD-63C560427900/NDP452-KB2901907-x86-x64-AllOS-ENU.exe"
  exit 1
end

if ! File.exists?('./resources/Octopus.Tentacle.2.6.0.778-x64.msi')
  puts 'Octopus Tentacle installer could not be found!'
  puts "Please run:\n  wget http://download.octopusdeploy.com/octopus/Octopus.Tentacle.2.6.0.778-x64.msi"
  exit 1
end

Vagrant.configure(2) do |config|

  config.vm.box = "ferventcoder/win2008r2-x64-nocm"
  config.vm.guest = :windows
  config.vm.communicator = "winrm"

  config.vm.provision :shell, path: "scripts/install-iis.cmd"
  config.vm.provision :shell, path: "scripts/delete-default-iis-website.ps1"
  config.vm.provision :shell, path: "scripts/install-dot-net.ps1"
  config.vm.provision :shell, path: "scripts/install-dot-net-45.cmd"
  config.vm.provision :shell, path: "scripts/install-octopus-tentacle.cmd"
  config.vm.provision :shell, path: "scripts/configure-octopus-tentacle.cmd"

  config.vm.boot_timeout = 600

  config.vm.define "dev" do |dev|
    dev.vm.network "private_network", ip: "192.168.100.10"
    dev.vm.host_name = "vagranttests.dev"
    dev.vm.network :forwarded_port, guest: 5985, host: 5985, id: "winrm", auto_correct: true
  end

  config.vm.define "fat" do |fat|
    fat.vm.network "private_network", ip: "192.168.100.11"
    fat.vm.host_name = "vagranttests.fat"
    fat.vm.network :forwarded_port, guest: 5985, host: 5986, id: "winrm", auto_correct: true
  end

  config.vm.define "sat" do |sat|
    sat.vm.network "private_network", ip: "192.168.100.12"
    sat.vm.host_name = "vagranttests.sat"
    sat.vm.network :forwarded_port, guest: 5985, host: 5987, id: "winrm", auto_correct: true
  end

  config.vm.provider "virtualbox" do |vb|
    # Display the VirtualBox GUI when booting the machine
    # vb.gui = true
    
    # Customize the amount of memory on the VM:
    vb.cpus = 2
    vb.memory = 2048
  end

end
</pre>

## Running the File
With your Vagrant file, you can now spin up a specific machine with:

<pre class="brush: bash">
vagrant up dev
</pre>

or spin up all with

<pre class="brush: bash">
vagrant up
</pre>

Here's a screen capture of the dev box spinning up. It takes a fair amount of time to run, so it's sped up quite a bit.

<img src="http://kwilson.me.uk/blog/wp-content/uploads/2015/01/vagrant-fast-optimised.gif" alt="" width="614" height="430" class="aligncenter size-full wp-image-2271" />

And there you have it. Easy provisioning of new Windows 2008, .NET compatible, Octopus Tentacle enabled boxes.

Download the source file for this at [github.com/kwilson/vagrant-octopus](https://github.com/kwilson/vagrant-octopus).
Hyper-V
======

Overview
--------

This section will cover creating a virtual machine (VM) for Security Onion 16.04 in Hyper-V.

Hyper-V can be installed on Windows Server > 2008, or Windows 10.

Things of notice
------------
No not enable dynamic memory.


Creating VM
------------

Follow the wizard in Hyper-V Manager, or use the following commands if creating the VM via PowerShell:

Create the virtual machine:
::

    New-VM -Name security-onion -MemoryStartupBytes 16GB -NewVHDPath C:\My-New-Drive.vhdx -NewVHDSizeBytes 100GB -BootDevice CD -Generation 2
Add the ISO to boot:
::
    Add-VMDvdDrive -VMName security-onion -Path C:\ISO\securityonion.iso
Set the desired number of CPU cores:
::
    Set-VMProcessor security-onion -Count 8

Creating Virtual Switches
------------
Security Onion requires both a management interface, as well as an interface for the traffic you want to monitor. To accomplish this, we need to allow the virtual switch to pass though the traffic we want to monitor to our VM, which requires some configuration.
Identify the physical network adapters you are using for management and sniffing traffic:
::
    Get-NetAdapter

Create a new virtual switch for your management interface. If you have an exsisting switch that you want to use for management traffic, you can skip this step:
::
    New-VMSwitch -name ManagementSwitch  -NetAdapterName "Ethernet" -AllowManagementOS $true  
    
Create a new virtual switch for your sniffing traffic:
::
    New-VMSwitch -name SniffingSwitch  -NetAdapterName "Ethernet 2" -Passthru

Allow external traffic to be passed through sniffing switch. This is effectivly setting the network adapter in promiscous mode:
::
    $portFeature=Get-VMSystemSwitchExtensionPortFeature -FeatureName "Ethernet Switch Port Security Settings"
    # None = 0, Destination = 1, Source = 2
    $portFeature.SettingData.MonitorMode = 2
    Add-VMSwitchExtensionPortFeature -ExternalPort -SwitchName SniffingSwitch -VMSwitchExtensionFeature $portFeature
    
Add network adapters to VM
--------------------------
Add management network adapter:
::
    Add-VMNetworkAdapter -VMName security-onion -SwitchName ManagementSwitch
Add sniffing network adapter:
::
    $sniffingAdapter = Add-VMNetworkAdapter -VMName security-onion -SwitchName SniffingSwitch
    Set-VMNetworkAdapter $sniffingAdapter -PortMirroring Destination
(Optional) Allow vlan traffic:
::
    Set-VMNetworkAdapterVlan $sniffingAdapter -Trunk -AllowedVlanIdList "100,101" -NativeVlanId 0
    

Sniffing
----------------------

-  With the sniffing interface in "bridged" mode, you will be able to
   see all traffic to/from the host machine's physical NIC. If you would
   like to see **ALL** the traffic on your network, you will need a
   method of forwarding that traffic to the interface to which the
   virtual adapter is bridged. This can be achieved by switch port
   mirroring (SPAN), or through the use of a
   `tap <Hardware#enterprise-tap-solutions>`__.

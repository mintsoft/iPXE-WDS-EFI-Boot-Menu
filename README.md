# A pre-boot menu for EFI that allows WDS by default and selection of others manually #

## Delivered entirely from WDS ##

1. Create a folder under `REMINST\Boot` for iPXE like: `REMINST\Boot\iPXE`
2. Download `snmponly.efi` for iPXE from http://boot.ipxe.org/snponly.efi and store as `REMINST\Boot\iPXE\snponly.efi`
3. It's critically important that in `WDS -> Server Properties -> Boot -> Unknown Clients` is set to `Always continue the PXE boot`

## Windows DHCP Configuration ##

1. Create User Class for iPXE (right click ipv4; click define user classes and name it iPXE and add `iPXE` as the ASCII value
2. Create Vendor Class for EFI x64 (right click ipv4; click Vendor classes, name it `PXEClient:Arch:00007` and add `PXEClient:Arch:00007` as the ASCII value
3. Add policy to deliver `snponly.efi` as the boot filename for EFI clients :: Either at the Scope or IPv4 level; right-click policies, click add, Call it "Deliver iPXE", add a Vendor Class filter to `PXEClient:Arch:00007` with appended wildcard and set Boofile name to `Boot\iPXE\snponly.efi`
4. Add policy to deliver the iPXE configuration when iPXE requests it :: At the same level as above, add a new Policy called "iPXE Configuration", add the condition the Vendor Class is `PXEClient:Arch:00007` with appended wildcard, change the radio to `AND` and User Class is `iPXE`. Set Bootfile name to the path to the iPXE configuration `Boot\iPXE\iPXE.conf`.
5. Change the Policy Order so that "iPXE Configuration" is Processing Order 1 and Deliver iPXE is Processing Order 2
6. Remove the PXEClient Option 60 from being set by any of the DHCP server settings (either Server Options or Scope Options) as this causes WDS to hijack the DHCP request

## Configure iPXE menu ##
1. Create the `REMIST\Boot\iPXE\iPXE.conf` file  with the following content:
```
#!ipxe
chain --autofree Boot\iPXE\boot.ipxe.cfg
```
2. Create the actual boot menu file `REMIST\Boot\iPXE\boot.ipxe.cfg` with the following content:
```
#!ipxe

# Figure out if client is 64-bit capable
cpuid --ext 29 && set arch x64 || set arch x86
cpuid --ext 29 && set archl amd64 || set archl i386

###################### MAIN MENU ####################################

:start
menu iPXE boot menu
item --gap --             -------------------------------Installation ------------------------------
item --key w wds          WDS
item --key l linux        Linux
item --gap --             ------------------------- Advanced options -------------------------------
item --key c config       Configure settings
item shell                Drop to iPXE shell
item reboot               Reboot computer
item
item --key x exit         Exit iPXE and continue BIOS boot
choose --timeout 5000 --default wds selected || goto cancel
set menu-timeout 0
goto ${selected}

:cancel
echo You cancelled the menu, dropping you to a shell

:shell
echo Type 'exit' to get the back to the menu
shell
set menu-timeout 0
set submenu-timeout 0
goto start

:failed
echo Booting failed, dropping to shell
goto shell

:reboot
reboot

:exit
exit

:config
config
goto start

:back
set submenu-timeout 0
clear submenu-default
goto start

############ MAIN MENU ITEMS ############

:wds
chain Boot\x64\wdsmgfw.efi || goto failed
goto start

:linux
kernel Boot\iPXE\debian-installer\linux initrd=Boot\iPXE\debian-installer\initrd.gz auto=true url=http://path/to/preseed/file.cfg
initrd Boot\iPXE\debian-installer\initrd.gz
boot
goto start
```

This is assuming that you have downloaded the Debian netboot.tar.gz (i.e. http://ftp.nl.debian.org/debian/dists/stretch/main/installer-amd64/current/images/netboot/netboot.tar.gz) and extracted the content to `REMINST\Boot\iPXE\debian-installer`. 

## Optional: Alter WDS TFTP to support both `\` and `/` ##
This will be required if you want to boot grub2 or pxelinux; ipxe doesn't require it however :D

1. Make WDS TFTP server accept both `/` and `\` 
2. Alter: `HKEY_LOCAL_MACHINE/SYSTEM/CurrentControlSet/Services/WDSServer/Providers/WDSTFTP/ReadFilter` and make sure that following are included:
```
boot/*
boot\*
/boot/*
\boot\*
/boot\*
```

3. Restart the `Windows Deployment Services` Service

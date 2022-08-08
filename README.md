# fedora-pxe-setup
Setup repository for a Fedora PXE server.

Loosely based on [Fedora PXE Setup](https://docs.fedoraproject.org/en-US/fedora/latest/install-guide/advanced/Network_based_Installations/#pxe-overview).

Files: 
- wallpapers (folder) - contains image files for desktop wallpaper. these can be any filetype supported by gnome for backgrounds. most that I curated are png's.
- .gitattributes - git attributes for the large file (settings.tar.gz). any files over 99MB will be handled by git lfs.
- adduser.local - *No Longer Used* - a script run after user creation on Virtual hosts that adds the newly created users to the VNC server. this should not be considered 'secure' but is a way to have centrally located user accounts on a virtual server. recommend reviewing the global VNC defaults at /etc/tigervnc/vncserver-settings?-default/-required to make it more secure.
- adduser.local.pp - *No Longer Used* - SELinux policy to allow adduser.local to run.
- basic-workstation.ks - a basic workstation kickstart file. this uses the entire hard disk, installs the base workstation package group and encrypts the hard drive. **you should change the disk encryption password here**, but **DO NOT git push back into the repo after**. 
- default - basic BIOS menu configuration; no longer used, in favor of default-automenu
- default-automenu - BIOS menu configuration - automatically generated from the kickstarts available on the server.
- dhcpd.conf - dhcpd configuration - automatically generated from ansible facts; uses the subnet it is connected on to serve dhcp, next server, and dns information to clients. **You need to set up networking via nmtui** (from package NetworkManager-tui) for this to work right, because it pulls DNS from the subsequent nmconnection file, and you should be using a manual IP address.
- grub-automenu.cfg - GRUB menu configuration - automatically generated from the kickstarts available on the server.
- grub.cfg - basic GRUB menu configuration; no longer used, in favor of grub-automenu.cfg
- LICENSE - license for the project, GPL 3.0
- manual-install.ks - manual installation kickstart; can be used to manually install a custom selection of software; meant to be used in testing to generate new automatic kickstarts if needed.
- networkminer.png - the software NetworkMiner comes with an icon that is glitched. this file was edited by myself from a random networkminer image and placed here to replace the icon.
- PXE Server SOP.docx - Standard Operating Procedure for various PXE Server tasks and administration.
- pxe-setup.yml - ansible playbook that sets up a base PXE server using the files in this directory. See pxe-setup flow for more info.
- README.md - this file.
- settings.tar.gz - contains local settings for firefox, chrome, vscode, local password policy, and .bashrc's for root and all users. this file will likely be removed from this repository and distributed ad-hoc due to the potentially sensitive nature of files in here.
- virtual-post.yml - ansible playbook that sets up a base Virtual Machine fedora workstation. be default some things are omitted from here, like vmware horizon; our use case requires smart cards, and we can't broker smartcard over VNC so it is not needed; also removed virtualization capability i.e. gnome-boxes and also added a VNC server (see adduser.local above). **this can probably still be improved**.
- virtual-workstation.ks - the accompanying kickstart file to automatically install with the virtual-post.yml configuration.
- vncuseradd - bash script placed in /usr/loca/bin on virtual workstations. Can be used to create new users with VNC capabilities.
- workstation-post.yml - the basic workstation ansible playbook that installs a standard loadout of software.

# flow of pxe-setup.yml

**this is a high-level overview and is not an exhaustive list of what happens; see the yml for more info**. the ansible script will download needed components for the server (httpd, dhcp-server, tftp-server) and then syslinux and grub-x64. It copies the relavant files from the latter two packages into /var/lib/tftpboot, the pxe server directory. it then downloads the kernel and initrd. afterward it writes a new repo file for the Everything branch of the fedora mirror, which will allow installations local not requiring internet. it writes the template config files to the relevant locations and then opens the firewall to allow the services to be accessible, and starts/restarts and enables the services. at the end it then syncs the repo. **this takes a long time, it is about 85 GB**.

IF any files are changed on the pxe server, i.e. the basic-workstation.ks file to change the LUKS encryption key, the pxe-setup.yml should be run again with `sudo ansible-playbook pxe-setup.yml` within this directory. **the local repo sync will take much less time, as it will only verify vice re-download**.

# HOWTO: Documentation
the following items need to be within the documentation:
- howto - setup from-scratch pxe server
- howto - setup from-VM-clone pxe server
- howto - upgrade Fedora version
- howto - edit files within this directory to customize the installation
- howto - build settings.tar.gz for ad-hoc distribution
- howto - add new software
- howto - create VNC virtual machine

## howto: Setup from-scratch PXE Server
1. create VM / Prep Server for Install
    - at least 4 CPUs
    - 4096 MB / 4 GB RAM
    - 200-250 GB Storage
    - for VM, Bridged Network Adapter, or in ESXi, connected to the subnet needed to image from.
2. Get ISO
    - Use latest Fedora Linux Server DVD: https://download.fedoraproject.org/pub/fedora/linux/releases/ (and then navigate to version > Server > x86_64 > iso and download the dvd iso)
        - this might change into a different hyperlink because it is a metalink to point to the closest mirror. the original is download.fedoraproject.org/pub/fedora/linux/releases
    - For physical servers, burn to CD or write to USB as needed
    - For a VM, load the ISO into the datastore, or in Player/Workstation/virtualbox set the CD drive to the iso file.
3. Boot to ISO
    - interrupt boot as necessary to boot from CD/USB
4. Installation
    - need keyboard, video and mouse to install.
    - Select language
    - Set the timezone
    - Click Network, change the hostname to pxe and click Apply. Make sure network is connected.
    - Installation destination: select Custom for partitioning, then click Done; after:
        - if there are other partitions, click the arrow next to them, click one, then click the Minus (-) button; when prompted, check the box to delete other partitions on this installation and then click to confirm deletion.
        - In the dropdown, select Standard Partition
        - Click to create standard Fedora mountpoints
        - If a /home partition is created, click it and click Minus (-) to remove it
        - change the root (/) partition to be the max size; click it and change it to 200/250 GB and click Apply Changes. it will auto-adjust.
        - Click Done, then click Accept Changes
    - Create user; make sure they are an admin. (you may have to scroll down to see the user creation)
    - Software Selection: Choose Minimal Install if available, or choose Fedora Custom OS.
    - Click to begin the install, and reboot when done.
5. First Boot
    - `sudo dnf update`
    - reboot if new kernel installed (it probably was)
    - `sudo dnf install ansible git NetworkManager-tui python3-netaddr`
    - `git clone https://github.com/sjtrotter/fedora-pxe-setup.git`
    - Use nmtui to set network information manually. Make sure you set the IP with a /XX for the CIDR, and make sure you set the DNS and Gateway appropriately. ( run `nmtui` )
    - If using an ad-hoc version of settings.tar.gz, place it in the fedora-pxe-setup directory.
    - `sudo ansible-playbook fedora-pxe-setup/pxe-setup.yml` (make sure you use the path into the git clone, if you have cd'd elsewhere)
6. Done.
    Once the ansible playbook completes successfully, the server is ready to PXE boot devices.

## howto: Setup from-VM-clone PXE server
This assumes you have previously set up a PXE server as a VM according to above and you wish to clone it.
1. Clone/move as necessary.
2. Boot from console:
    - Login, then use nmtui ( run `nmtui` ) to set network information manually.
    - `cd fedora-pxe-setup` and then `git pull` to update files
    - if using an ad-hoc version of settings.tar.gz, place it in the fedora-pxe-setup directory.
    - `sudo ansible-playbook pxe-setup.yml` to update and place all files.
3. Done.
    Once playbook completes successfully, the server is ready to PXE boot devices.

## howto: Upgrade Fedora version
Fedora upgrades about every 6 months, in April-ish and October-ish. When ready to test the next version, change the `version: ##` line at the top of pxe-setup.yml (line 5) to the appropriate number.

One potential breakage this may cause is the CERT Forensic Tools packages. The administrator of the PXE server should ensure that the repository at https://forensics.cert.org/fedora/cert/ is available for the new Fedora version before attempting upgrade (make sure there is a folder for the new version). They should also check to ensure a new key is not needed, by reviewing documentation at https://forensics.cert.org/#fedorasupport

Setting this number 2 versions higher than the original installation of the PXE server will remove the 2nd-to-last installation directory before cloning the repository. this should keep the total size of the server under 200GB.

## howto: Customize the installation
- task: change hard drive encryption password
    - file: basic-workstation.ks - within this file, the line `autopart --encrypted --passphrase workstation` (line 30) should have the last item, workstation, changed to whatever you want the encryption password to be. This should be in quotes if you use any special characters. The virtual-workstation.ks file does not encrypt by default because usually you do not want to encrypt a virtual machine, because you'd have to access the console to unlock it.
- task: add default user
    - file: basic-workstation.ks, virtual-workstation.ks - if you decide you want a default user to be set, uncomment the line `#user --groups=wheel --name=user --password=workstation --gecos="user"` (around line 40) (remove the '#') and then edit the --name= and --gecos= and --password= values to setup the user. By default, users are not set; this allows (forces?) the end-user of the laptop to create their own user account. Remember to quote the password if any special characters are used.
- task: change VNC password
    - file: vncuseradd - within this file, edit the line `sudo su $user -c 'printf "password\npassword\n" | vncpasswd 2>&1>/dev/null' >/dev/null` (line 109-ish) and replace the default password with the desired password, in both places.

Once you change any files, **DO NOT** git push back into the repository, if you have set up to push.

If you change any files, you need to run/re-run `sudo ansible-playbook pxe-setup.yml` to apply the changes.

## howto: Customize settings.tar.gz
The default settings.tar.gz contains files that are meant to be uncompressed in the root (/) directory. It contains default settings for Firefox, Chrome, and .bashrc and /root/.bashrc, and also contains a more robust password policy. You may want to include things like default settings for Horizon or for Autopsy to connect to a multi-user database. If so, you will need to add it to the settings tarball to have it take effect.

In order to update the settings, first run an installation on a machine. it can be real or virtual, we just need a baseline. After install, open a terminal and do the following:
- `cd /`
- `sudo wget [ PXE SERVER IP ]/f[ fedora version ]-inst.local/other/settings.tar.gz`
- `sudo gunzip settings.tar.gz`

Once you have the tarball unzipped, you need to identify the files you want to standardize. For example, if you want the Autopsy configuration, you should open Autopsy then edit the preferences as you need, then look in the user home folder and find the settings. (for autopsy, it is in /home/USER/.autopsy). You should then move them to /etc/skel like so:
- `sudo cp -r /home/USER/.autopsy /etc/skel`

and then you should add the folder to the settings tarball like so:
- `sudo tar -rf settings.tar /etc/skel/.autopsy`

When done adding the settings you want, re-zip the files:
- `sudo gzip settings.tar`

And then, scp the files to the PXE server. (make sure SSH is on, on the pxe server and the laptop, with `sudo systemctl start sshd`)
- `scp settings.tar.gz [pxe user]@[pxe ip]:/path/to/fedora-pxe-setup/settings.tar.gz`

## howto: adding new software to the base image
At times new software may be requested to be added to the base image. in order to do this, testing must be done:
- first, have a baseline image laptop to test on
- search the repositories to see if the software is already available. you can use a terminal and type in `dnf search [ software ]` to see if anything matches.
    - if anything does, you can add it to the baseline by editing workstation-post.yml and/or virtual-post.yml around line 157/160, starting with `- { name: mono-devel,... }` where you can add it to the comma-separated list of software to install.
- if it is not available in the repositories, the laptop should be used to test how the software can be installed, and the process recorded; it should then be automated with ansible and added to the workstation-post.yml and/or virtual-post.yml as appropriate.
- once install is verified, default settings should be vetted for the application, if needed, and added to settings.tar.gz  as detailed above.

## howto: create VNC virtual machine
For single-user virtual machines, modest RAM and storage can be used (something like 2 CPUs, 4G RAM, 50GB storage).

If you are making a main VNC server, make sure to scale storage and memory according to how many users will be created to use the machine. For example: for 15 users, a conservative configuration might be 8 CPUs, 16G RAM, and 200G storage.

the script `vncuseradd` can be used to create a new user that has VNC capabilities. you should *NOT* use VNC for the main user created on the account as this will prevent login from the console GUI.

you can use vncuseradd to add users in bulk, like this: `vncuseradd -p PASSWORD LOGIN1 LOGIN2` etc.. or using bash expansion, like this: `vncuseradd -p PASSWORD LOGIN{1..5}` which will create 5 users called LOGIN# and assign them displays in VNC.

this is not a secure method of setting passwords, and by default the VNC password will just be "password" - have users change their passwords with `passwd` and `vncpasswd` once they log in.
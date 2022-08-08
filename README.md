# fedora-pxe-setup
Setup repository for a stub Fedora PXE server.

Loosely based on [Fedora PXE Setup](https://docs.fedoraproject.org/en-US/fedora/latest/install-guide/advanced/Network_based_Installations/#pxe-overview).

this is a companion "slave" server that can be used with the main fedora-pxe-setup repo, https://github.com/sjtrotter/fedora-pxe-setup which is a full pxe server installation.

Files: 
- default-automenu - BIOS menu configuration - automatically generated from the kickstarts available on the remote server.
- dhcpd.conf - dhcpd configuration - automatically generated from ansible facts; uses the subnet it is connected on to serve dhcp, next server, and dns information to clients. **You need to set up networking via nmtui** (from package NetworkManager-tui) for this to work right, because it pulls DNS from the subsequent nmconnection file, and you should be using a manual IP address.
- grub-automenu.cfg - GRUB menu configuration - automatically generated from the kickstarts available on the server.
- LICENSE - license for the project, GPL 3.0
- PXE Server SOP.docx - Standard Operating Procedure for various PXE Server tasks and administration.
- pxe-stub.yml - ansible playbook that sets up a base PXE server using the files in this directory. See pxe-setup flow for more info.
- README.md - this file.

# flow of pxe-stub.yml

**this is a high-level overview and is not an exhaustive list of what happens; see the yml for more info**. the ansible script will download needed components for the server (dhcp-server, tftp-server) and then syslinux and grub-x64. It copies the relavant files from the latter two packages into /var/lib/tftpboot, the pxe server directory. it writes the template config files to the relevant locations and then opens the firewall to allow the services to be accessible, and starts/restarts and enables the services.

IF any files are changed on the pxe server, i.e. the IP address changes or any configurations are updated, the pxe-stub.yml should be run again with `sudo ansible-playbook pxe-stub.yml` within this directory.

# HOWTO: Documentation
See documentation on main pxe server: [HOWTO: Documentation](https://github.com/sjtrotter/fedora-pxe-setup#howto-documentation)

## howto: Setup from-scratch Stub PXE Server
0. Setup remote PXE server
    - like shown in [fedora-pxe-setup](https://github.com/sjtrotter/fedora-pxe-setup)
    - Ensure networking is set up to where this server is available wherever you are installing this stub from.
    - Record its IP address/DNS name for later (the one you can access from the internet).
1. create VM / Prep Server for Install
    - 2 CPUs
    - 2048 MB / 2 GB RAM
    - 50 GB Storage
    - for VM, Bridged Network Adapter, or in ESXi, connected to the subnet needed to image from.
2. Get ISO
    - Use latest Fedora Linux Server DVD: https://download.fedoraproject.org/pub/fedora/linux/releases/ (and then navigate to version > Server > x86_64 > iso and download the dvd iso)
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
        - change the root (/) partition to be the max size; click it and change it to 50 GB and click Apply Changes. it will auto-adjust.
        - Click Done, then click Accept Changes
    - Create user; make sure they are an admin. (you may have to scroll down to see the user creation)
    - Software Selection: Choose Minimal Install if available, or choose Fedora Custom OS.
    - Click to begin the install, and reboot when done.
5. First Boot
    - `sudo dnf update`
    - reboot if new kernel installed (it probably was)
    - `sudo dnf install ansible git NetworkManager-tui python3-netaddr`
    - `git clone https://github.com/sjtrotter/fedora-pxe-stub.git`
    - Use nmtui to set network information manually. Make sure you set the IP with a /XX for the CIDR, and make sure you set the DNS and Gateway appropriately. ( run `nmtui` )
    - Edit pxe-stub.yml - `vi fedora-pxe-stub/pxe-stub.yml` and change the variable at the top to the correct IP address. this line: `pxe_main_IP: 192.168.1.250`
    - `sudo ansible-playbook fedora-pxe-stub/pxe-stub.yml` (make sure you use the path into the git clone, if you have cd'd elsewhere)
6. Done.
    Once the ansible playbook completes successfully, the server is ready to PXE boot devices - as long as the remote server is set up.

## howto: Setup from-VM-clone PXE server
This assumes you have previously set up a PXE server as a VM according to above and you wish to clone it.
1. Clone/move as necessary.
2. Boot from console:
    - Login, then use nmtui ( run `nmtui` ) to set network information manually.
    - `cd fedora-pxe-stub` and then `git pull` to update files
    - `sudo ansible-playbook pxe-stub.yml` to update and place all files.
3. Done.
    Once playbook completes successfully, the server is ready to PXE boot devices - as long as the remote server is set up.

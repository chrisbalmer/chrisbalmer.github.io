---
layout: post
title: Build a RHEL 7.1 or CentOS 7.1 Vagrant Box for VMware
categories: [vagrant]
tags: [vagrant, linux, rhel, centos, vmware]
fullview: false
---
These instructions will cover RHEL and CentOS. Some sections will be limited to a specific distribution and will be marked as such. Screenshots provided show RHEL 7.1.

## Install your VM

- Create a new VM and point VMware to the RHEL 7.1 Server ISO, choose to customize it at the end.
- Remove the built in camera by choosing camera from the settings and clicking remove.
- Adjust any other settings as desired (RAM, vCPU, etc).
- Start the VM.
- Select Install and press enter (I usually skip the installation source verfication as I always check the ISO with the hash).
![Install CentOS.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/install-skip-test.png)

- Select your language.
![Select language.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/install-select-language.png)

- Click `Installation Destination` and then just press `Done` in the upper left.
![After choosing destination.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/install-summary-after-dest.png)

- Click `Network & Host Name`.
- Click `Configure`.
- Click the `General` tab.
- Tick `Automatically connect to this network when it is available`.
![Edit NIC settings.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/install-edit-nic.png)

- Click `Save`.
![After editing NIC.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/install-after-edit-nic.png)

- Click `Done`.
- Click `Begin Installation`.
![Set root password.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/install-summary-begin.png)

- Click `Root Password`
- Enter `vagrant` for the root password and verify it.
![Set root password.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/install-root-password.png)

- Click `Done` twice.
- Click `User Creation`.
- Enter `vagrant` for the `full name`, `username`, `password` and `confirm password`.
- Tick `Make this user administrator`.
![Set vagrant password.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/install-vagrant-password.png)

- Click `Done` twice.
- Wait for the install to finish.
![Wait for install to finish.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/finish-install.png)

- When the installation finishes, click on `Reboot`.
![Click reboot.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/install-finished-reboot.png)


## Connect to your VM

- Log into the console under the user vagrant after it reboots.
- Find the name of your network connection:

~~~~~~~~
sudo nmcli con show
~~~~~~~~

- Find the IP of your network connection using the device name from the above command:

~~~~~~~~
sudo nmcli con show eno16777736 | grep IP4.ADD
~~~~~~~~

![Get IP Address.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/get-ip-address.png)

- Use SSH on your host machine to connect to your VM.
![Connect via SSH.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/connect-via-ssh.png)


## Configure your VM

### Red Hat Specific Subscription and Repo Config (RHEL Only)
Since we need to update the VM and also add some packages, we will need to register it with Red Hat so it can connect to the RH repos. After everything is done, we will unregister it to clean up everything nicely. When you use the box in Vagrant you can then register each VM independently as needed. There is some information in the notes at the end for how to handle this.

- Register Red Hat Subscription:

~~~~~~~~
sudo subscription-manager register --username <red hat account username>
~~~~~~~~

- Then enter your password when prompted.

![Register RHEL.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/register-rhel.png)

- Attach Red Hat Subscription:

~~~~~~~~
sudo subscription-manager attach --auto
~~~~~~~~

![Attach RHEL subscription.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/attach-rhel.png)


### Install Prereq files

~~~~~~~~
sudo yum update -y
sudo yum install yum-utils net-tools gcc make open-vm-tools kernel-headers   kernel-devel perl -y
~~~~~~~~

![YUM update.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/yum-update.png)
![YUM install.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/yum-install.png)
![After YUM install.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/yum-install-complete.png)

- The packages `gcc`, `make`, `kernel-headers`, `kernel-devel`, and `perl` are for building the HGFS driver that enables folder sharing in VMware.

- The package `net-tools` is for Vagrant to configure the network adapters.

- The package `open-vm-tools` is for the preferred method of adding VMware Tools (aside from the HGFS driver).

- The package `yum-utils` is for bootstrap installers (like SaltStack) which need to add additional repos. If you have Vagrant install Salt-Master or Salt-Minion on the VM, this will be required as it will need to add the `epel-release` repository. You will also need to register your subscription before having salt installed (see notes below for an option on how to register subscriptions on `vagrant up`).

### Configure Sudoers
Vagrant requires the `vagrant` user to be able to run `sudo` without a password. To do this we configure a group named `admin` to be able to access `sudo` without a password. Then we add the `vagrant` user to this group.

In addition to this, Vagrant will not work correctly if `sudo` requires a `tty`. So we comment out the line requiring this in the `/etc/sudoers` file. This line exists by default on CentOS and RHEL.

~~~~~~~~
sudo sed -i 's/^Defaults\s*requiretty/# Defaults requiretty/g' /etc/sudoers
echo "%admin  ALL=NOPASSWD: ALL" | sudo tee -a /etc/sudoers
sudo groupadd admin
sudo usermod -G admin vagrant
~~~~~~~~

![Configure sudoers and accounts.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/configure-accounts.png)

### Configure SSH
Vagrant needs a generic SSH key to have access to the `vagrant` user. Once it connects the first time it will then replace this with a randomly generated key for security. To grant this initial access we create the `.ssh` folder and download the public key into the `authorized_keys` file for the SSH key Vagrant has built in. Then we configure the permissions so openSSH will correctly authenticate with the key.

~~~~~~~~
cd ~/
mkdir .ssh
curl https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub > .ssh/authorized_keys`
chmod 700 .ssh/
sudo chmod 600 .ssh/authorized_keys
sudo chown -R vagrant:vagrant .ssh/
~~~~~~~~

![Configure SSH.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/import-ssh-key.png)

### Reboot
Because of the kernel update, you will need to reboot the VM. The HGFS driver will not compile otherwise and Vagrant will fail when it tries to mount shared folders.

~~~~~~~~
sudo reboot
~~~~~~~~

### Install HGFS Driver
Insert the VMware Tools installation CD via VMware Fusion or Workstation. It will be listed as `Reinstall VMware Tools` in the menu since the open-vm-tools driver is running. This is normal. We need this part to get the HGFS driver that open-vm-tools does not come with.

~~~~~~~~
cd ~/
sudo mount /dev/cdrom /media
tar xvf /media/VMwareTools-*.tar.gz
sudo ./vmware-tools-distrib/vmware-install.pl --default
rm -rf  vmware-tools-distrib/
~~~~~~~~

![Sucessful install.]({{ site.url }}/assets/media/2015-07-02-build-rhel-centos-7-vagrant-box/done-installing-vmware-tools.png)

## Cleanup

### Clear Red Hat Subscription (RHEL Only)
Because you may not want the box file associated with your RHEL account, we will unregister it here. Then whomever uses the box file in the future can reregister it on `vagrant up`.

~~~~~~~~
sudo yum clean all
sudo subscription-manager remove --all
sudo subscription-manager unregister
~~~~~~~~

### Remove Temp Data and Shutdown
This is just to clear out temporary files and anything else unnecessary for the packaging of the VM as a box file. This information can be regenerated by the system when needed (i.e. yum cache).

~~~~~~~~
sudo yum clean all
sudo rm -rf /tmp/*
rm ~/.bash_history
history -c
sudo poweroff
~~~~~~~~

## Convert VM to Vagrant Box
*These commands are from OS X. You will need to modify them appropriately for your host OS if it is not OS X.*

- Open terminal and go to the directory of your VM (example is a VM named `vagrant-centos7-1503-x64`):

~~~~~~~~
cd ~/Documents/Virtual\ Machines.localized/Source_Images/vagrant-centos7-1503-x64.vmwarevm/
~~~~~~~~

- Defragment the disk:

~~~~~~~~
/Applications/VMware\ Fusion.app/Contents/Library/vmware-vdiskmanager -d Virtual\ Disk.vmdk
~~~~~~~~

- Shrink the disk:

~~~~~~~~
/Applications/VMware\ Fusion.app/Contents/Library/vmware-vdiskmanager -k Virtual\ Disk.vmdk
~~~~~~~~

- Clear any logs:

~~~~~~~~
rm *.log
~~~~~~~~

- Add required metadata (change appropriately if using VMware Workstation):

~~~~~~~~
echo '{' > metadata.json
echo '    "provider": "vmware_fusion"' >> metadata.json
echo '}' >> metadata.json
~~~~~~~~

- Create box file for import:

~~~~~~~~
tar cvzf ../vagrant-centos7-1503-x64.box ./*
~~~~~~~~

- Import the box file:

~~~~~~~~
vagrant box add vagrant-centos7-1503-x64 ../vagrant-centos7-1503-x64.box
~~~~~~~~

- Remove the box file (you can save if desired):

~~~~~~~~
rm vagrant-centos7-1503-x64.box
~~~~~~~~

## Notes

### Saving the VM
Optionally you can delete your VM after you're done. However I recommend saving it to speed up adding or changing things on it later on. You can just boot it up, make the changes and rerun the cleanup and conversion to a Vagrant box steps.

### Network Changes During Provisioning
Because of the way Vagrant handles network settings, if you need to add or change network settings you will want to reboot the NetworkManager and network services when the box is provisioned and after the changes. Example shell provisioning:

~~~~~~~~
config.vm.provision "shell", inline: <<-SHELL
  sudo systemctl restart NetworkManager
  sudo systemctl restart network
SHELL
~~~~~~~~

Without this, it will still work but if, for example, you add a new NIC, it won't have an IP until after you reboot. This quick provisioning script resolves that issue.

### Fedora
I originally wrote this with Fedora 22 as one of the distributions however I could not get the networking to work correctly. These instructions do cover Fedora 22 but when you run `vagrant up` you will get a failure due to the networking configuration. I am unsure of what the fix is at the moment. When I resolve the issue I will add in Fedora 22 to this post with the few differences needed.

Example error:

~~~~~~~~
Bringing machine 'fedora22test' up with 'vmware_fusion' provider...
==> fedora22test: Cloning VMware VM: 'vagrant-fedora22-x64'. This can take some time...
==> fedora22test: Verifying vmnet devices are healthy...
==> fedora22test: Preparing network adapters...
==> fedora22test: Starting the VMware VM...
==> fedora22test: Waiting for machine to boot. This may take a few minutes...
    fedora22test: SSH address: 192.168.226.134:22
    fedora22test: SSH username: vagrant
    fedora22test: SSH auth method: private key
    fedora22test:
    fedora22test: Vagrant insecure key detected. Vagrant will automatically replace
    fedora22test: this with a newly generated keypair for better security.
    fedora22test:
    fedora22test: Inserting generated public key within guest...
    fedora22test: Removing insecure key from the guest if its present...
    fedora22test: Key inserted! Disconnecting and reconnecting using new SSH key...
==> fedora22test: Machine booted and ready!
==> fedora22test: Forwarding ports...
    fedora22test: -- 22 => 2222
==> fedora22test: Setting hostname...
The following SSH command responded with a non-zero exit status.
Vagrant assumes that this means the command failed!

service network restart

Stdout from the command:

Restarting network (via systemctl):  [FAILED]


Stderr from the command:

Job for network.service failed. See "systemctl status network.service" and "journalctl -xe" for details.
~~~~~~~~

### Managing RHEL Subscription Registration (RHEL Only)
I recommend installing the [vagrant-registration](https://github.com/projectatomic/adb-vagrant-registration) plugin for managing your RHEL subscriptions. You can install this via `vagrant plugin install vagrant-registration`. Once installed you can create a file in your home directory to always register your RHEL subscription on `vagrant up` and remove it on `vagrant halt` and `vagrant destroy`.

From the GitHub documentation linked above:

~~~~~~~~
mkdir ~/.vagrant.d/
cat >> ~/.vagrant.d/Vagrantfile << EOF
if Vagrant.has_plugin?('vagrant-registration')
  config.registration.username = 'foo'
  config.registration.password = 'bar'
end
EOF
~~~~~~~~

Additionally you'll need to add in any required repos via a SHELL script. Example that covers repos needed for bootstrapping SaltStack:

~~~~~~~~
config.vm.provision "shell", run: "always", inline: <<-SHELL
  sudo subscription-manager repos --enable rhel-7-server-optional-rpms
  sudo subscription-manager repos --enable rhel-7-server-extras-rpms
SHELL
~~~~~~~~

There are some caveats I've come across using this method so far:

- Your RH creds are stored in plaintext. There may be a better way using a built in keychain package or something else to encrypt/obfuscate the creds. I highly recommend not having shared passwords in your accounts in general and this highlights one of the reasons for this.

- If you `vagrant halt` and then `vagrant up` it asks you if you want to register it and if you answer `yes` it will then ask you for your RH username and password even though it already has them. (Doesn't always happen)

- It has no support for adding repos so you have to use a shell provisioner to add repos and make sure it runs anytime it reregisters because it unregisters on halt.

## Closing
This is one of my first complete tutorials. I hope everything has been covered adequately from start to finish. I found a lot of them missed chunks, made assumptions or were outdated when looking for help building my own clean box files for Linux distros.

As mentioned, Fedora 22 will be added once I work out a few kinks. I also have a Debian 8.1 in the works as well that has a few issues that need to be resolved.

Chris

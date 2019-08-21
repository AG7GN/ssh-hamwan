# ssh-hamwan
Version 20190820

## Background

HamWAN uses frequencies that are allocated for use by Amateur Radio operators.  As such, the use of those frequencies is subject to FCC Part 97.  Part 97 prohibits passing "messages encoded for the purpose of obscuring their meaning".  This means that you cannot use the privacy features of cryptography because it would "obscure the message".  This does not ban cryptography on ham frequencies outright, though.  The [HamWAN web site](http://hamwan.org) has a nice summary of how Part 97 affects HamWAN entitled [Internet and Part 97](http://hamwan.org/Administrative/Internet%20and%20Part%2097.html).

By design, Secure Shell (SSH) uses encryption to prevent prying eyes from seeing traffic between hosts.  SSH is commonly used to manage hosts and other end systems.  This document describes how to install __ssh-hamwan__, an OpenSSH server and client, on a Raspberry Pi.  It allows SSH to be used without encrypting the traffic.  __ssh-hamwan__ can operate concurrently with the regular SSH client and server that's installed and usually enabled on most Linux hosts including the Pi.

## Notes

__IMPORTANT:__ The __ssh-hamwan__ applications should only be used when the traffic is carried on frequencies that are subject to FCC Part 97 rules.  The standard SSH server and client installed and usually enabled by default on the Raspberry Pi and other Linux/UNIX hosts remains fully operational with all of the latest encryption features enabled.  Use the standard SSH server and client when your traffic is carried over the Internet or your local network and not over Part 97 controlled frequencies.

The __ssh-hamwan__ build of OpenSSH differs from a typical OpenSSH installation in the following ways:

- It is built without __OpenSSL__ to make it more portable.
- The client binaries are named `ssh-hamwan`, `scp-hamwan`, `sftp-hamwan` and are located in `/usr/local/bin`.  
- The server is called `sshd-hamwan` and is located in `/usr/local/sbin`.
- The configuration files and keys are located in `/usr/local/etc/ssh-hamwan`.
- The client initiates connections to the server on TCP port 222 by default rather than the usual SSH TCP port 22.  The server listens on TCP port 222 by default.
- The server has option `NoneEnabled` set to `yes` in `/usr/local/etc/ssh-hamwan/sshd_config`.  The client has both `NoneEnabled` and `NoneSwitch` options set to `yes` for all hosts in `/usr/local/etc/ssh-hamwan/ssh_config`.
- Since there is no OpenSSL support, only `ed25519` keys, which are supported by OpenSSH natively, are used.

## Prerequisites

- Raspberry Pi 3B or 3B+ or 4 running Raspbian Stretch or Buster
- Pi must be connected to the Internet (NOT via HamWAN for this installation procedure)
- Familiarity with basic Linux commands in the Terminal application
- User __pi__ has passwordless sudo privileges

## Installation on Raspberry Pi (Stretch or Buster)

### Install the Package

- Open a Terminal and run these commands:

		cd ~
		sudo apt-get update
		sudo apt-get -y install git
		rm -rf ssh-hamwan
		git clone https://github.com/AG7GN/ssh-hamwan
		cd ssh-hamwan
		cd ~/ssh-hamwan
		sudo dpkg -i ssh-hamwan_6.8p1-9_armhf.deb 

### Add and Enable the `ssh-hamwan.service`

- As root, open a text editor and create a file called `/lib/systemd/system/ssh-hamwan.service` with the following content:

		[Unit]
		Description=OpenBSD Secure Shell server allows no encryption for HamWAN use
		After=network.target auditd.service
		ConditionPathExists=!/usr/local/etc/ssh-hamwan/sshd_not_to_be_run
		
		[Service]
		PIDFile=/var/run/sshd-hamwan.pid
		ExecStartPre=/usr/local/sbin/sshd-hamwan -t
		ExecStart=/usr/local/sbin/sshd-hamwan -D -f /usr/local/etc/ssh-hamwan/sshd_config
		ExecReload=/usr/local/sbin/sshd-hamwan -t
		ExecReload=/bin/kill -HUP $MAINPID
		KillMode=process
		Restart=on-failure
		
		[Install]
		WantedBy=multi-user.target
		Alias=sshd-hamwan.service
		
- Save and close the file.

## Server Operation

- Enable the service

	- Run these commands in the Terminal:
	
			sudo systemctl enable ssh-hamwan.service
			sudo systemctl start ssh-hamwan.service
		
- Verify the service is operational:

	- Run this command in the Terminal:
	
			sudo ss -plunt | grep "ssh\|^Netid"
		
	The output should look similar to this:
	
		Netid  State    Recv-Q Send-Q Local Address:Port  Peer Address:Port              
		tcp    LISTEN   0      128    *:22                *:*              Users:(("sshd",pid=596,fd=3))
		tcp    LISTEN   0      128    *:222               *:*              users:(("sshd-hamwan",pid=580,fd=3))
		tcp    LISTEN   0      128    :::22               :::*             users:(("sshd",pid=596,fd=4))
		tcp    LISTEN   0      128    :::222              :::*             users:(("sshd-hamwan",pid=580,fd=4))
		
	Note that the regular SSH service __sshd__ is listening on port 22 and the __sshd-hamwan__ service is listening on port 222.
	
- Modify your firewall rules if necessary to allow TCP port 222 inbound.

## Client Operation

By default (and this behavior can be changed in `/usr/local/etc/ssh-hamwan/ssh_config`), the ssh-hamwan client will encrypt the authentication part of the conversation only (your password is encrypted), but once authenticated, all other traffic is sent in the clear.  This approach is believed to be in compliance with Part 97 rules.

	ssh-hamwan user@host
		
Where user is the username on the target host and host is the target hostname or IP address.  This command will initiate an SSH connection to TCP port 222 for `user` at `host`.

If you want all traffic, including authentication, passed in the clear, use this command instead:

	ssh-hamwan -c none user@host

Note that the server must also support "No Encryption" (`NoneEnabled`) in order for this to work.  The server in __ssh-hamwan__ is configured with `NoneEnabled` set to `yes`.

For file transfer, `scp-hamwan` and `sftp-hamwan` work in in the same way.

You can generate your own private and public keys for use with ssh-hamwan.  Open a Terminal and run these commands:

	cd ~/.ssh
	ssh-keygen-hamwan -t ed25519

and follow the instructions.  You can optionally use a passphrase to protect your private key (recommended).  If you use a passphrase, you'll be prompted to enter it whenever ssh-hamwan needs to use your key.

## (Optional) Compling and Building ssh-hamwan from Source

This section documents how to configure and build the [openssh-6.8p1](http://anduin.linuxfromscratch.org/~bdubbs/blfs-book-xsl/postlfs/openssh.html) software to allow it to be used over HamWAN without encryption.  You don't need to do these steps if you are simply installing the Debian package in this repository on your Raspberry Pi.  See the previous __Installation on Raspberry Pi (Stretch or Buster)__ section for how to install the Debian package on your Pi.  

The patch modifies various files in the OpenSSH source code to allow no encryption to be used.

- Install required packages and sources

	Open a Terminal and run these commands:
	
		cd ~
		sudo apt-get update
		sudo apt-get -y install git
		sudo apt-get -y build-dep openssh
		sudo groupadd sshd
		sudo useradd -g sshd -c 'sshd privsep' -d /var/empty -s /bin/false sshd 
		rm -rf ssh-hamwan
		git clone https://github.com/AG7GN/ssh-hamwan
		cd ssh-hamwan
		wget http://ftp.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-6.8p1.tar.gz
		tar xzf openssh-6.8p1.tar.gz
		cd openssh-6.8p1
		patch -p1 < ../openssh-6.8p1-HamWAN-9.patch
		./configure --prefix=/usr/local --sysconfdir=/usr/local/etc/ssh-hamwan --with-pam --with-pid-dir=/var/run --with-privsep-path=/var/empty --without-openssl
		make -j8
		sudo make install
		
	To build the Raspberry Pi Debian package, I ran the following command instead of `sudo make install` in the previous step:
	
		sudo checkinstall --pkgname ssh-hamwan --pkgversion 6.8p1 --pkgrelease 9 --pkggroup hamradio --pkgsource http://www.linuxfromscratch.org/blfs/view/7.6/postlfs/openssh.html --maintainer nobody@example.com --provides ssh-hamwan make install
		






	  

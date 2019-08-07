# ssh-hamwan

## Background

HamWAN uses frequencies that are allocated to for use by Amateur Radio operators.  As such, the use of those frequencies is subject to FCC Part 97.  Part 97 prohibits passing "messages encoded for the purpose of obscuring their meaning".  This means that you cannot use the privacy features of cryptography because it would "obscure the message".  This does not ban cryptography on ham frequencies outright, though.

The [HamWAN](http://hamwan.org) has a nice summary entitled [Internet and Part 97](http://hamwan.org/Administrative/Internet%20and%20Part%2097.html).

By design, Secure Shell (SSH) uses encryption to prevent prying eyes from seeing traffic between hosts.  SSH is commonly used to manage hosts and other end systems.  This document describes how to build and install a second SSH server and client on a Raspberry Pi that allows SSH to be used without encrypting the traffic without having to modify the source code of OpenSSH.

The second ssh server should only be used when the traffic is carried on frequencies that are subject to FCC Part 97 rules.  The standard SSH server and client installed by default on the Raspberry Pi remain fully operational with all of the latest encryption features enabled.

This procedure will install [HPN-SSH](https://www.psc.edu/hpn-ssh) (High Performance SSH) server and client, which allows encryption to be disabled in the `sshd_config` file.

## Prerequisites

- Raspberry Pi 3B or 3B+ or 4 running Raspbian Stretch or Buster
- Pi must be connected to the Internet (NOT via HamWAN for this installation procedure)
- Familiarity with basic Linux commands in the Terminal application
- User pi has passwordless sudo privileges

## Installation

- Install required packages and sources

	Open a Terminal and run these commands:
	
		cd ~
		sudo apt-get update
		sudo apt-get -y install autoconf git
		sudo apt-get -y build-dep openssh
		git clone https://github.com/rapier1/openssh-portable
		cd openssh-portable/
		autoreconf
		./configure
		make
		sudo make install

- Rename the binaries.  Run these commands in the Terminal:

		cd /usr/local/bin
		sudo mv ssh ssh-hamwan
		sudo mv ssh-add ssh-hamwan-add
		sudo mv ssh-agent ssh-hamwan-agent
		sudo mv ssh-keygen ssh-hamwan-keygen
		sudo mv ssh-keyscan ssh-hamwan-keyscan
		sudo mv sftp sftp-hamwan
		sudo mv scp scp-hamwan
		cd /usr/local/sbin
		sudo mv sshd sshd-hamwan
		cd ~

- As root, open a text editor and create a file called `/lib/systemd/system/ssh-hamwan.service` with the following content:

		[Unit]
		Description=OpenBSD Secure Shell server allows no encryption for HamWAN use
		After=network.target auditd.service
		ConditionPathExists=!/etc/ssh/sshd_not_to_be_run

		[Service]
		EnvironmentFile=-/etc/default/ssh
		PIDFile=/var/run/sshd-hamwan.pid
		ExecStartPre=/usr/local/sbin/sshd-hamwan -t
		ExecStart=/usr/local/sbin/sshd-hamwan -D -f /usr/local/etc/sshd_config $SSHD_OPTS
		ExecReload=/usr/local/sbin/sshd-hamwan -t
		ExecReload=/bin/kill -HUP $MAINPID
		KillMode=process
		Restart=on-failure

		[Install]
		WantedBy=multi-user.target
		Alias=sshd-hamwan.service
		
	Save and close the file.

-	As root, make a backup copy of open `/usr/local/etc/sshd_config`

		sudo cp /usr/local/etc/sshd_config /usr/local/etc/sshd_config.original

-	As root, open `/usr/local/etc/sshd_config` in a text editor.

	Change this line:
	
			#Port 22
		
	to:
	
			Port 222
		
	and change this line:
	
			#NoneEnabled no
		
	to:
	
			NoneEnabled yes

	Save and close the file.

## (OPTIONAL) Server Operation
	
If you only need the ssh client, skip this section.  You should only enable this server on hosts that are connected to HamWAN!
	
- Enable the service

	Run these commands in the Terminal:
	
		sudo systemctl enable ssh-hamwan.service
		sudo systemctl start ssh-hamwan.service
		
- Verify the service is operational:

	Run this command in the Terminal:
	
		sudo ss -plunt | grep "ssh\|^Netid"
		
	The output should look similar to this:
	
		Netid  State    Recv-Q Send-Q Local Address:Port  Peer Address:Port              
		tcp    LISTEN   0      128    *:22                *:*              Users:(("sshd",pid=596,fd=3))
		tcp    LISTEN   0      128    *:222               *:*              users:(("sshd-hamwan",pid=580,fd=3))
		tcp    LISTEN   0      128    :::22               :::*             users:(("sshd",pid=596,fd=4))
		tcp    LISTEN   0      128    :::222              :::*             users:(("sshd-hamwan",pid=580,fd=4))
		
	Note that the regular SSH service is listening on port 22 and the ssh-hamwan service is listening on port 222.

## Client Operation

On hosts where you've installed this server and client per this procedure, the client is named `ssh-hamwan`.  To connect to a `sshd-hamwan` server (also installed per this procedure), run this command from a terminal:

	ssh -p 222 -o NoneSwitch=yes user@host
		
where user is the username on the target host and host is the target.

This will encrypt your authentication (your password won't be passed "in the clear"), but the message traffic will be unencrypted.  This conforms to Part 97.

Note that most recent (within the last few years) SSH implementations do not allow `NoneSwitch=yes`, so you will likely have to install HSP-SSH as described here, and use the `ssh-hamwan` client to connect to the `sshd-hamwan` server.


	  

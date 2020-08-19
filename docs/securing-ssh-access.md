# Securing ssh access to a node on Cardano mainnet (Ubuntu 20.04)

-- DISCLAIMER: This guide is for background educational purposes only.  
-- DISCLAIMER: By using this guide, you assume sole risk and waive any claims of liability against the author.  

-- Note: This guide is for running cardano-node  on a virtual private server (VPS), running Ubuntu 20.04.  
-- Note: This guide assumes your local machine is a Mac, but most instructions are executed on the remote machine.  
-- Note: anything preceded by "#" is a comment.  
-- Note: anything all-caps in between "<>" is an placeholder; e.g. `"<FILENAME>"` could be `"foo.txt"`.  
-- Note: anything in between "${}" is a variable that will be evaluated by your shell.  

* Author: Chris Graffagnino, Cardano Ambassador (stake-pool: __MASTR__)  

First off, thank you for endeavoring to run a node on Cardano mainnet. You are a pioneer on the next financial operating-system for the world!  

Of course, with great privilege comes great responsibility. We must protect our nodes from nefarious actors that try to break in to our servers. Let's get started on our first line of defense - secure SSH access.  

## Generate private/public ssh keys
```
# Generate private & public keys on your *LOCAL MACHINE* (public key will have a ".pub" extension)
# When prompted, name it something other than "id_rsa" (in case you're using that somewhere else)
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519 -C "<YOUR EMAIL HERE>"

# Lock down private key
chmod 400 ~/.ssh/<YOUR KEY>

# Do you have brew installed?
brew -v

# Install brew if you don't have it:
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

# Now install ssh-copy-id
brew install ssh-copy-id

# Push key up to your box
# See below if using Digital Ocean for vps
ssh-copy-id -i ~/.ssh/<YOUR KEYNAME>.pub root@<YOUR VPS PUBLIC IP ADDRESS>

# Login to VPS via ssh
`ssh -i ~/.ssh/<PATH TO SSH PRIVATE KEY ON YOUR LOCAL MACHINE> root@<YOUR VPS PUBLIC IP ADDRESS>`

# Copy ssh key to non-root user
rsync --archive --chown=<USERNAME>:<USERNAME> ~/.ssh /home/<USERNAME>
```

## Change default SSH port
```
# Changing this setting REQUIRES also opening the same port with ufw (next section of this guide)
# Don't skip the ufw section, or else you will be locked out.

# Note: there is also a file called "ssh_config"... don't edit that one
nano /etc/ssh/sshd_config

# Change the line "#Port 22", to "Port <CHOOSE A PORT BETWEEN 1024 AND 65535>"
# Remember to remove the "#"

# Disabling root login is considered a security best-practice
(Change "PermitRootLogin" from "yes" to "no")

# Disabling log-in via password helps mitigate brute-force attacks
(Change "PasswordAuthentication" to "no")

# Type ctrl+o to save, ctrl+x to exit
```

## Configure "uncomplicated firewall" (ufw)
```
# Set defaults for incoming/outgoing ports
ufw default deny incoming
ufw default allow outgoing

# Open ssh port (rate limiting enabled - max 10 attempts within 30 seconds)
ufw limit proto tcp from any to any port <THE PORT YOU JUST CHOSE IN sshd_config>

# Open a port for cardano-node. This is the port other nodes will connect to.  
# Note if this is a block-producing node, consider restricting access to just your relay nodes.
ufw allow proto tcp from any to any port <CHOOSE A PORT BETWEEN 1024 AND 65535>

# NOTE: The following may interrupt cardano-node if it is already running.
# Re-enable firewall
ufw enable

# Double-check the port you chose for ssh was the same as what you set in /etc/ssh/sshd_config
grep Port /etc/ssh/sshd_config			
ufw status verbose

# Reload sshd
service sshd reload
```

## Install fail2ban
Fail2ban protects our ssh port by silently banning IP addresses that unsucessfully attempt to login over and over.
```
# Install fail2ban
apt install fail2ban

# Make a local copy of the config file
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

# Edit the file around line 288 to enable ssh protection (look for "[sshd]")
# Make this section look like the following:

port    = <YOUR NEW SSH PORT>
logpath = %(sshd_log)s
backend = %(sshd_backend)s
enabled = true
maxretry = 3

# Enable and Start fail2ban
systemctl enable fail2ban.service
systemctl start fail2ban.service
```

## What have we done to secure SSH access?
-- Changed the default port  
-- Restricted SSH access root user  
-- Restricted access by password  
-- Rate-limited the SSH port  
-- Installed  fail2ban to deny access to repeat offenders.  

## How do I check who's tried to access my SSH port?
Check your SSH logs from time to time with `journalctl -u ssh`.  

## Congratulations!
You've just secured your ssh port. If you have any questions, find me on [twitter](https://twitter.com/ChrisGraff), or at the [Cardano Forum](https://forum.cardano.org/c/english/operators-talk/119). Happy mining!



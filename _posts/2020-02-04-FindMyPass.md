---
layout: post
title: HackTM - FindMyPass
subtitle: Dumping Kdbx db from Memory
comments: false
---

* **Category:** Forensics
* **Points:** 500

## Challenge

> I managed to forget my password for my KeePass Database but luckily I had it still open and managed to get a dump of the system's memory. Can you please help me recover my password?

Author: Legacy

https://mega.nz/#!IdUVwY6I!uJWGZ932xab44H4EJ-zVAqu6_UWNJcCVA4_PPXdqCyc

https://drive.google.com/open?id=1hUlGqJZYgbWaEu7w0JnPMqgYdFr8qVJe

password: eD99mLkU

Hint! I am not very good with computers, I use my one safe password where I want to keep everything safe from hackers.


## Solution

since its a forensic challenge and we have a "Vmem" file it should be a vmware memory image
so lets start doing our usual things with volatility

> volatility -f HackTM.vmem imageinfo 
![Alt text](https://github.com/blackwarriorxtn/CTF_Writeups/blob/master/HackTM/Forensics/FindMyPass/imginf.png?raw=true)

since this is windows ,let's check all process running..

>└──╼ $volatility -f HackTM.vmem --profile=Win7SP1x86_23418 pslist
![Alt text](https://github.com/blackwarriorxtn/CTF_Writeups/blob/master/HackTM/Forensics/FindMyPass/psslist.png?raw=true)
there's a KeePass.exe and challenge description says something about finding a password so
Maybe the idea behind this is to find a way to leak the keepass .kdbx database and his master password from memory.

First thing to do is a dump of the memory allocated for pid=3620 KeePass.exe

> └──╼ $volatility -f HackTM.vmem --profile=Win7SP1x86_23418 memdump -p 3620 -D dump 

So, if there's a keepass2 database loaded into memory, then you have a XML header too..
How did I know this? read this manual : https://github.com/Stoom/KeePass/wiki/KDBX-v2-File-Format and u will find the xml format 
![Alt text](http://dann.com.br/content/images/2017/05/Selection_834.png)
so lets search that xml format in our dump using strings to confirm that we are not doing smth wrong !

> └──╼ $strings 3620.dmp |grep xml
![Alt text](https://github.com/blackwarriorxtn/CTF_Writeups/blob/master/HackTM/Forensics/FindMyPass/header.png?raw=true)

and with this we can confirm that our encrypted database can be found at memory, 

Now we still need:

. Leak the master password from memory (maybe its on clipboard?)

. Leak the entire keepass database from memory (not only the XML)

lets check the clipboard for the master password ! 
> └──╼ $volatility -f ../HackTM.vmem --profile=Win7SP1x86_23418 clipboard -v
![Alt text](https://github.com/blackwarriorxtn/CTF_Writeups/blob/master/HackTM/Forensics/FindMyPass/clipboard.png?raw=true)

the "DmVZQ" should be it so lets grep for it 
> └──╼ $strings ../HackTM.vmem |grep dmVZQmd
![Alt text](https://github.com/blackwarriorxtn/CTF_Writeups/blob/master/HackTM/Forensics/FindMyPass/masterpass.png?raw=true)

that should be the master password lets save it for later use and dump the entire .kdbx from memory !
if you remember  in the manual : 
       |   kdb      |  kdbx pre-rel | kdbx relese |
byte 1 | 0x9aa2d903 | 0x9aa2d903    | 0x9aa2d903  |  
byte 2 | 0xb54bfb65 | 0xb54bfb66    | 0xb54bfb67  |  

since its kdbx we should be searching for "0x9aa2d903 0xb54bfb67" (first byte + second byte of the kdbx rel)
but  since its in memory it will appear like this "d903 9aa2 fb67 b54b" instead
so lets grep for that hex in our keepass memory dump 
> └──╼ $hexdump 3620.dmp | grep "d903 9aa2 fb67 b54b"
![Alt text](https://github.com/blackwarriorxtn/CTF_Writeups/blob/master/HackTM/Forensics/FindMyPass/hexdump.png?raw=true)

 but the problem is that we have more than one entry of this file at memory, we need to extract the correct one or it will be  a corrupted database.
 so lets use wXhexEditor.
Search by our hex kdbx file signature 03 D9 A2 9A 67 FB 4B B5
Look for an entry that has no blank spaces breaking the header and file footer..
the first offset is a perfect example so lets dump it 
![Alt text](https://github.com/blackwarriorxtn/CTF_Writeups/blob/master/HackTM/Forensics/FindMyPass/kdbx.png?raw=true)
doing file on it shows that we dumped the right thing 
![Alt text](https://github.com/blackwarriorxtn/CTF_Writeups/blob/master/HackTM/Forensics/FindMyPass/ok.png?raw=true)
so now lets start keepass using this database and the master password we got earlier ! 
![Alt text](https://github.com/blackwarriorxtn/CTF_Writeups/blob/master/HackTM/Forensics/FindMyPass/attach.png?raw=true)
that should be it, saving that file and extracting it with the master password will give u the flag 



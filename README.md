# RedTeam-ADLab
Automated Active Directory lab deployment for RedTeam practice. Multi-forest environment with intentional vulnerabilities including Kerberoasting, delegation abuse, ADCS ESC1-ESC15, and more.

### Network Architecture

```
Forest 1: redteam.lab (192.168.100.0/24)
├── DC01 (Root DC) - 192.168.100.10
├── dev.redteam.lab (Child Domain)
│   └── DC02 (Child DC) - 192.168.100.11
├── SRV02 (SQL Server) - 192.168.100.12
├── SRV03 (IIS Web) - 192.168.100.13
└── WS01 (Workstation) - 192.168.100.20

Forest 2: external.lab (with trust)
└── DC03 (External DC) - 192.168.100.15

Attacker Machine
└── Kali Linux - 192.168.100.50 (DHCP or static)
```

## Deployment Order

### Phase 1: Domain Controllers

1. **DC01 - Root Domain Controller**
    
    - Install Windows Server 2025
    - Run DC01 script parts 1-3
    - Verify domain is operational
2. **DC02 - Child Domain Controller**
    
    - Install Windows Server 2025
    - Run DC02 script parts 1-3
    - Verify parent-child trust
3. **DC03 - External Forest**
    
    - Install Windows Server 2025
    - Run DC03 script parts 1-3
    - Configure conditional forwarder
    - Establish forest trust

### Phase 2: Member Servers

4. **SRV02 - SQL Server**
    
    - Install Windows Server 2019
    - Run SRV02 script
    - Install SQL Server Express
    - Run SQL configuration script
    - Test SQL connectivity
5. **SRV03 - Web Server**
    
    - Install Windows Server 2025
    - Run SRV03 script
    - Verify IIS is running
    - Test vulnerable web pages

### Phase 3: Workstation (Day 2-3)

6. **WS01 - User Workstation**
    - Install Windows 11
    - Run WS01 script
    - Login as domain users
    - Verify file shares access

### Phase 4: Forest Trust Configuration

7. **Establish External Trust**
    
    

## VM Resource Requirements

|Machine|RAM|CPU|Disk|OS|
|---|---|---|---|---|
|DC01|2GB|2|60GB|Server 2025|
|DC02|2GB|2|60GB|Server 2025|
|DC03|2GB|2|60GB|Server 2025|
|SRV02|4GB|2|80GB|Server 2019|
|SRV03|2GB|2|60GB|Server 2025|
|WS01|4GB|2|60GB|Windows 11|
|Kali|4GB|2|80GB|Kali Linux|

**Total: 20GB RAM, 14 CPUs, 460GB Disk**

## Critical Credentials

### redteam.lab Domain

- Administrator: `P@ssw0rd123!`
- Domain Users: `Summer2024!`
- Service Accounts: `Summer2024!`
- SQL SA: `SQLAdmin123!`

### dev.redteam.lab Domain

- Administrator: `P@ssw0rd123!`
- Domain Users: `DevPass2024!`

### external.lab Domain

- Administrator: `P@ssw0rd123!`
- Domain Users: `External2024!`

### Local Accounts

- WS01 localadmin: `LocalAdmin123!`

## Key Vulnerabilities Configured

### Domain Level

1. **Unconstrained Delegation** - DC01
2. **Constrained Delegation** - SRV03, svc_web
3. **Resource-Based Constrained Delegation** - SRV02
4. **Kerberoasting** - svc_sql, svc_iis, svc_web, svc_app, svc_external
5. **ACL Abuse** - adev → devadmin (GenericAll)
6. **Trust Relationships** - Parent-child and forest trusts

### Server Level

7. **SQL xp_cmdshell** - SRV02
8. **Command Injection** - SRV03 web app
9. **SQL Injection** - SRV03 web app
10. **Weak Service Permissions** - Multiple services

### Workstation Level

11. **AlwaysInstallElevated** - WS01
12. **Weak File ACLs** - C:\Scripts on WS01
13. **Scheduled Task Hijacking** - BackupTask on WS01
14. **Saved Credentials** - Multiple plaintext credential files

### Network Level

15. **SMB Signing Disabled** - All machines
16. **LLMNR/NBT-NS Enabled** - All machines
17. **Weak File Shares** - SQLBackups, WebFiles

## Post-Deployment Validation

Run these commands to verify your lab:

### From DC01

```powershell
# Verify domains
Get-ADDomain
Get-ADForest

# Verify trusts
Get-ADTrust -Filter *

# Verify users
Get-ADUser -Filter * | Select Name, SamAccountName

# Verify SPNs
Get-ADUser -Filter {ServicePrincipalName -like "*"} -Properties ServicePrincipalName
```

### From Attacker Machine

```bash
# Test DNS resolution
nslookup dc01.redteam.lab 192.168.100.10
nslookup dc02.dev.redteam.lab 192.168.100.10
nslookup dc03.external.lab 192.168.100.15

# Test connectivity
ping dc01.redteam.lab
ping srv02.redteam.lab
ping ws01.redteam.lab

# Test SMB shares
smbclient -L //srv02.redteam.lab -N
smbclient -L //srv03.redteam.lab -N
```

## Attack Paths to Practice

### Initial Access

1. LLMNR/NBT-NS poisoning with Responder
2. SMB relay attacks
3. Web application exploitation (SQLi, command injection)

### Privilege Escalation

4. Kerberoasting service accounts
5. AlwaysInstallElevated on WS01
6. Scheduled task hijacking
7. ACL abuse (adev → devadmin)

### Lateral Movement

8. Pass-the-Hash attacks
9. Overpass-the-Hash
10. SQL Server linked servers
11. Constrained delegation abuse
12. RBCD attacks

### Domain Dominance

13. DCSync attacks
14. Golden Ticket attacks
15. Silver Ticket attacks
16. Unconstrained delegation exploitation
17. Trust relationship abuse
18. Cross-forest attacks

## Tools to Install on Kali

```bash
# Essential AD tools
sudo apt update
sudo apt install -y python3-impacket bloodhound neo4j crackmapexec

# Additional tools
sudo apt install -y smbclient nmap responder enum4linux-ng

# Install Rubeus, Mimikatz, PowerView (transfer to Windows machines)
# Download BloodHound ingestors
```

## BloodHound Collection

From Windows machine with SharpHound:

```powershell
.\SharpHound.exe -c All -d redteam.lab --outputdirectory C:\Temp
.\SharpHound.exe -c All -d dev.redteam.lab --outputdirectory C:\Temp
.\SharpHound.exe -c All -d external.lab --outputdirectory C:\Temp
```

From Linux with bloodhound-python:

```bash
bloodhound-python -u jsmith -p 'Summer2024!' -d redteam.lab -ns 192.168.100.10 -c all
```

## Troubleshooting

### DNS Issues

- Ensure all machines point to DC01 (192.168.100.10) as primary DNS
- Verify forward and reverse lookup zones on DC01

### Trust Issues

- Verify time synchronization between DCs
- Check DNS resolution between forests
- Verify conditional forwarders are configured

### Service Account Issues

- Ensure SPNs are properly registered
- Verify service accounts have proper permissions
- Check service startup accounts

## Lab Maintenance

### Snapshot Strategy

Take snapshots after each phase:

1. After DC configuration (clean domain state)
2. After server configuration
3. After workstation configuration
4. Before each practice session

### Reset Procedure

To reset lab to clean state:

1. Revert all VMs to "Clean Configuration" snapshot
2. Verify all services are running
3. Check domain trust relationships
4. Verify DNS resolution

## Additional Resources

- **BloodHound**: https://github.com/BloodHoundAD/BloodHound
- **PowerView**: https://github.com/PowerShellMafia/PowerSploit
- **Impacket**: https://github.com/SecureAuthCorp/impacket

## Security Notice

**⚠️ WARNING**: This lab contains intentional security vulnerabilities.

- NEVER deploy this on production networks
- Use isolated lab network only
- Do not connect to the internet
- Take proper VM snapshots before practice sessions

---

Good luck with your RedTeam preparation! 🚀

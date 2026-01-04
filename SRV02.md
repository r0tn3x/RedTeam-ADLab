# ============================================
# SRV02 - SQL Server Setup (redteam.lab)
# IP: 192.168.100.12
# ============================================

# Run this script as Administrator

# Set static IP
$IPAddress = "192.168.100.12"
$PrefixLength = 24
$Gateway = "192.168.100.1"
$DNS = "192.168.100.10"  # DC01

New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress $IPAddress -PrefixLength $PrefixLength -DefaultGateway $Gateway
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses $DNS

# Set computer name
Rename-Computer -NewName "SRV02" -Force

Write-Host "Please restart and run Part 2 of this script" -ForegroundColor Yellow
# Restart-Computer -Force

# ============================================
# PART 2 - Join domain and install SQL
# ============================================

# Join domain
$DomainCred = Get-Credential -Message "Enter REDTEAM\Administrator credentials"
Add-Computer -DomainName "redteam.lab" -Credential $DomainCred -Restart

# ============================================
# PART 3 - Run after domain join restart
# ============================================

# Move computer to correct OU (run from DC01 or as Domain Admin)
# Get-ADComputer -Identity SRV02 | Move-ADObject -TargetPath "OU=REDTEAM-Servers,DC=redteam,DC=lab"

# Install SQL Server 2019 Express (download first from Microsoft)
# Download URL: https://go.microsoft.com/fwlink/?linkid=866658

# For lab purposes, we'll configure SQL manually or use this automated approach:
Write-Host "Installing SQL Server..." -ForegroundColor Cyan

# Create SQL install configuration SQLConfig.ini
[OPTIONS]
ACTION="Install"
FEATURES=SQLENGINE
INSTANCENAME="MSSQLSERVER"

SQLSVCACCOUNT="REDTEAM\svc_sql"
SQLSVCPASSWORD="Summer2024!"

SQLSYSADMINACCOUNTS="REDTEAM\sqladmin" "REDTEAM\Domain Admins"

SECURITYMODE="SQL"
SAPWD="SQLAdmin123!"

TCPENABLED="1"
NPENABLED="1"
BROWSERSVCSTARTUPTYPE="Automatic"

IACCEPTSQLSERVERLICENSETERMS="True"

# If SQL installer is available, run:
# .\SQLServer2019-x64-ENU.exe /ConfigurationFile=C:\SQLConfig.ini /IACCEPTSQLSERVERLICENSETERMS /QUIET

# Manual SQL Configuration after install:
Write-Host ""
Write-Host "After SQL is installed, configure it with these settings:" -ForegroundColor Yellow
Write-Host "1. Enable SQL Authentication"
Write-Host "2. Enable xp_cmdshell"
Write-Host "3. Create SQL login 'sa' with password 'SQLAdmin123!'"
Write-Host "4. Service account: REDTEAM\svc_sql"

# ============================================
# SQL Configuration Script (Run in SSMS or sqlcmd)
# ============================================

$SQLConfig = @"
-- Enable xp_cmdshell (VULNERABLE)
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;

-- Enable SQL Server Authentication
EXEC xp_instance_regwrite N'HKEY_LOCAL_MACHINE', 
N'Software\Microsoft\MSSQLServer\MSSQLServer', 
N'LoginMode', REG_DWORD, 2;

-- Create vulnerable SQL users
CREATE LOGIN sqladmin WITH PASSWORD = 'SQLAdmin123!';
ALTER SERVER ROLE sysadmin ADD MEMBER sqladmin;

CREATE LOGIN webapp WITH PASSWORD = 'WebApp2024!';
GRANT EXECUTE TO webapp;

-- Create test database
CREATE DATABASE TestDB;
GO

USE TestDB;
GO

-- Create table with sensitive data
CREATE TABLE Users (
    UserID INT IDENTITY(1,1) PRIMARY KEY,
    Username NVARCHAR(50),
    Password NVARCHAR(100),
    Email NVARCHAR(100),
    IsAdmin BIT
);

-- Insert sample data
INSERT INTO Users (Username, Password, Email, IsAdmin) VALUES
('admin', 'AdminPass123!', 'admin@redteam.lab', 1),
('jsmith', 'JohnPassword', 'jsmith@redteam.lab', 0),
('sconnor', 'Sarah2024!', 'sconnor@redteam.lab', 0);

-- Create stored procedure vulnerable to SQL injection
CREATE PROCEDURE GetUserByName
    @Username NVARCHAR(50)
AS
BEGIN
    DECLARE @SQL NVARCHAR(MAX);
    SET @SQL = 'SELECT * FROM Users WHERE Username = ''' + @Username + '''';
    EXEC sp_executesql @SQL;
END;
GO
"@

$SQLConfig | Out-File "C:\SQLSetup.sql" -Encoding ASCII

# Configure SQL Server service to run as domain account
# This should be done during SQL installation with the service account

# Configure SQL Server for network access
New-NetFirewallRule -DisplayName "SQL Server" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow

# Enable SQL Browser
Set-Service -Name "SQLBrowser" -StartupType Automatic
Start-Service -Name "SQLBrowser"

# Disable SMB Signing
Set-SmbClientConfiguration -RequireSecuritySignature $false -Force
Set-SmbServerConfiguration -RequireSecuritySignature $false -Force

# Create vulnerable share
New-Item -Path "C:\SQLBackups" -ItemType Directory -Force
New-SmbShare -Name "SQLBackups" -Path "C:\SQLBackups" -FullAccess "Everyone"

# Configure Resource-Based Constrained Delegation (RBCD) vulnerability
# Run this from DC01 as Domain Admin:
Write-Host ""
Write-Host "RBCD Configuration (Run on DC01):" -ForegroundColor Yellow
Write-Host '$TargetComputer = Get-ADComputer SRV02' -ForegroundColor White
Write-Host '$SourceComputer = Get-ADComputer WS01' -ForegroundColor White
Write-Host 'Set-ADComputer $TargetComputer -PrincipalsAllowedToDelegateToAccount $SourceComputer' -ForegroundColor White

Write-Host ""
Write-Host "SRV02 SQL Server setup instructions complete!" -ForegroundColor Green
Write-Host "SQL Service Account: REDTEAM\svc_sql" -ForegroundColor Cyan
Write-Host "SA Password: SQLAdmin123!" -ForegroundColor Cyan
Write-Host "Run C:\SQLSetup.sql in SSMS after SQL installation" -ForegroundColor Cyan

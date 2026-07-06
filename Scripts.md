# Windows Server Enterprise Deployment Scripts

This page contains the core automation scripts used to configure hostnames and Active Directory roles.

## 1. Automated Server Rename Script

Run this script in an elevated PowerShell session to rename a standard member server.

```powershell
# Define the new server name
$NewServerName = "ENT-SRV-01"

# Check current name and rename if different
if ($env:COMPUTERNAME -ne $NewServerName) {
    Write-Host "Renaming server from $env:COMPUTERNAME to $NewServerName..." -ForegroundColor Cyan
    Rename-Computer -NewName $NewServerName -Force -PassThru
    Write-Host "Restarting server in 5 seconds..." -ForegroundColor Yellow
    Start-Sleep -Seconds 5
    Restart-Computer
} else {
    Write-Host "Server is already named $NewServerName." -ForegroundColor Green
}

# 1. Install Active Directory Domain Services Role and Management Tools
Write-Host "Installing Active Directory Domain Services Feature..." -ForegroundColor Cyan
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# 2. Define Domain Configuration
$DomainName = "corp.local"
$NetBIOSName = "CORP"

# 3. Promote Server to Domain Controller
Write-Host "Promoting server to Domain Controller for forest: $DomainName" -ForegroundColor Yellow
Install-ADDSForest -DomainName $DomainName `
                   -CreateDnsDelegation:$false `
                   -DatabasePath "C:\Windows\NTDS" `
                   -LogPath "C:\Windows\NTDS" `
                   -SysVolPath "C:\Windows\SYSVOL" `
                   -NoRebootOnCompletion:$false `
                   -Force:$true

Import-Module ActiveDirectory

$DomainDN    = (Get-ADDomain).DistinguishedName
$BaseOUName  = "Enterprise-Objects"
$BaseOUDN    = "OU=$BaseOUName,$DomainDN"
$Departments = @("IT", "HR", "Sales", "Finance", "Executives")
$Categories  = @("Staff", "Groups", "Workstations", "Servers")

Write-Host "[*] Starting Active Directory baseline provisioning..." -ForegroundColor Cyan

# Create Root Lab OU
if (-not (Get-ADOrganizationalUnit -Filter "Name -eq '$BaseOUName'")) {
    New-ADOrganizationalUnit -Name $BaseOUName -Path $DomainDN -Description "Root container for all lab resources"
}

# Create Core Category OUs
foreach ($Category in $Categories) {
    if (-not (Get-ADOrganizationalUnit -Filter "Name -eq '$Category'" -SearchBase $BaseOUDN -SearchScope OneLevel)) {
        New-ADOrganizationalUnit -Name $Category -Path $BaseOUDN
    }
}

# Create Departmental Sub-OUs (Under Staff)
$StaffRootDN = "OU=Staff,$BaseOUDN"
foreach ($Dept in $Departments) {
    if (-not (Get-ADOrganizationalUnit -Filter "Name -eq '$Dept'" -SearchBase $StaffRootDN -SearchScope OneLevel)) {
        New-ADOrganizationalUnit -Name $Dept -Path $StaffRootDN
    }
}

# Create Functional Security Groups (Under Groups)
$GroupRootDN = "OU=Groups,$BaseOUDN"
foreach ($Dept in $Departments) {
    $GroupName = "SG-$Dept-Users"
    if (-not (Get-ADGroup -Filter "Name -eq '$GroupName'")) {
        New-ADGroup -Name $GroupName -GroupScope Global -GroupCategory Security -Path $GroupRootDN -Description "Standard privileges for $Dept team."
    }
}

# Define Test Data
$TestData = @(
    @{First="Alice";   Last="Smith";     Dept="HR";         Title="HR Manager"}
    @{First="Bob";     Last="Jones";     Dept="IT";         Title="Systems Administrator"}
    @{First="Charlie"; Last="Miller";    Dept="Sales";      Title="Account Executive"}
    @{First="Diana";   Last="Prince";    Dept="Executives"; Title="Chief Executive Officer"}
    @{First="Evan";    Last="Hardened";  Dept="IT";         Title="Security Engineer"}
)

$SecurePassword = ConvertTo-SecureString "Welcome2CorpLab2026!" -AsPlainText -Force

foreach ($User in $TestData) {
    $SAM = ($User.First + "." + $User.Last).ToLower()
    $UPN = "$SAM@" + (Get-ADDomain).DNSRoot
    $TargetOU = "OU=$($User.Dept),$StaffRootDN"
    
    if (-not (Get-ADUser -Filter "SamAccountName -eq '$SAM'")) {
        New-ADUser -Name "$($User.First) $($User.Last)" `
                   -SamAccountName $SAM `
                   -UserPrincipalName $UPN `
                   -GivenName $User.First `
                   -Surname $User.Last `
                   -Title $User.Title `
                   -Department $User.Dept `
                   -Path $TargetOU `
                   -AccountPassword $SecurePassword `
                   -Enabled $true `
                   -ChangePasswordAtLogon $true # Enforces security baseline
        
        Add-ADGroupMember -Identity "SG-$($User.Dept)-Users" -Members $SAM
        Write-Host "[+] Provisioned User and Group Assignment: $SAM" -ForegroundColor Green
    }
}



Write-Host "[*] Executing Endpoint Network Hardening Baseline..." -ForegroundColor Cyan

# 1. Disable LLMNR (Link-Local Multicast Name Resolution) via Registry
$RegistryPath = "HKLM:\Software\Policies\Microsoft\Windows NT\DNSClient"
if (-not (Test-Path $RegistryPath)) {
    New-Item -Path $RegistryPath -Force | Out-Null
}
New-ItemProperty -Path $RegistryPath -Name "EnableMulticast" -Value 0 -PropertyType DWord -Force | Out-Null
Write-Host "[+] LLMNR Mitigation Applied Successfully." -ForegroundColor Green

# 2. Disable NetBIOS over TCP/IP on all active Network Adapters
$Adapters = Get-CimInstance -ClassName Win32_NetworkAdapterConfiguration -Filter "IPEnabled = True"
foreach ($Adapter in $Adapters) {
    # SetTcpipNetbios: 0 = Use DHCP, 1 = Enable, 2 = Disable
    $Adapter.InvokeMethod("SetTcpipNetbios", 2) | Out-Null
    Write-Host "[+] Disabled NetBIOS on Adapter: $($Adapter.Description)" -ForegroundColor Green
}

Write-Host "[*] Endpoint network hardening baseline complete." -ForegroundColor Cyan

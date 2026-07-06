# Windows Server Enterprise Deployment & Hardening Automation

This repository houses the core PowerShell automation scripts engineered to handle the complete lifecycle of the enterprise lab environment—spanning initial system provisioning, Active Directory forest deployment, corporate identity generation, and host-level security baseline hardening.

---

## 🛑 Phase 1: Host Preparation (Server Rename)

Before promoting a server to a core network infrastructure role, this script establishes an enterprise naming convention. It programmatically checks the current hostname and renames the system if a mismatch is detected, preventing execution errors on subsequent runs.

```powershell
<#
.SYNOPSIS
    Automates standard member server renaming conventions.
.DESCRIPTION
    Checks the current host computer name against the target string. 
    Applies the change and initiates a clean reboot if necessary.
#>

# Define the target server name
$NewServerName = "ENT-SRV-01"

# Check current name and rename if different (Idempotent check)
if ($env:COMPUTERNAME -ne $NewServerName) {
    Write-Host "Renaming server from $env:COMPUTERNAME to $NewServerName..." -ForegroundColor Cyan
    Rename-Computer -NewName $NewServerName -Force -PassThru
    Write-Host "Restarting server in 5 seconds to commit changes..." -ForegroundColor Yellow
    Start-Sleep -Seconds




<#
.SYNOPSIS
    Automates AD DS installation and root forest promotion.
.NOTE
    Executing this script will prompt for the Directory Services Restore Mode (DSRM) 
    password. The server will automatically reboot to complete promotion.
#>

# 1. Install Active Directory Domain Services Role and Management Tools
Write-Host "[*] Installing Active Directory Domain Services Feature..." -ForegroundColor Cyan
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# 2. Define Domain Configuration Variables
$DomainName  = "corp.local"
$NetBIOSName = "CORP"

# 3. Promote Server to Forest Root Domain Controller
Write-Host "[*] Promoting server to Domain Controller for forest: $DomainName" -ForegroundColor Yellow
Install-ADDSForest -DomainName $DomainName `
                   -CreateDnsDelegation:$false `
                   -DatabasePath "C:\Windows\NTDS" `
                   -LogPath "C:\Windows\NTDS" `
                   -SysVolPath "C:\Windows\SYSVOL" `
                   -NoRebootOnCompletion:$false `
                   -Force:$true




<#
.SYNOPSIS
    Provisions structural OUs, security groups, and baseline test identities.
.DESCRIPTION
    Enforces clean logic by verifying if containers or users already exist 
    prior to creation routines, ensuring the script can run safely multiple times.
#>

Import-Module ActiveDirectory

# 1. Define Core Configuration & Distinguished Names
$DomainDN    = (Get-ADDomain).DistinguishedName
$BaseOUName  = "Enterprise-Objects"
$BaseOUDN    = "OU=$BaseOUName,$DomainDN"
$Departments = @("IT", "HR", "Sales", "Finance", "Executives")
$Categories  = @("Staff", "Groups", "Workstations", "Servers")

Write-Host "[*] Starting Active Directory baseline provisioning..." -ForegroundColor Cyan

# 2. Create Root Lab OU
if (-not (Get-ADOrganizationalUnit -Filter "Name -eq '$BaseOUName'")) {
    New-ADOrganizationalUnit -Name $BaseOUName -Path $DomainDN -Description "Root container for all lab resources"
    Write-Host "[+] Created Root OU: $BaseOUName" -ForegroundColor Green
}

# 3. Create Core Category OUs
foreach ($Category in $Categories) {
    if (-not (Get-ADOrganizationalUnit -Filter "Name -eq '$Category'" -SearchBase $BaseOUDN -SearchScope OneLevel)) {
        New-ADOrganizationalUnit -Name $Category -Path $BaseOUDN
        Write-Host "[+] Created Category OU: $Category" -ForegroundColor Green
    }
}

# 4. Create Departmental Sub-OUs (Under Staff)
$StaffRootDN = "OU=Staff,$BaseOUDN"
foreach ($Dept in $Departments) {
    if (-not (Get-ADOrganizationalUnit -Filter "Name -eq '$Dept'" -SearchBase $StaffRootDN -SearchScope OneLevel)) {
        New-ADOrganizationalUnit -Name $Dept -Path $StaffRootDN
        Write-Host "[+] Created Departmental Staff OU: $Dept" -ForegroundColor Green
    }
}

# 5. Create Functional Security Groups (Under Groups)
$GroupRootDN = "OU=Groups,$BaseOUDN"
foreach ($Dept in $Departments) {
    $GroupName = "SG-$Dept-Users"
    if (-not (Get-ADGroup -Filter "Name -eq '$GroupName'")) {
        New-ADGroup -Name $GroupName -GroupScope Global -GroupCategory Security -Path $GroupRootDN -Description "Standard privileges for the $Dept team."
        Write-Host "[+] Created Security Group: $GroupName" -ForegroundColor Green
    }
}

# 6. Define and Provision Standard Corporate Users
$TestData = @(
    @{First="Alice";   Last="Smith";     Dept="HR";         Title="HR Manager"}
    @{First="Bob";     Last="Jones";     Dept="IT";         Title="Systems Administrator"}
    @{First="Charlie"; Last="Miller";    Dept="Sales";      Title="Account Executive"}
    @{First="Diana";   Last="Prince";    Dept="Executives"; Title="Chief Executive Officer"}
    @{First="Evan";    Last="Hardened";  Dept="IT";         Title="Security Engineer"}
    @{First="Fiona";   Last="Gallagher"; Dept="Finance";    Title="Financial Analyst"}
    @{First="George";  Last="Clark";     Dept="Sales";      Title="Sales Representative"}
)

# Establish a secure temporary baseline password
$SecurePassword = ConvertTo-SecureString "Welcome2CorpLab2026!" -AsPlainText -Force

foreach ($User in $TestData) {
    $SAM = ($User.First + "." + $User.Last).ToLower()
    $UPN = "$SAM@" + (Get-ADDomain).DNSRoot
    $TargetOU = "OU=$($User.Dept),$StaffRootDN"
    
    if (-not (Get-ADUser -Filter "SamAccountName -eq '$SAM'")) {
        # Provision the user with strict corporate attributes
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
                   -ChangePasswordAtLogon $true # Enforces secure identity onboarding
        
        Write-Host "[+] Provisioned User: $SAM ($($User.Title))" -ForegroundColor Green
        
        # Automatically map user to their departmental group
        $TargetGroup = "SG-$($User.Dept)-Users"
        Add-ADGroupMember -Identity $TargetGroup -Members $SAM
        Write-Host "    [->] Added $SAM to $TargetGroup" -ForegroundColor Yellow
    } else {
        Write-Host "[-] User $SAM already exists. Skipping creation." -ForegroundColor Yellow
    }
}

Write-Host "[*] Active Directory object population complete." -ForegroundColor Cyan

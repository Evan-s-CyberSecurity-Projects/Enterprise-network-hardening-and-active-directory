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

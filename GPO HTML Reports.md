# Group Policy Object (GPO) HTML Reports

This directory stores the configuration auditing reports exported directly from the Active Directory Domain Controllers.

## Active GPO Baseline Reports

* 📄 **[Default Domain Policy](./reports/Default_Domain_Policy.html)**  
  *Enforces global enterprise account lockouts, password lengths, and Kerberos rules.*
* 📄 **[Server Hardening Baseline](./reports/Server_Hardening_Baseline.html)**  
  *Configures advanced Windows Firewall settings, restricts local administrative access, and disables legacy TLS.*
* 📄 **[Workstation Desktop Restrictions](./reports/Workstation_Desktop_Baseline.html)**  
  *Deploys enterprise wallpaper, maps network drives, and disables USB storage keys.*

## Configuration Auditing Instructions
To refresh these files in this repository when a policy changes:
1. Open PowerShell as an Administrator on `DC-NEW`.
2. Execute: `Get-GPOReport -Name "Your-GPO-Name" -ReportType Html -Path ".\reports\Your-GPO-Name.html"`
3. Commit and push the updated `.html` file to the GitHub main branch.


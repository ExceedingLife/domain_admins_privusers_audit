#Import-Module ActiveDirectory

# Add/remove groups as needed
$AdminGroups = @(
    "Domain Admins",
    "Enterprise Admins",
    "Schema Admins",
    "Administrators",
    "Account Operators",
    "Server Operators",
    "Backup Operators",
    "Print Operators",
    "Group Policy Creator Owners",
    "Protected Users",
    "DnsAdmins"
)

$Results = @()

foreach ($Group in $AdminGroups) {

    Write-Host "Checking $Group..." -ForegroundColor Cyan

    try {
        $Members = Get-ADGroupMember -Identity $Group -Recursive -ErrorAction Stop

        foreach ($Member in $Members) {
            try {
                if ($Member.objectClass -eq "user") {
                    $User = Get-ADUser $Member.SamAccountName -Properties `
                    Enabled,
                    Created,
                    PasswordLastSet,
                    PasswordNeverExpires,
                    LastLogonDate,
                    AccountExpirationDate,
                    LockedOut,
                    AdminCount,
                    DistinguishedName

                    $OU = ($User.DistinguishedName -replace '^CN=.*?,','')

                    $Results += [PSCustomObject]@{
                        Group                  = $Group
                        Name                   = $User.Name
                        Username               = $User.SamAccountName
                        ObjectType             = "User"
                        Enabled                = $User.Enabled
                        Created                = $User.Created
                        PasswordLastSet        = $User.PasswordLastSet
                        PasswordNeverExpires   = $User.PasswordNeverExpires
                        LastLogon              = $User.LastLogonDate
                        AccountExpires         = $User.AccountExpirationDate
                        LockedOut              = $User.LockedOut
                        AdminCount             = $User.AdminCount
                        OU                     = $OU
                    }
                }

                elseif ($Member.objectClass -eq "group") {

                    $GroupInfo = Get-ADGroup $Member.SamAccountName -Properties Created, DistinguishedName

                    $OU = ($GroupInfo.DistinguishedName -replace '^CN=.*?,','')

                    $Results += [PSCustomObject]@{
                        Group                  = $Group
                        Name                   = $GroupInfo.Name
                        Username               = $GroupInfo.SamAccountName
                        ObjectType             = "Group"
                        Enabled                = "N/A"
                        Created                = $GroupInfo.Created
                        PasswordLastSet        = "N/A"
                        PasswordNeverExpires   = "N/A"
                        LastLogon              = "N/A"
                        AccountExpires         = "N/A"
                        LockedOut              = "N/A"
                        AdminCount             = "N/A"
                        OU                     = $OU
                    }
                }

            }
            catch {
                Write-Warning "Failed processing member: $($Member.Name)"
            }
        }

    }
    catch {
        Write-Warning "$Group not found"
    }
}

$Results = $Results | Sort-Object Group,Name

$Results | Format-Table -AutoSize

# Optional CSV export - remove # below if you want .csv output
#$Results | Export-Csv ".\AD_Admin_Overview.csv" -NoTypeInformation

Write-Host ""
Write-Host "Exported to AD_Admin_Overview.csv" -ForegroundColor Green
tere!
cd C:\ADProjekt
.\Loo-DomeeniKasutajad.ps1


Import-Module ActiveDirectory

$Domain = "DC=kehtna,DC=com"
$UsersOU = "OU=Kasutajad,$Domain"
$GroupsOU = "OU=Grupid,$Domain"
$CsvPath = Join-Path $PSScriptRoot "nimekiri.csv"
$Password = ConvertTo-SecureString "Koolitoo2026!" -AsPlainText -Force

$u = [char]0x00FC

$OUList = @(
    "Juhtkond",
    "IT",
    ("M" + $u + $u + "k"),
    "Toode",
    "Turundus",
    "Personal",
    "Administratsioon",
    "Haldus",
    "Finants",
    "Disain",
    ("Anal" + $u + $u + "tika"),
    "Tehnika",
    "Juriidika",
    "Projektid",
    "Muu"
)

if (-not (Get-ADOrganizationalUnit -Identity $UsersOU -ErrorAction SilentlyContinue)) {
    New-ADOrganizationalUnit -Name "Kasutajad" -Path $Domain -ProtectedFromAccidentalDeletion $false
}

if (-not (Get-ADOrganizationalUnit -Identity $GroupsOU -ErrorAction SilentlyContinue)) {
    New-ADOrganizationalUnit -Name "Grupid" -Path $Domain -ProtectedFromAccidentalDeletion $false
}

foreach ($OUName in $OUList) {
    $OUPath = "OU=$OUName,$UsersOU"

    if (-not (Get-ADOrganizationalUnit -Identity $OUPath -ErrorAction SilentlyContinue)) {
        New-ADOrganizationalUnit -Name $OUName -Path $UsersOU -ProtectedFromAccidentalDeletion $false
    }
}

if (-not (Test-Path $CsvPath)) {
    Write-Host "CSV faili ei leitud: $CsvPath" -ForegroundColor Red
    exit
}

$Users = Import-Csv $CsvPath

function Convert-ToSamAccountName {
    param (
        [string]$Name
    )

    $Result = $Name.ToLower()

    $Result = $Result.Replace("ü", "u")
    $Result = $Result.Replace("ä", "a")
    $Result = $Result.Replace("ö", "o")
    $Result = $Result.Replace("õ", "o")

    $Result = $Result -replace '[^a-z0-9 ]', ''
    $Result = $Result.Trim()
    $Result = $Result -replace '\s+', '.'

    return $Result
}

foreach ($User in $Users) {

    $FullName = $User.Name
    $City = $User.City
    $Job = $User.Job

    $NameParts = $FullName -split '\s+'
    $GivenName = $NameParts[0]
    $Surname = $NameParts[-1]

    $SamAccountName = Convert-ToSamAccountName $FullName

    $ExistingUser = Get-ADUser -Filter "SamAccountName -eq '$SamAccountName'" -ErrorAction SilentlyContinue

    switch -Regex ($Job) {

        "^(CEO|COO|CTO)$" {
            $DepartmentOU = "Juhtkond"
            break
        }

        "IT manager|IT Support|Software Engineer|Software Developer|Web Developer|Web Engineer|Database Developer" {
            $DepartmentOU = "IT"
            break
        }

        "Sales$|Sales Executive|Sales Support" {
            $DepartmentOU = "M" + $u + $u + "k"
            break
        }

        "Product Manager|Cleaning Manager" {
            $DepartmentOU = "Toode"
            break
        }

        "Marketing|Marketing Manager" {
            $DepartmentOU = "Turundus"
            break
        }

        "HR Specialist" {
            $DepartmentOU = "Personal"
            break
        }

        "Administrator" {
            $DepartmentOU = "Administratsioon"
            break
        }

        "Accountant|Financial Advisor" {
            $DepartmentOU = "Finants"
            break
        }

        "Graphic Artist|Graphic Designer" {
            $DepartmentOU = "Disain"
            break
        }

        "Data Analyst|Research Scientist|Business Analyst" {
            $DepartmentOU = "Anal" + $u + $u + "tika"
            break
        }

        "Architect|Mechanical Engineer" {
            $DepartmentOU = "Tehnika"
            break
        }

        "Lawyer" {
            $DepartmentOU = "Juriidika"
            break
        }

        "Project Manager|Event Planner" {
            $DepartmentOU = "Projektid"
            break
        }

        default {
            $DepartmentOU = "Muu"
        }
    }

    $TargetOU = "OU=$DepartmentOU,$UsersOU"

    if ($ExistingUser) {

        Set-ADUser -Identity $ExistingUser `
            -City $City `
            -Title $Job `
            -GivenName $GivenName `
            -Surname $Surname `
            -DisplayName $FullName

        Move-ADObject -Identity $ExistingUser.DistinguishedName -TargetPath $TargetOU

    }
    else {

        New-ADUser -Name $FullName `
            -GivenName $GivenName `
            -Surname $Surname `
            -DisplayName $FullName `
            -SamAccountName $SamAccountName `
            -UserPrincipalName "$SamAccountName@kehtna.com" `
            -City $City `
            -Title $Job `
            -Path $TargetOU `
            -AccountPassword $Password `
            -Enabled $true `
            -ChangePasswordAtLogon $false
    }

    $GroupName = "GRP-" + ($Job -replace '[^a-zA-Z0-9]+','-').Trim('-')

    $ExistingGroup = Get-ADGroup -SearchBase $GroupsOU -Filter "Name -eq '$GroupName'" -ErrorAction SilentlyContinue

    if (-not $ExistingGroup) {

        New-ADGroup -Name $GroupName `
            -SamAccountName $GroupName `
            -GroupCategory Security `
            -GroupScope Global `
            -Path $GroupsOU
    }

    $ADUser = Get-ADUser -Identity $SamAccountName

    $IsMember = Get-ADGroupMember -Identity $GroupName -Recursive -ErrorAction SilentlyContinue |
        Where-Object { $_.SamAccountName -eq $SamAccountName }

    if (-not $IsMember) {
        Add-ADGroupMember -Identity $GroupName -Members $ADUser
    }
}

Write-Host "Kasutajate ja gruppide loomine lõpetatud!" -ForegroundColor Green

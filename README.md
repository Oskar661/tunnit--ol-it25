tere!
cd C:\ADProjekt
.\Loo-DomeeniKasutajad.ps1







Import-Module ActiveDirectory

# ==============================
# SEADISTUSED
# ==============================

$Domain = "DC=kehtna,DC=com"
$UsersOU = "OU=Kasutajad,$Domain"
$GroupsOU = "OU=Grupid,$Domain"

$CsvPath = Join-Path $PSScriptRoot "nimekiri.csv"

$Password = ConvertTo-SecureString "Koolitoo2026!" -AsPlainText -Force

# Eesti täpitähed
$u = [char]0x00FC   # ü
$U = [char]0x00DC   # Ü
$a = [char]0x00E4   # ä
$A = [char]0x00C4   # Ä
$o = [char]0x00F6   # ö
$O = [char]0x00D6   # Ö
$e = [char]0x00F5   # õ
$E = [char]0x00D5   # Õ

# ==============================
# OU-D
# ==============================

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

# Kasutajate OU
if (-not (Get-ADOrganizationalUnit -Identity $UsersOU -ErrorAction SilentlyContinue) {
    New-ADOrganizationalUnit `
        -Name "Kasutajad" `
        -Path $Domain `
        -ProtectedFromAccidentalDeletion $false
}

# Gruppide OU
if (-not (Get-ADOrganizationalUnit -Identity $GroupsOU -ErrorAction SilentlyContinue)) {
    New-ADOrganizationalUnit `
        -Name "Grupid" `
        -Path $Domain `
        -ProtectedFromAccidentalDeletion $false
}

# Alam-OU-d
foreach ($OUName in $OUList) {

    $OUPath = "OU=$OUName,$UsersOU"

    if (-not (Get-ADOrganizationalUnit -Identity $OUPath -ErrorAction SilentlyContinue)) {

        New-ADOrganizationalUnit `
            -Name $OUName `
            -Path $UsersOU `
            -ProtectedFromAccidentalDeletion $false

        Write-Host "OU loodud: $OUName" -ForegroundColor Green
    }
}

# ==============================
# CSV
# ==============================

if (-not (Test-Path $CsvPath)) {
    Write-Host "CSV faili ei leitud: $CsvPath" -ForegroundColor Red
    exit
}

$Users = Import-Csv $CsvPath

# ==============================
# FUNKTSIOON SAMACCOUNTNAME'I JAOKS
# ==============================

function Convert-ToSamAccountName {
    param (
        [string]$Name
    )

    $Result = $Name.ToLower()

    $Result = $Result.Replace($u, "u")
    $Result = $Result.Replace($U.ToString().ToLower(), "u")

    $Result = $Result.Replace($a, "a")
    $Result = $Result.Replace($A.ToString().ToLower(), "a")

    $Result = $Result.Replace($o, "o")
    $Result = $Result.Replace($O.ToString().ToLower(), "o")

    $Result = $Result.Replace($e, "o")
    $Result = $Result.Replace($E.ToString().ToLower(), "o")

    $Result = $Result -replace '[^a-z0-9 ]', ''
    $Result = $Result.Trim()
    $Result = $Result -replace '\s+', '.'

    return $Result
}

# ==============================
# KASUTAJATE LOOMINE
# ==============================

foreach ($User in $Users) {

    $FullName = $User.Name
    $City = $User.City
    $Job = $User.Job

    # Ees- ja perekonnanimi
    $NameParts = $FullName -split '\s+'

    $GivenName = $NameParts[0]
    $Surname = $NameParts[-1]

    $SamAccountName = Convert-ToSamAccountName $FullName

    # Otsi olemasolev kasutaja
    $ExistingUser = Get-ADUser `
        -Filter "SamAccountName -eq '$SamAccountName'" `
        -ErrorAction SilentlyContinue

    # ==========================
    # MÄÄRA OU AMETI JÄRGI
    # ==========================

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
            $DepartmentOU = ("M" + $u + $u + "k")
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
            $DepartmentOU = ("Anal" + $u + $u + "tika")
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

    # ==========================
    # KASUTAJA LOOMINE / UUENDAMINE
    # ==========================

    if ($ExistingUser) {

        Set-ADUser `
            -Identity $ExistingUser `
            -City $City `
            -Title $Job `
            -GivenName $GivenName `
            -Surname $Surname `
            -DisplayName $FullName

        Move-ADObject `
            -Identity $ExistingUser.DistinguishedName `
            -TargetPath $TargetOU

        Write-Host "Uuendatud: $FullName" -ForegroundColor Yellow
    }
    else {

        New-ADUser `
            -Name $FullName `
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

        Write-Host "Loodud: $FullName -> $DepartmentOU" -ForegroundColor Green
    }

    # ==========================
    # AMETIGRUPP
    # ==========================

    $GroupName = "GRP-" + ($Job -replace '[^a-zA-Z0-9]+','-').Trim('-')

    $ExistingGroup = Get-ADGroup `
        -SearchBase $GroupsOU `
        -Filter "Name -eq '$GroupName'" `
        -ErrorAction SilentlyContinue

    if (-not $ExistingGroup) {

        New-ADGroup `
            -Name $GroupName `
            -SamAccountName $GroupName `
            -GroupCategory Security `
            -GroupScope Global `
            -Path $GroupsOU

        Write-Host "Grupp loodud: $GroupName" -ForegroundColor Cyan
    }

    # Kasutaja gruppi
    $ADUser = Get-ADUser -Identity $SamAccountName

    $IsMember = Get-ADGroupMember `
        -Identity $GroupName `
        -Recursive |
        Where-Object { $_.SamAccountName -eq $SamAccountName }

    if (-not $IsMember) {

        Add-ADGroupMember `
            -Identity $GroupName `
            -Members $ADUser

        Write-Host "Grupiliige: $FullName -> $GroupName" -ForegroundColor Cyan
    }
}

Write-Host ""
Write-Host "========================================" -ForegroundColor Green
Write-Host "AD kasutajate loomine on lõpetatud!" -ForegroundColor Green
Write-Host "========================================" -ForegroundColor Green

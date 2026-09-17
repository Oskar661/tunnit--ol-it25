cd C:\ADProjekt

Set-ExecutionPolicy -Scope Process Bypass

käivitada käsuga:
.\Loo-DomeeniKasutajad.ps1

Domeen: kehtna.com










#requires -Modules ActiveDirectory

Import-Module ActiveDirectory

$DomainDN = "DC=kehtna,DC=com"
$CsvPath = Join-Path $PSScriptRoot "nimekiri.csv"

$PasswordPlain = "Koolitoo2026!"
$Password = ConvertTo-SecureString $PasswordPlain -AsPlainText -Force

Write-Host ""
Write-Host "============================================"
Write-Host "ACTIVE DIRECTORY KASUTAJATE LOOMINE"
Write-Host "============================================"
Write-Host ""

# Kontrollime CSV-d
if (-not (Test-Path $CsvPath)) {
    Write-Host "VIGA: nimekiri.csv ei leitud!" -ForegroundColor Red
    exit
}

$Users = Import-Csv $CsvPath

Write-Host "Domeen: kehtna.com"
Write-Host "Kasutajaid CSV-s: $($Users.Count)"
Write-Host ""

# --------------------------------------------------
# 1. KASUTAJATE PÕHI-OU
# --------------------------------------------------

$UsersOU = Get-ADOrganizationalUnit `
    -Filter "Name -eq 'Kasutajad'" `
    -SearchBase $DomainDN

if (-not $UsersOU) {
    New-ADOrganizationalUnit `
        -Name "Kasutajad" `
        -Path $DomainDN

    $UsersOU = Get-ADOrganizationalUnit `
        -Filter "Name -eq 'Kasutajad'" `
        -SearchBase $DomainDN
}

Write-Host "Kasutajate OU: $($UsersOU.DistinguishedName)"

# --------------------------------------------------
# 2. GRUPPIDE OU
# --------------------------------------------------

$GroupsOU = Get-ADOrganizationalUnit `
    -Filter "Name -eq 'Grupid'" `
    -SearchBase $DomainDN

if (-not $GroupsOU) {
    New-ADOrganizationalUnit `
        -Name "Grupid" `
        -Path $DomainDN

    $GroupsOU = Get-ADOrganizationalUnit `
        -Filter "Name -eq 'Grupid'" `
        -SearchBase $DomainDN
}

Write-Host "Gruppide OU: $($GroupsOU.DistinguishedName)"
Write-Host ""

# --------------------------------------------------
# 3. OLEMASOLEVATE OU-DE LEIDMINE
# --------------------------------------------------

$AllUserOUs = Get-ADOrganizationalUnit `
    -SearchBase $UsersOU.DistinguishedName `
    -SearchScope OneLevel `
    -Filter *

Write-Host "Kasutajate OU-d kontrollitud."

# --------------------------------------------------
# 4. AMET -> OU
# --------------------------------------------------

$JobToOU = @{
    "CEO"                 = "Juhtkond"
    "COO"                 = "Juhtkond"

    "CTO"                 = "IT"
    "IT manager"          = "IT"
    "IT Support"          = "IT"
    "Software Engineer"   = "IT"
    "Software Developer"  = "IT"
    "Web Developer"       = "IT"
    "Web Engineer"        = "IT"
    "Database Developer"  = "IT"

    "Sales"               = "Müük"
    "Sales Executive"     = "Müük"
    "Sales Support"       = "Müük"

    "Product Manager"     = "Toode"

    "Marketing"           = "Turundus"
    "Marketing Manager"   = "Turundus"

    "HR Specialist"       = "Personal"

    "Administrator"       = "Administratsioon"

    "Cleaning Manager"    = "Haldus"

    "Financial Advisor"   = "Finants"
    "Accountant"          = "Finants"

    "Graphic Designer"    = "Disain"
    "Graphic Artist"      = "Disain"

    "Data Analyst"        = "Analüütika"
    "Research Scientist"  = "Analüütika"
    "Business Analyst"    = "Analüütika"

    "Architect"           = "Tehnika"
    "Mechanical Engineer" = "Tehnika"

    "Lawyer"              = "Juriidika"

    "Project Manager"     = "Projektid"

    "Journalist"          = "Muu"
    "Event Planner"       = "Muu"
    "Social Worker"       = "Muu"
}

# --------------------------------------------------
# 5. LOOME PUUDUVAD OUD
# --------------------------------------------------

$RequiredOUs = @(
    "Juhtkond",
    "IT",
    "Müük",
    "Toode",
    "Turundus",
    "Personal",
    "Administratsioon",
    "Haldus",
    "Finants",
    "Disain",
    "Analüütika",
    "Tehnika",
    "Juriidika",
    "Projektid",
    "Muu"
)

foreach ($OUName in $RequiredOUs) {

    $ExistingOU = Get-ADOrganizationalUnit `
        -SearchBase $UsersOU.DistinguishedName `
        -SearchScope OneLevel `
        -Filter "Name -eq '$OUName'" `
        -ErrorAction SilentlyContinue

    if (-not $ExistingOU) {

        New-ADOrganizationalUnit `
            -Name $OUName `
            -Path $UsersOU.DistinguishedName

        Write-Host "OU loodud: $OUName"
    }
}

Write-Host ""
Write-Host "Kõik vajalikud OUid on olemas."
Write-Host ""

# --------------------------------------------------
# 6. DIACRITICS -> KASUTAJANIMI
# --------------------------------------------------

function Remove-Diacritics {
    param(
        [string]$Text
    )

    $normalized = $Text.Normalize(
        [System.Text.NormalizationForm]::FormD
    )

    $result = ""

    foreach ($char in $normalized.ToCharArray()) {

        if (
            [Globalization.CharUnicodeInfo]::GetUnicodeCategory($char) `
            -ne [Globalization.UnicodeCategory]::NonSpacingMark
        ) {
            $result += $char
        }
    }

    return $result.Normalize(
        [System.Text.NormalizationForm]::FormC
    )
}

# --------------------------------------------------
# 7. KASUTAJATE LOOMINE
# --------------------------------------------------

$NewUsers = 0
$UpdatedUsers = 0

foreach ($Person in $Users) {

    $FullName = $Person.Name.Trim()
    $City = $Person.City.Trim()
    $Job = $Person.Job.Trim()

    # Leiame OU
    $TargetOUName = $JobToOU[$Job]

    if (-not $TargetOUName) {
        $TargetOUName = "Muu"
    }

    # Leiame OU täpselt nime järgi
    $TargetOU = Get-ADOrganizationalUnit `
        -SearchBase $UsersOU.DistinguishedName `
        -SearchScope OneLevel `
        -Filter "Name -eq '$TargetOUName'" `
        -ErrorAction Stop

    # Ees- ja perekonnanimi
    $NameParts = $FullName -split " "

    $FirstName = $NameParts[0]
    $LastName = $NameParts[-1]

    # Kasutajanimi
    $FirstClean = Remove-Diacritics $FirstName
    $LastClean = Remove-Diacritics $LastName

    $Sam = "$($FirstClean.ToLower()).$($LastClean.ToLower())"

    if ($Sam.Length -gt 20) {
        $Sam = $Sam.Substring(0,20)
    }

    # Kontrollime olemasolevat kasutajat
    $ExistingUser = Get-ADUser `
        -Filter "SamAccountName -eq '$Sam'" `
        -ErrorAction SilentlyContinue

    if ($ExistingUser) {

        Set-ADUser `
            -Identity $ExistingUser `
            -City $City `
            -Title $Job `
            -GivenName $FirstName `
            -Surname $LastName `
            -DisplayName $FullName

        # Liigutame õigesse OU-sse
        Move-ADObject `
            -Identity $ExistingUser.DistinguishedName `
            -TargetPath $TargetOU.DistinguishedName

        Write-Host "UUENDATUD: $FullName"
        Write-Host "   Kasutajanimi: $Sam"
        Write-Host "   Amet: $Job"
        Write-Host "   Linn: $City"
        Write-Host "   OU: $TargetOUName"
        Write-Host ""

        $UpdatedUsers++
    }
    else {

        New-ADUser `
            -Name $FullName `
            -GivenName $FirstName `
            -Surname $LastName `
            -DisplayName $FullName `
            -SamAccountName $Sam `
            -UserPrincipalName "$Sam@kehtna.com" `
            -AccountPassword $Password `
            -Enabled $true `
            -ChangePasswordAtLogon $false `
            -City $City `
            -Title $Job `
            -Path $TargetOU.DistinguishedName

        Write-Host "LOODUD: $FullName"
        Write-Host "   Kasutajanimi: $Sam"
        Write-Host "   Amet: $Job"
        Write-Host "   Linn: $City"
        Write-Host "   OU: $TargetOUName"
        Write-Host ""

        $NewUsers++
    }
}

# --------------------------------------------------
# 8. AMETITE GRUPID
# --------------------------------------------------

Write-Host ""
Write-Host "============================================"
Write-Host "AMETITE GRUPPIDE LOOMINE"
Write-Host "============================================"
Write-Host ""

$Jobs = $Users | Select-Object -ExpandProperty Job -Unique

foreach ($Job in $Jobs) {

    $GroupName = "GRP-" + ($Job -replace "[^a-zA-Z0-9-]", "-")

    $ExistingGroup = Get-ADGroup `
        -Filter "Name -eq '$GroupName'" `
        -SearchBase $GroupsOU.DistinguishedName `
        -ErrorAction SilentlyContinue

    if (-not $ExistingGroup) {

        New-ADGroup `
            -Name $GroupName `
            -SamAccountName $GroupName `
            -GroupCategory Security `
            -GroupScope Global `
            -Path $GroupsOU.DistinguishedName

        $ExistingGroup = Get-ADGroup `
            -Identity $GroupName
    }

    Write-Host "GRUPP: $GroupName"

    $JobUsers = $Users | Where-Object {
        $_.Job -eq $Job
    }

    foreach ($Person in $JobUsers) {

        $CleanFirst = Remove-Diacritics $Person.Name.Split(" ")[0]
        $CleanLast = Remove-Diacritics $Person.Name.Split(" ")[-1]

        $Sam = "$($CleanFirst.ToLower()).$($CleanLast.ToLower())"

        if ($Sam.Length -gt 20) {
            $Sam = $Sam.Substring(0,20)
        }

        $ADUser = Get-ADUser `
            -Filter "SamAccountName -eq '$Sam'" `
            -ErrorAction SilentlyContinue

        if ($ADUser) {

            $AlreadyMember = Get-ADGroupMember `
                -Identity $ExistingGroup `
                -Recursive `
                -ErrorAction SilentlyContinue |
                Where-Object {
                    $_.SamAccountName -eq $Sam
                }

            if (-not $AlreadyMember) {

                Add-ADGroupMember `
                    -Identity $ExistingGroup `
                    -Members $ADUser

                Write-Host "   + $($Person.Name)"
            }
        }
    }

    Write-Host ""
}

# --------------------------------------------------
# 9. VALMIS
# --------------------------------------------------

Write-Host ""
Write-Host "============================================"
Write-Host "VALMIS!"
Write-Host "============================================"
Write-Host ""

Write-Host "Domeen: kehtna.com"
Write-Host "CSV kasutajaid: $($Users.Count)"
Write-Host "Uusi kasutajaid: $NewUsers"
Write-Host "Uuendatud kasutajaid: $UpdatedUsers"
Write-Host ""

Write-Host "Kõik kasutajad on loodud/uuendatud."
Write-Host "Kõik ametite grupid on loodud."
Write-Host "Asukohad on City väljal."
Write-Host "Ametid on Job Title väljal."
Write-Host ""
Write-Host "Algne parool kõigile kasutajatele:"
Write-Host "Koolitoo2026!"
Write-Host ""

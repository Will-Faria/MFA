# Conectar ao Microsoft Graph com permissões necessárias
Connect-MgGraph -Scopes "User.Read.All", "UserAuthenticationMethod.Read.All", "Directory.Read.All"

# SMTP Config
$smtpServer = "xxxxxxxxxxxxxx"
$smtpFrom = "xxxxxxxxxxxxxx"
$smtpTo = "xxxxxxxxxxxxxx"
$smtpSubject = "🔒 Relatório: Usuários com MFA Desabilitado"

# Mapeamento de códigos SKU
$skuMap = @{
    "ENTERPRISEPACK" = "Office 365 E3"
    "EMS" = "Enterprise Mobility + Security E3"
    "EMSPREMIUM" = "Enterprise Mobility + Security E5"
    "E5" = "Office 365 E5"
    "BUSINESS_PREMIUM" = "Microsoft 365 Business Premium"
    "M365BUSINESS" = "Microsoft 365 Business Standard"
    "STANDARDPACK" = "Office 365 E1"
    "DESKLESSPACK" = "Microsoft 365 F1"
    "M365F1" = "Microsoft 365 F1"
    "M365F3" = "Microsoft 365 F3"
    "PROJECTPROFESSIONAL" = "Project Plan 3"
    "VISIOONLINEPLAN2" = "Visio Plan 2"
    "POWER_BI_PRO" = "Power BI Pro"
    "DYN365_ENTERPRISE_PLAN1" = "Dynamics 365 Enterprise Plan 1"
}

# Coleta os usuários
$Users = Get-MgUser -All -Property "Id", "DisplayName", "UserPrincipalName", "UserType", "Mail", "ProxyAddresses", "AccountEnabled", "CreatedDateTime"

# Inicializa lista
$UsuariosFiltrados = @()

foreach ($User in $Users) {
    # Ignora usuários que não são do domínio alvo
    if ($User.UserPrincipalName -notlike "*@xxxxxxxxxxxxxx" -or $User.UserPrincipalName -eq "imap@xxxxxxxxxxxxxx") {
        continue
    }

    try {
        $MFAState = Invoke-MgGraphRequest -Uri "https://graph.microsoft.com/beta/users/$($User.Id)/authentication/requirements" -Method GET
        $SignInPrefs = Invoke-MgGraphRequest -Uri "https://graph.microsoft.com/beta/users/$($User.Id)/authentication/signInPreferences" -Method GET

        $mfaMethod = $SignInPrefs.userPreferredMethodForSecondaryAuthentication
        if (-not $mfaMethod) {
            $mfaMethod = "Não habilitado"
        }

        if ($MFAState.PerUserMfaState -eq "disabled") {
            # Licenças
            $licenses = @()
            try {
                $LicenseDetails = Get-MgUserLicenseDetail -UserId $User.Id
                foreach ($detail in $LicenseDetails) {
                    $skuName = $detail.SkuPartNumber
                    $licenses += if ($skuMap.ContainsKey($skuName)) { $skuMap[$skuName] } else { $skuName }
                }
            } catch {
                $licenses = @()
            }

            if ($licenses.Count -gt 0) {
                $UsuariosFiltrados += [PSCustomObject]@{
                    DisplayName = $User.DisplayName
                    UserPrincipalName = $User.UserPrincipalName.Replace("#EXT#", "")
                    MFAState = $MFAState.PerUserMfaState
                    Licenses = ($licenses -join "; ")
                }
            }
        }
    } catch {
        continue
    }
}

# Gera o HTML do email
if ($UsuariosFiltrados.Count -eq 0) {
    $htmlBody = "<p>Todos os usuários com domínio @xxxxxxxxxxxxxx estão com MFA ativado.</p>"
} else {
    $htmlBody = @"
<html>
<head>
    <style>
        table { border-collapse: collapse; width: 100%; font-family: Arial; font-size: 14px; }
        th, td { border: 1px solid #ccc; padding: 8px; text-align: left; }
        thead { background-color: #004080; color: white; }
        tr:nth-child(even) { background-color: #f9f9f9; }
        .disabled { color: red; font-weight: bold; }
    </style>
</head>
<body>
    <p>Olá,</p>
    <p>Segue abaixo a lista de usuários com <strong>MFA desabilitado</strong> no domínio <em>@xxxxxxxxxxxxxx</em>:</p>
    <table>
        <thead>
            <tr><th>Nome</th><th>UPN</th><th>Status</th><th>Licenças</th></tr>
        </thead>
        <tbody>
"@

    foreach ($usuario in $UsuariosFiltrados) {
        $htmlBody += "<tr>
            <td>$($usuario.DisplayName)</td>
            <td>$($usuario.UserPrincipalName)</td>
            <td class='disabled'>$($usuario.MFAState)</td>
            <td>$($usuario.Licenses)</td>
        </tr>"
    }

    $htmlBody += @"
        </tbody>
    </table>
    <p>Por favor, verifique a necessidade de ativação do MFA para esses usuários.</p>
    <p>Att,<br>Equipe de TI</p>
</body>
</html>
"@
}

# Envia o e-mail
try {
    $smtp = New-Object System.Net.Mail.SmtpClient($smtpServer, 25)
    $mail = New-Object System.Net.Mail.MailMessage
    $mail.From = $smtpFrom
    $smtpTo.Split(",") | ForEach-Object { $mail.To.Add($_) }
    $mail.Subject = $smtpSubject
    $mail.IsBodyHtml = $true
    $mail.Body = $htmlBody

    $smtp.Send($mail)
    Write-Host "[Email] Relatório enviado com sucesso!" -ForegroundColor Green
} catch {
    Write-Host "[Erro] Falha ao enviar e-mail: $_" -ForegroundColor Red
}

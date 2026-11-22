# Install Argus in your own Azure

Argus 0.2.28. This guide takes you from an empty Azure subscription to Argus's sign-in page.

## 1. What it builds, and what it costs

One deployment puts everything in one resource group in your own subscription:

- a container app that runs Argus, reachable only from the addresses you give, and asleep when nobody uses it;
- a nightly job that collects from Microsoft 365, read-only;
- a 32 GiB file share for Argus's data, reachable only from Argus's own network;
- a key vault for the keys you save in Argus's settings;
- a managed identity that Argus runs as, with no password or certificate to look after;
- a virtual network that ties them together.

Azure bills you for these directly. Most of the cost is the file share and the container app; the key vault and the
network cost little or nothing. Prices depend on your region and agreement, so check them in Microsoft's
[pricing calculator](https://azure.microsoft.com/pricing/calculator/), with
[Container Apps](https://azure.microsoft.com/pricing/details/container-apps/),
[Azure Files](https://azure.microsoft.com/pricing/details/storage/files/) (Premium, NFS) and
[Key Vault](https://azure.microsoft.com/pricing/details/key-vault/). Argus's licence is separate.

## 2. Before you start

| You need | How to check |
|---|---|
| The pull credential for Argus's image: a user name and a password. | Higate sends it before the install. Keep the password in your password manager. |
| **Owner** on the resource group Argus goes in, or **Contributor** plus **Role Based Access Control Administrator**. | Azure portal > the resource group > **Access control (IAM)** > **Check access** > **View my access**. |
| Room for one Container Apps environment in the region. | Step 3 tells you. If it stops with `MaxNumberOfGlobalEnvironmentsInSubExceeded`, ask for more in the portal: **Quotas** > Azure Container Apps > **Managed environment count**, for that region ([how](https://learn.microsoft.com/azure/container-apps/quota-requests)). |
| An Entra admin for step 5: a Global Administrator, or a Privileged Role Administrator who is also an Application Administrator and a Groups Administrator, with Contributor on the resource group. | Entra admin center > **Roles and administrators** > the role > **Assignments** lists them as **Active**. |
| Microsoft Entra ID P1 or P2 in the tenant. | Entra admin center > **Overview** > **License**. |
| Your office's public IPv4 addresses. Nothing else can reach Argus. | On an office PC, search the web for "what is my IP". |

## 3. Check it first (what-if)

What-if shows what the deployment would build, and builds nothing. Open [Azure Cloud Shell](https://shell.azure.com) in
**PowerShell**. Change the first five lines to your own values (leave `$template` as it is), then paste it all:

```powershell
$subscription = 'THE_SUBSCRIPTION_ARGUS_GOES_IN'
$rg = 'argus'
$region = 'eastus'
$addresses = @('203.0.113.10')
$user = 'THE_USER_NAME_HIGATE_SENT'
$template = 'https://raw.githubusercontent.com/Higate-Security/argus-connect/v0.2.28/azuredeploy.json'
$registry = 'higateargus.azurecr.io'
$params = $null
if (Set-AzContext -Subscription $subscription -ErrorAction SilentlyContinue) {
  $params = @{ allowedAddresses = $addresses; registryUsername = $user; registryPassword = (Read-Host 'Pull credential password' -MaskInput) }
  $basic = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("$($user):$($params.registryPassword)"))
  $check = ''
  try { $answer = Invoke-RestMethod -Uri "https://$registry/oauth2/token?service=$registry&scope=repository:argus:pull" -Headers @{ Authorization = "Basic $basic" } }
  catch { $check = if ([int]$_.Exception.Response.StatusCode -eq 401) { 'refused' } else { "it couldn't be reached: $($_.Exception.Message)" } }
  if (-not $check) {
    try {
      $part = "$(if ($answer.access_token) { $answer.access_token } else { $answer.token })".Split('.')[1].Replace('-', '+').Replace('_', '/')
      $claims = [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($part + ('=' * ((4 - $part.Length % 4) % 4)))) | ConvertFrom-Json
      if (-not @($claims.access | Where-Object { $_.type -eq 'repository' -and $_.name -eq 'argus' -and @($_.actions) -contains 'pull' })) { $check = 'no pull' }
    } catch { $check = "its answer couldn't be read" }
  }
  $basic = $null; $answer = $null
  if ($check -eq 'refused') { $params = $null; Write-Host "Stopped: the registry refused that user name and password. Check both against what Higate sent, then paste this block again. Nothing was changed." -ForegroundColor Red }
  elseif ($check -eq 'no pull') { $params = $null; Write-Host "Stopped: the registry accepted that user name and password, but they can't pull Argus's image. Ask Higate to check the pull credential. Nothing was changed." -ForegroundColor Red }
  elseif ($check) { Write-Host "Couldn't check the password with $registry ($check). Carrying on; check it against what Higate sent." -ForegroundColor Yellow }
  if ($params) {
    if (-not (Get-AzResourceGroup -Name $rg -ErrorAction SilentlyContinue)) { New-AzResourceGroup -Name $rg -Location $region }
    New-AzResourceGroupDeployment -WhatIf -ResourceGroupName $rg -TemplateUri $template -TemplateParameterObject $params
  }
} else { Write-Host "Stopped: your account can't use the subscription '$subscription' (step 2). Nothing was made." -ForegroundColor Red }
```

If it stops because it can't use the subscription, your account has no role on it (step 2). If your resource group
already exists, the block uses it as it is, and `$region` isn't used. It asks for the password;
paste it (it shows as stars). It checks the user name and password with Higate's registry before anything else: if the
registry refuses them, or they can't pull Argus's image, it stops there and changes nothing, because a deploy with a
wrong password would take Argus offline. On an empty group it lists 11 resources to create. An empty resource group costs nothing.

## 4. Deploy

In the same Cloud Shell, the same command with a name in place of `-WhatIf`. It then clears the password from the
shell, so to deploy again, or in a new Cloud Shell, paste step 3's block first:

```powershell
if ($params) {
  New-AzResourceGroupDeployment -Name argus -ResourceGroupName $rg -TemplateUri $template -TemplateParameterObject $params
  $params.Remove('registryPassword'); $params = $null
} else { Write-Host "Stopped: step 3 didn't pass in this Cloud Shell, so nothing was deployed. Paste step 3's block first." -ForegroundColor Red }
```

Or use the portal:

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FHigate-Security%2Fargus-connect%2Fv0.2.28%2Fazuredeploy.json)

Choose the resource group, then fill in **Allowed Addresses** as a list, like `["203.0.113.10"]`, **Registry Username**
and **Registry Password**. Leave the rest as it is. The portal doesn't check the password with the registry: a wrong
one still reports **Succeeded**, but Argus can't start until you deploy again with the right one. Step 3's block checks
it first.

Deploying again with the same values changes nothing. Give the password every time you
deploy.

## 5. Connect Argus to Microsoft 365

The deployment ends with one line to run. In Cloud Shell (PowerShell), signed in as the Entra admin from step 2, show it.
If this is a new Cloud Shell, or you deployed from the portal, set `$subscription` and `$rg` to yours first:

```powershell
$subscription = 'THE_SUBSCRIPTION_ARGUS_GOES_IN'
$rg = 'argus'
Set-AzContext -Subscription $subscription | Out-Null
(Get-AzResourceGroupDeployment -ResourceGroupName $rg | Where-Object { $_.Outputs -and $_.Outputs.ContainsKey('nextStep') } | Sort-Object Timestamp -Descending | Select-Object -First 1).Outputs['nextStep'].Value
```

In the portal it's under the resource group > **Deployments** > the deployment > **Outputs** > **nextStep**.

Copy the line it shows and paste it into Cloud Shell. It downloads Argus Connect, checks it, signs you in once, shows
its whole plan and asks before it changes anything. It makes Argus's sign-in app, gives Argus's identity read-only
access, makes the Argus Admins and Argus Viewers groups, and starts the first collection. It's safe to run again.

You're already in Argus Admins. To give others access, open the Entra admin center > **Groups** > **All groups** >
**Argus Admins** (they can change settings) or **Argus Viewers** (they can only look) > **Members** > **Add members**.

## 6. Open Argus

In the same Cloud Shell (with `$subscription` and `$rg` set as in step 5):

```powershell
(Get-AzResourceGroupDeployment -ResourceGroupName $rg | Where-Object { $_.Outputs -and $_.Outputs.ContainsKey('siteUrl') } | Sort-Object Timestamp -Descending | Select-Object -First 1).Outputs['siteUrl'].Value
```

Open that address from one of your allowed addresses and sign in. Then add the 90-day beta licence file Higate sent
you.

## Moving from an Argus server

If this organisation already runs Argus on a Windows server, Connect leaves the server's sign-in app as it is and makes
this Argus its own, named with its address (Argus sign-in (your address)). The server keeps working while you move, and
Connect's plan names both apps before it changes anything. The server's **Argus Admins** and **Argus Viewers** groups are
already there, so Connect stops at them and shows their object IDs. To give this Argus the same people, paste Connect's
line again with ` -AdminsGroupId` and ` -ViewersGroupId` and those two IDs added at the end.

Once nobody uses the server, remove it as its own guide says. In Entra, the server's apps are **Argus sign-in** and
**Argus Collector**. This Argus's sign-in app is the one named with its own address: don't delete that one.

## Removing Argus

Delete the resource group. That removes everything, including Argus's data. The key vault can't be purged: Azure keeps
it, deleted, for 90 days. To deploy into a group with the same name within those days, make the group again and recover
the vault into it first. Set the first two lines to the group's name and region:

```powershell
$rg = 'argus'
$region = 'eastus'
New-AzResourceGroup -Name $rg -Location $region
$vault = Get-AzKeyVault -InRemovedState | Where-Object { "$($_.Id) $($_.ResourceId)" -like "*/resourceGroups/$rg/*" } |
    Sort-Object DeletionDate -Descending | Select-Object -First 1
Undo-AzKeyVaultRemoval -VaultName $vault.VaultName -ResourceGroupName $rg -Location $vault.Location
```

## Checking the template

The template for this release is `azuredeploy.json`, SHA-256 `91e0ac660f1e79d7c0cafdec4e8bd155bb85e66448f55daf372f92aeca0eb7d2`. In Cloud Shell:

```powershell
$template = 'https://raw.githubusercontent.com/Higate-Security/argus-connect/v0.2.28/azuredeploy.json'
Invoke-WebRequest -Uri $template -OutFile ./azuredeploy.json
(Get-FileHash ./azuredeploy.json -Algorithm SHA256).Hash
```

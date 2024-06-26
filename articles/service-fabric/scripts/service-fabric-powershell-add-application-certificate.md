---
title: Add application cert to a cluster in PowerShell
description: Azure PowerShell Script Sample - Add an application certificate to a Service Fabric cluster.
services: service-fabric
author: athinanthny
manager: chackdan
ms.service: service-fabric
ms.topic: sample
ms.date: 01/18/2018
ms.author: atsenthi
ms.custom: mvc, devx-track-azurepowershell
---

# Add an application certificate to a Service Fabric cluster

This sample script walks through how to create a certificate in Key Vault and then deploy it to one of the virtual machine scale sets your cluster runs on. This scenario does not use Service Fabric directly, but rather depends on Key Vault and on virtual machine scale sets.

[!INCLUDE [updated-for-az](../../../includes/updated-for-az.md)]

If needed, install the Azure PowerShell using the instruction found in the [Azure PowerShell guide](/powershell/azure/) and then run `Connect-AzAccount` to create a connection with Azure. 

## Create a certificate in Key Vault

```powershell
$VaultName = ""
$CertName = ""
$SubjectName = "CN="

$policy = New-AzKeyVaultCertificatePolicy -SubjectName $SubjectName -IssuerName Self -ValidityInMonths 12
Add-AzKeyVaultCertificate -VaultName $VaultName -Name $CertName -CertificatePolicy $policy
```

## Or upload an existing certificate into Key Vault

```powershell
$VaultName= ""
$CertName= ""
$CertPassword= ""
$PathToPFX= ""

$bytes = [System.IO.File]::ReadAllBytes($PathToPFX)
$base64 = [System.Convert]::ToBase64String($bytes)
$jsonBlob = @{
   data = $base64
   dataType = 'pfx'
   password = $CertPassword
   } | ConvertTo-Json
$contentbytes = [System.Text.Encoding]::UTF8.GetBytes($jsonBlob)
$content = [System.Convert]::ToBase64String($contentbytes)

$SecretValue = ConvertTo-SecureString -String $content -AsPlainText -Force

# Upload the certificate to the key vault as a secret
$Secret = Set-AzKeyVaultSecret -VaultName $VaultName -Name $CertName -SecretValue $SecretValue

```

## Update virtual machine scale sets profile with certificate

1. In PowerShell, define the below variables:
```powershell
$ResourceGroupName = ""
$VMSSName = ""
$CertStore = "My" # Update this with the store you want your certificate placed in, this is LocalMachine\My
```

2. Create the `$CertConfig` variable:

   2.1. If you have added your certificate to the KeyVault certificates, run the command:
   ```powershell
   $CertConfig = New-AzVmssVaultCertificateConfig -CertificateUrl (Get-AzKeyVaultCertificate -VaultName $VaultName -Name $CertName).SecretId -CertificateStore $CertStore
   ```
   2.2. Alternatively, if you have added your certificate to the KeyVault secrets, run the command:
   ```powershell
   $CertConfig = New-AzVmssVaultCertificateConfig -CertificateUrl (Get-AzKeyVaultSecret -VaultName $VaultName -Name $CertName).Id -CertificateStore $CertStore
   ```

3. Retrieve the Service Fabric VMSS resource:
```powershell
$VMSS = Get-AzVmss -ResourceGroupName $ResourceGroupName -VMScaleSetName $VMSSName
```

4. Add the certificate configuration we created in step 2 to the VMSS profile:
   
   4.1. If the KeyVault is already known by the virtual machine scale set, for example if the cluster certificate is deployed from this KeyVault, run the command: 
   ```powershell
   $VMSS.virtualmachineprofile.osProfile.secrets[0].vaultCertificates.Add($CertConfig)
   ```
   4.2. Otherwise use the below one:
   ```powershell
   $VMSS = Add-AzVmssSecret -VirtualMachineScaleSet $VMSS -SourceVaultId (Get-AzKeyVault -VaultName $VaultName).ResourceId  -VaultCertificate $CertConfig
   ```
5. Update the virtual machine scale set:
```powershell
Update-AzVmss -ResourceGroupName $ResourceGroupName -VirtualMachineScaleSet $VMSS -VMScaleSetName $VMSSName
```

> If you would like the certificate placed on multiple node types in your cluster, points 3, 4 and 5 should be repeated for each node type that should have the certificate. Do note that you'll also need to change the variable values we defined in point 1 to the new node types.

## Script explanation

This script uses the following commands: Each command in the table links to command specific documentation.

| Command | Notes |
|---|---|
| [New-AzKeyVaultCertificatePolicy](/powershell/module/az.keyvault/New-AzKeyVaultCertificatePolicy) | Creates an in-memory policy representing the certificate |
| [Add-AzKeyVaultCertificate](/powershell/module/az.keyvault/Add-AzKeyVaultCertificate)| Deploys the policy to Key Vault Certificates |
| [Set-AzKeyVaultSecret](/powershell/module/az.keyvault/Set-AzKeyVaultSecret)| Deploys the policy to Key Vault Secrets |
| [New-AzVmssVaultCertificateConfig](/powershell/module/az.compute/New-AzVmssVaultCertificateConfig) | Creates an in-memory config representing the certificate in a VM |
| [Get-AzVmss](/powershell/module/az.compute/Get-AzVmss) |  |
| [Add-AzVmssSecret](/powershell/module/az.compute/Add-AzVmssSecret) | Adds the certificate to the in-memory definition of the virtual machine scale set |
| [Update-AzVmss](/powershell/module/az.compute/Update-AzVmss) | Deploys the new definition of the virtual machine scale set |

## Next steps

For more information on the Azure PowerShell module, see [Azure PowerShell documentation](/powershell/azure/).

Additional Azure PowerShell samples for Azure Service Fabric can be found in the [Azure PowerShell samples](../service-fabric-powershell-samples.md).

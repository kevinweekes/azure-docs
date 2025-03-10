---
title: Log in to a Linux VM with Azure Active Directory credentials
description: Learn how to create and configure a Linux VM to sign in using Azure Active Directory authentication.

ms.service: virtual-machines
ms.topic: how-to
ms.workload: infrastructure
ms.date: 10/21/2021
ms.author: joflore
author: MicrosoftGuyJFlo
manager: daveba
ms.reviewer: sandeo
ms.custom: references_regions

ROBOTS: NOINDEX
---
# Deprecated: Login to a Linux virtual machine in Azure with Azure Active Directory using device code flow authentication

> [!CAUTION]
> **The public preview feature described in this article was deprecated August 15, 2021.**
> 
> This feature is being replaced with the ability to use Azure AD and SSH via certificate-based authentication. For more information see the article, [Preview: Login to a Linux virtual machine in Azure with Azure Active Directory using SSH certificate-based authentication](../../active-directory/devices/howto-vm-sign-in-azure-ad-linux.md). To migrate from the old version to this version, see [Migration from previous preview](../../active-directory/devices/howto-vm-sign-in-azure-ad-linux.md#migration-from-previous-preview)

To improve the security of Linux virtual machines (VMs) in Azure, you can integrate with Azure Active Directory (AD) authentication. When you use Azure AD authentication for Linux VMs, you centrally control and enforce policies that allow or deny access to the VMs. This article shows you how to create and configure a Linux VM to use Azure AD authentication.

There are many benefits of using Azure AD authentication to log in to Linux VMs in Azure, including:

- **Improved security:**
  - You can use your corporate AD credentials to log in to Azure Linux VMs. There is no need to create local administrator accounts and manage credential lifetime.
  - By reducing your reliance on local administrator accounts, you do not need to worry about credential loss/theft, users configuring weak credentials etc.
  - The password complexity and password lifetime policies configured for your Azure AD directory help secure Linux VMs as well.
  - To further secure login to Azure virtual machines, you can configure multi-factor authentication.
  - The ability to log in to Linux VMs with Azure Active Directory also works for customers that use [Federation Services](../../active-directory/hybrid/how-to-connect-fed-whatis.md).

- **Seamless collaboration:** With Azure role-based access control (Azure RBAC), you can specify who can sign in to a given VM as a regular user or with administrator privileges. When users join or leave your team, you can update the Azure RBAC policy for the VM to grant access as appropriate. This experience is much simpler than having to scrub VMs to remove unnecessary SSH public keys. When employees leave your organization and their user account is disabled or removed from Azure AD, they no longer have access to your resources.

## Supported Azure regions and Linux distributions

The following Linux distributions are currently supported during the preview of this feature:

| Distribution | Version |
| --- | --- |
| CentOS | CentOS 6, CentOS 7 |
| Debian | Debian 9 |
| openSUSE | openSUSE Leap 42.3 |
| RedHat Enterprise Linux | RHEL 6, RHEL 7 | 
| SUSE Linux Enterprise Server | SLES 12 |
| Ubuntu Server | Ubuntu 14.04 LTS, Ubuntu Server 16.04, and Ubuntu Server 18.04 |

> [!IMPORTANT]
> The preview is not supported in Azure Government or sovereign clouds.
>
> It's not supported to use this extension on Azure Kubernetes Service (AKS) clusters. For more information, see [Support policies for AKS](../../aks/support-policies.md).

If you choose to install and use the CLI locally, this tutorial requires that you are running the Azure CLI version 2.0.31 or later. Run `az --version` to find the version. If you need to install or upgrade, see [Install Azure CLI]( /cli/azure/install-azure-cli).

## Network requirements

To enable Azure AD authentication for your Linux VMs in Azure, you need to ensure your VMs network configuration permits outbound access to the following endpoints over TCP port 443:

* https:\//login.microsoftonline.com
* https:\//login.windows.net
* https:\//device.login.microsoftonline.com
* https:\//pas.windows.net
* https:\//management.azure.com
* https:\//packages.microsoft.com

> [!NOTE]
> Currently, Azure network security groups can't be configured for VMs enabled with Azure AD authentication.

## Create a Linux virtual machine

Create a resource group with [az group create](/cli/azure/group#az_group_create), then create a VM with [az vm create](/cli/azure/vm#az_vm_create) using a supported distro and in a supported region. The following example deploys a VM named *myVM* that uses *Ubuntu 16.04 LTS* into a resource group named *myResourceGroup* in the *southcentralus* region. In the following examples, you can provide your own resource group and VM names as needed.

```azurecli-interactive
az group create --name myResourceGroup --location southcentralus

az vm create \
    --resource-group myResourceGroup \
    --name myVM \
    --image UbuntuLTS \
    --admin-username azureuser \
    --generate-ssh-keys
```

It takes a few minutes to create the VM and supporting resources.

## Install the Azure AD login VM extension

> [!NOTE]
> If deploying this extension to a previously created VM ensure the machine has at least 1GB of memory allocated else the extension will fail to install

To log in to a Linux VM with Azure AD credentials, install the Azure Active Directory login VM extension. VM extensions are small applications that provide post-deployment configuration and automation tasks on Azure virtual machines. Use [az vm extension set](/cli/azure/vm/extension#az_vm_extension_set) to install the *AADLoginForLinux* extension on the VM named *myVM* in the *myResourceGroup* resource group:

```azurecli-interactive
az vm extension set \
    --publisher Microsoft.Azure.ActiveDirectory.LinuxSSH \
    --name AADLoginForLinux \
    --resource-group myResourceGroup \
    --vm-name myVM
```

The *provisioningState* of *Succeeded* is shown once the extension is successfully installed on the VM. The VM needs a running VM agent to install the extension. For more information, see [VM Agent Overview](../extensions/agent-windows.md).

## Configure role assignments for the VM

Azure role-based access control (Azure RBAC) policy determines who can log in to the VM. Two Azure roles are used to authorize VM login:

- **Virtual Machine Administrator Login**: Users with this role assigned can log in to an Azure virtual machine with Windows Administrator or Linux root user privileges.
- **Virtual Machine User Login**: Users with this role assigned can log in to an Azure virtual machine with regular user privileges.

> [!NOTE]
> To allow a user to log in to the VM over SSH, you must assign either the *Virtual Machine Administrator Login* or *Virtual Machine User Login* role. The Virtual Machine Administrator Login and Virtual Machine User Login roles use dataActions and thus cannot be assigned at management group scope. Currently these roles can only be assigned at the subscription, resource group or resource scope. An Azure user with the *Owner* or *Contributor* roles assigned for a VM do not automatically have privileges to log in to the VM over SSH. 

The following example uses [az role assignment create](/cli/azure/role/assignment#az_role_assignment_create) to assign the *Virtual Machine Administrator Login* role to the VM for your current Azure user. The username of your active Azure account is obtained with [az account show](/cli/azure/account#az_account_show), and the *scope* is set to the VM created in a previous step with [az vm show](/cli/azure/vm#az_vm_show). The scope could also be assigned at a resource group or subscription level, and normal Azure RBAC inheritance permissions apply. For more information, see [Azure RBAC](../../role-based-access-control/overview.md)

```azurecli-interactive
username=$(az account show --query user.name --output tsv)
vm=$(az vm show --resource-group myResourceGroup --name myVM --query id -o tsv)

az role assignment create \
    --role "Virtual Machine Administrator Login" \
    --assignee $username \
    --scope $vm
```

> [!NOTE]
> If your AAD domain and logon username domain do not match, you must specify the object ID of your user account with the *--assignee-object-id*, not just the username for *--assignee*. You can obtain the object ID for your user account with [az ad user list](/cli/azure/ad/user#az_ad_user_list).

For more information on how to use Azure RBAC to manage access to your Azure subscription resources, see using the [Azure CLI](../../role-based-access-control/role-assignments-cli.md), [Azure portal](../../role-based-access-control/role-assignments-portal.md), or [Azure PowerShell](../../role-based-access-control/role-assignments-powershell.md).

## Using Conditional Access

You can enforce Conditional Access policies such as multi-factor authentication or user sign-in risk check before authorizing access to Linux VMs in Azure that are enabled with Azure AD sign in. To apply Conditional Access policy, you must select "Microsoft Azure Linux Virtual Machine Sign-In" app from the cloud apps or actions assignment option and then use Sign-in risk as a condition and/or require multi-factor authentication as a grant access control. 

> [!WARNING]
> Per-user Enabled/Enforced Azure AD Multi-Factor Authentication is not supported for VM sign-in.

## Log in to the Linux virtual machine

First, view the public IP address of your VM with [az vm show](/cli/azure/vm#az_vm_show):

```azurecli-interactive
az vm show --resource-group myResourceGroup --name myVM -d --query publicIps -o tsv
```

Log in to the Azure Linux virtual machine using your Azure AD credentials. The `-l` parameter lets you specify your own Azure AD account address. Replace the example account with your own. Account addresses should be entered in all lowercase. Replace the example IP address with the public IP address of your VM from the previous command.

```console
ssh -l azureuser@contoso.onmicrosoft.com 10.11.123.456
```

You are prompted to sign in to Azure AD with a one-time use code at [https://microsoft.com/devicelogin](https://microsoft.com/devicelogin). Copy and paste the one-time use code into the device login page.

When prompted, enter your Azure AD login credentials at the login page. 

The following message is shown in the web browser when you have successfully authenticated: `You have signed in to the Microsoft Azure Linux Virtual Machine Sign-In application on your device.`

Close the browser window, return to the SSH prompt, and press the **Enter** key. 

You are now signed in to the Azure Linux virtual machine with the role permissions as assigned, such as *VM User* or *VM Administrator*. If your user account is assigned the *Virtual Machine Administrator Login* role, you can use `sudo` to run commands that require root privileges.

## Sudo and AAD login

The first time that you run sudo, you will be asked to authenticate a second time. If you don't want to have to authenticate again to run sudo, you can edit your sudoers file `/etc/sudoers.d/aad_admins` and replace this line:

```bash
%aad_admins ALL=(ALL) ALL
```

With this line:

```bash
%aad_admins ALL=(ALL) NOPASSWD:ALL
```

## Troubleshoot sign-in issues

Some common errors when you try to SSH with Azure AD credentials include no Azure roles assigned, and repeated prompts to sign in. Use the following sections to correct these issues.

### Access denied: Azure role not assigned

If you see the following error on your SSH prompt, verify that you have configured Azure RBAC policies for the VM that grants the user either the *Virtual Machine Administrator Login* or *Virtual Machine User Login* role:

```output
login as: azureuser@contoso.onmicrosoft.com
Using keyboard-interactive authentication.
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code FJX327AXD to authenticate. Press ENTER when ready.
Using keyboard-interactive authentication.
Access denied:  to sign-in you be assigned a role with action 'Microsoft.Compute/virtualMachines/login/action', for example 'Virtual Machine User Login'
Access denied
```
> [!NOTE]
> If you are running into issues with Azure role assignments, see [Troubleshoot Azure RBAC](../../role-based-access-control/troubleshooting.md#azure-role-assignments-limit).

### Continued SSH sign-in prompts

If you successfully complete the authentication step in a web browser, you may be immediately prompted to sign in again with a fresh code. This error is typically caused by a mismatch between the sign-in name you specified at the SSH prompt and the account you signed in to Azure AD with. To correct this issue:

- Verify that the sign-in name you specified at the SSH prompt is correct. A typo in the sign-in name could cause a mismatch between the sign-in name you specified at the SSH prompt and the account you signed in to Azure AD with. For example, you typed *azuresuer\@contoso.onmicrosoft.com* instead of *azureuser\@contoso.onmicrosoft.com*.
- If you have multiple user accounts, make sure you don't provide a different user account in the browser window when signing in to Azure AD.
- Linux is a case-sensitive operating system. There is a difference between 'Azureuser@contoso.onmicrosoft.com' and 'azureuser@contoso.onmicrosoft.com', which can cause a mismatch. Make sure that you specify the UPN with the correct case-sensitivity at the SSH prompt.

### Other limitations

Users that inherit access rights through nested groups or role assignments aren't currently supported. The user or group must be directly assigned the [required role assignments](#configure-role-assignments-for-the-vm). For example, the use of management groups or nested group role assignments won't grant the correct permissions to allow the user to sign in.

## Preview feedback

Share your feedback about this preview feature or report issues using it on the [Azure AD feedback forum](https://feedback.azure.com/d365community/forum/22920db1-ad25-ec11-b6e6-000d3a4f0789)

## Next steps

For more information on Azure Active Directory, see [What is Azure Active Directory](../../active-directory/fundamentals/active-directory-whatis.md)

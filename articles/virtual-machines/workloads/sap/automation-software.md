---
title: Download SAP software for automation framework
description: Download the SAP software to your Azure environment using Ansible playbooks to use the SAP deployment automation framework on Azure.
author: kimforss
ms.author: kimforss
ms.reviewer: kimforss
ms.date: 11/17/2021
ms.topic: how-to
ms.service: virtual-machines-sap
---

# Download SAP software

You need a copy of the SAP software before you can use [the SAP deployment automation framework on Azure](automation-deployment-framework.md). [Prepare your Azure environment](#configure-key-vault) so you can put the SAP media in your storage account. Then, [download the SAP software using Ansible playbooks](#download-sap-software).

## Prerequisites

- An Azure subscription. If you don't have an Azure subscription, you can [create a free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F).
- An SAP user account (SAP-User or S-User account) with software download privileges.

## Configure key vault

First, configure your deployer key vault secrets. For this example configuration, the resource group is `DEMO-EUS2-DEP00-INFRASTRUCTURE` or `DEMO-SCUS-DEP00-INFRASTRUCTURE`.

1. [Sign in to the Azure CLI](/cli/azure/authenticate-azure-cli) with the account you want to use.

    ```azurecli-interactive
    az login
    ```

1. Add a secret with the username for your SAP user account. Replace `<keyvault-name>` with the name of your deployer key vault. Also replace `<sap-username>` with your SAP username.

    ```azurecli-interactive
     az keyvault secret set --name "S-Username" --vault-name "<keyvault-name>" --value "<sap-username>";
    ```

2. Add a secret with the password for your SAP user account. Replace `<keyvault-name>` with the name of your deployer key vault. Also replace `<sap-password>` with your SAP password.

```azurecli-interactive
az keyvault secret set --name "S-Password" --vault-name "<keyvault-name>" --value "<sap-password>";
```

3. There are two other secrets which are needed in this step for the storage account `sapbits`, are automatically setup by the automation framework. However its always good to verify whether these are existed in your deployer keyvault or not.

    ```azurecli-interactive
    sapbits-access-key
    sapbits-location-base-path
    ```

## Download SAP software

Next, [configure your SAP parameters file](#configure-parameters-file) for the download process. Then, [download the SAP software using Ansible playbooks](#download-sap-software). 

### Configure parameters file

Configure the SAP parameters file:

1. Navigate to the SAP deployment workspace's directory.

    ```bash
    cd ~/Azure_SAP_Automated_Deployment/WORKSPACES/
    ```

1. Create a new directory called `BOMS`:

    ```bash
    mkdir -p ~/Azure_SAP_Automated_Deployment/WORKSPACES/BOMS; cd $_
    ```

1. Create the SAP parameters YAML file.

    ```bash
    cat <<EOF > sap-parameters.yaml
    ---
    bom_base_name:               S41909SPS03_v0006ms
    sapbits_location_base_path:  https://<storage_account_FQDN>/sapbits
    kv_name: Name of your Management/Control Plane keyvault
    secret_prefix:
    ...
    EOF
    ```

1. Open `sap-parameters.yaml` in an editor.

    ```bash
    vi sap-parameters.yaml
    ``` 

1. Update the following parameters:

    1. Change the value of `bom_base_name` to `S41909SPS03_v0006ms`.

    2. Change the value of `sapbits_location_base_path` to the path to your storage account `sapbits`.

    3. Change the value of `kv_name` to the name of the deployer key vault.
   
   4. (If needed) Change the value of `secret_prefix` to match the prefix in your environment (for example DEV-WEEU-SAP)
   
### Execute Ansible playbooks

Then, execute the Ansible playbooks. One way you can execute the playbooks is to use the validator test menu:

1. Run the validator test menu script:

    ```bash
    ~/Azure_SAP_Automated_Deployment/sap-automation/deploy/ansible/validator_test_menu.sh
    ```

1. Select the playbook to execute. For example:
    
    ```output
    1) BoM Downloader
    2) Quit
    Please select playbook: 
    ```


Another option is to execute the Ansible playbooks using the command `ansible-playbook`. 

```bash
ansible-playbook                                                                                   \
  --user        azureadm                                                                           \
  --extra-vars="@sap-parameters.yaml"                                                              \
  ~/Azure_SAP_Automated_Deployment/sap-automation/deploy/ansible/playbook_bom_downloader.yaml
```

## Next steps

> [!div class="nextstepaction"]
> [Get started with deploying the automation framework](automation-get-started.md)

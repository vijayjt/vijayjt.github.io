+++ 
draft = false
date = 2024-01-05T19:05:53Z
title = "Using fzf with Azure AAD Login to SSH interactively to Azure VMs with fzf"
description = ""
slug = ""
authors = []
tags = ["Azure", "AAD Login", "fzf"]
categories = ["tips"]
externalLink = ""
series = []
+++

[AAD Login](https://learn.microsoft.com/en-us/entra/identity/devices/howto-vm-sign-in-azure-ad-linux) or EntraID login now, is a feature that allows you to login to Azure VMs
using your Entra ID account.

It requires that you have the Virtual Machine Login or Virtual Machine Administrator Login roles (the former logs you in as a regular user and the latter an administrator).

[fzf](https://github.com/junegunn/fzf) is a command-line fuzzy finder, as the project site states it is an interactive UNIX filter for command-line that can be used with any list, files, etc.

Two example functions are provided below that use fzf to allow you to interactively select the subscription, VM/VMSS to SSH to.

```bash
function az_ssh_vmss_instance(){
   AZ_SSH_CFG="${HOME}/.azure-sshconfig"
   AZ_SSH_KEYS="${HOME}/.azure-sshkeys"

   sub=$(az account list -o table --query "[].{Subscription:name}"|fzf --tac)
   vmss=$(az vmss list -o table --subscription $sub --query "[].{name:name,resourceGroup:resourceGroup,location:location,env:tags.Environment,domain:tags.AuthProxyDomain,usedBy:tags.UsedBy}" | fzf --tac)

   vmss_name=$(echo $vmss | awk '{print $1}')
   rg=$(echo $vmss | awk '{print $2}')
   instance_ip=$(az vmss nic list --resource-group $rg --vmss-name $vmss_name --subscription $sub -o json | jq -r '.[].ipConfigurations[].privateIPAddress' | fzf --tac)
   ssh-keygen -f "${HOME}/.ssh/known_hosts" -R "${instance_ip}"
   az ssh config --ip "${instance_ip}" --file "${AZ_SSH_CFG}" --keys-destination-folder "${AZ_SSH_KEYS}"
   az ssh vm --ip "${instance_ip}"

   # We can then use SCP like this to copy files to/from the remote vm
   # scp -F ./azure-sshconfig <local file> <vm ip>:~/temp
   # scp -F ./azure-sshconfig <vm ip>:~/some/file <local file>
}

function az_ssh_vm(){
   sub=$(az account list -o table --query "[].{Subscription:name}"|fzf --tac)
   vms=$(az vm list -o table --subscription $sub --query "[].{name:name,resourceGroup:resourceGroup,location:location,env:tags.Environment,component:tags.Component,client:tags.Client}" | fzf --tac)
   vm_name=$(echo $vms | awk '{print $1}')
   rg=$(echo $vms | awk '{print $2}')
   nic_id=$(az vm nic list --resource-group $rg --vm-name $vm_name --subscription $sub -o json | jq -r '.[].id')
   instance_ip=$(az vm nic show --resource-group $rg --vm-name $vm_name --subscription $sub --nic $nic_id -o json | jq -r '.ipConfigurations[].privateIPAddress' | fzf --tac)
   az ssh config --file ~/.ssh/config -n ${vm_name} -g  ${rg} --subscription ${sub} --prefer-private-ip
   az ssh vm --ip "${instance_ip}"
}
```

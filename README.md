# Local Kubernetes Lab on Windows WSL

This Vagrant file will provision a working kubernetes cluster consisting of 1 controlplane node and 2 worker nodes. Vagrant file will run ansible to install and configure kubernetes on all nodes using kubeadm.

## Requirements

1. WSL (tested on Ubuntu)
2. VirtualBox installed on Windows Host
3. Vagrant installed on WSL
4. Ansible installed on WSL

## Steps

1. Create this file in wsl to fix error *"private key not secure"*. Resolves permissions issue since files are on windows host and not WSL

  > */etc/wsl.conf*  
  > `[automount]`  
  > `options = "metadata,umask=22,fmask=11"`

2. Append snippet into wsl .bashrc or .zshrc

  > `export VAGRANT_WSL_ENABLE_WINDOWS_ACCESS="1"`  
  > `export VAGRANT_WSL_WINDOWS_ACCESS_USER_HOME_PATH="/mnt/c/Users/angelos/"`  
  > `export PATH="$PATH:/mnt/d/Program Files/Oracle/VirtualBox"`

3. On windows, go to Control Panel\All Control Panel Items\Windows Defender Firewall\Allowed apps. Check all VirtualBox items (e.g VirtualBox Headless Frontend, VirtualBox Virtual Machine, etc.)

4. Run Vagrant, while on repository root folder type `vagrant up`
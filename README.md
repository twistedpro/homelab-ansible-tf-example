### Home Lab

This repo is a work in progress for my homelab. It contains Terraform, Ansible and Packer (TODO) code/configurations to provision and configure my homelab using IaC and configuration tools with minimal manual input.

# Provisioning Proxmox VMs with Terraform and Ansible
2022-09-24 (Last Updated 2022-10-08) — Written by Lachlan — 9 min read
#proxmox  #ansible  #terraform 
Provisioning Proxmox VMs with Terraform and Ansible
One of the key requirements of this project was to build everything using reproducible processes. I’m very glad that I did because I had to test the limits while I was building the new k3s cluster. For reasons I’ve covered in other posts, the k3s cluster was unstable and would go down. I had to delete the VMs completely and rebuild it more times than I can count.

Terraform is described as an Infrastructure-as-code tool. It can be used to provision, change, and version resources within the environment. It’s primarily used with cloud providers to automate and manage the resource configuration for the cloud resources.

Proxmox Virtual Environment (PVE) is a converged compute, network, and storage virtualization environment. It’s built on top of Debian Linux and supports the KVM hypervisor, Linux Containers (LXC), software defined storage, and networking.

Ansible is an automation configuration management tool. Configuration and automation tasks are defined in code and executed by playbooks. The main features is that all of the tasks are intended to be idempotent, meaning that they can be be running again and again and always achieve the same outcome. If I need a line entered in a configuration file, it will check if that line already exists as I defined it before modifying the file.

The approach I took to integrate all three of these components was to use use Ansible to execute Terraform with the Proxmox provider. Within the Ansible repository, I created a terraform directory and subdirectories for each Terraform project.

#Terraform 
The way you would normally use Terraform is, in a new directory, create a main.tf file which defines blocks of code that define providers, variables, and resources:
```yaml
terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = ">=2.8.0"
    }
  }
}

provider "proxmox" {
  pm_api_url          = var.pve_api_url
  pm_api_token_id     = var.pve_token_id
  pm_api_token_secret = var.pve_token_secret
  pm_log_enable       = false
  pm_log_file         = "terraform-plugin-proxmox.log"
  pm_parallel         = 1
  pm_timeout          = 600
  pm_log_levels = {
    _default    = "debug"
    _capturelog = ""
  }
}

variable "pve_api_url" {
  description = "The URL to the Proxmox API"
  type        = string
  sensitive   = false
  default     = "https://proxmox-host:8006/api2/json"
}
variable "pve_token_id" {
  description = "Proxmox API token"
  type        = string
  sensitive   = false
  default     = "token"
}
variable "pve_token_secret" {
  description = "Proxmox API secret"
  type        = string
  sensitive   = false
  default     = "secret"
}
```
Copy
Note: In my repository, the variables values are stored in a separate vars.tf file so that they could be encrypted with ansible-vault and only decrypted at runtime. More on that later.

Once the main.tf is created, you would initiate the terraform directory with the command “terraform init”. The init command will download all of the provider plugins and create the state files which allow it to keep track of the status of the resources.

Whenever resources are added or changed, executing the command “terraform plan” will show the changes that need to be made and “terraform apply” will attempt to create or change the resources to match the definitions. Sometimes that requires deleting the resources and recreating them.

When you are done with the resources, simply use the “terraform destroy” command. I also created Ansible playbooks to do the the equivalent of these commands using the Ansible Terraform community plugin.

# Create the template VM 
Before we dive into creating VMs, we need to create a template VM in Proxmox. I used the Debian cloud images as my base image so that I could configure them on first boot with cloud-init. At the time I created the cluster, k3s did not support nftables which is default in Debian 11 Bullseye so I stuck with Debian 10 Buster. Actually I tried Debian 11 first and had to scrap that and start over.

In order to create the template, first, I needed to install the qemu-guest-agent in the cloud image:

Install libguestfs-tools on Proxmox server:

`apt install libguestfs-tools`
Copy
Use virt-customize to install qemu-guest-agent directly into the image:

``virt-customize -a debian-10-generic-amd64.qcow2 --install qemu-guest-agent`
Copy
Create the new VM in proxmox (I use VMIDs of 9000+ for templates)

```
qm create <VMID> --memory 4096 --net0 virtio,bridge=vmbr0
qm importdisk <VMID> debian-10-generic-amd64.qcow2 <STORAGE POOL> --format qcow2
qm set <VMID> --scsihw virtio-scsi-pci --scsi0 <STORAGE POOL>:vm-<VMID>-disk-0
qm set <VMID> --agent enabled=1,fstrim_cloned_disks=1
qm set <VMID> --name k3s-template
```
Create Cloud-Init Disk and configure boot:
```
qm set <VMID> --ide2 <STORAGE POOL>:cloudinit
qm set <VMID> --boot c --bootdisk scsi0
qm set <VMID> --serial0 socket --vga serial0
```
Now convert the new VM to a template:

`qm template $VM_ID`

Finally, I modified the cloud-init config on the template to add my ssh keys.

#Create Proxmox Role and User for Terraform 
Since we will be using Terraform to interact with the Proxmox API, we need to create a role with an API token.
```
pveum role add TerraformProv -privs "VM.Allocate VM.Clone VM.Config.CDROM VM.Config.CPU VM.Config.Cloudinit VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Monitor VM.Audit VM.PowerMgmt Datastore.AllocateSpace Datastore.Audit"
pveum user add terraform-prov@pve --password <password>
pveum aclmod / -user terraform-prov@pve -role TerraformProv
```
I created the API token using the Proxmox web GUI for use in the Terraform variables, but the provider supports using a username and password.

# Full Terraform 
The main.tf I used for create the VMs allows for creating a variable number of master node and a variable number of worker nodes. Each VM can be created on a different Proxmox node. I used 3 master nodes and 3 worker VMs and alternated them between the two Proxmox nodes.

Each VM has 4 cores, 8GB memory, a 32GB root disk, and a 250GB data disk for Longhorn volumes. They have statically assigned IP addresses and they are on their own VLAN. An ansible user will be created as part of cloud-init which allows logins from my Ansible ssh key.
```
variable "num_k3s_masters" {
  default = 3
}

variable "k3s_master_pve_node" {
  description = "The PVE node to target"
  type        = list(string)
  sensitive   = false
  default     = ["proxmox1", "proxmox2", "proxmox1"]
}

variable "k3s_master_ip_addresses" {
  description = "List of IP addresses for master node(s)"
  type        = list(string)
  default     = ["xxx.xxx.xxx.201/24", "xxx.xxx.xxx.202/24", "xxx.xxx.xxx.203/24"]
}

variable "num_k3s_master_mem" {
  default = "8192"
}

variable "k3s_master_cores" {
  default = "4"
}

variable "k3s_master_root_disk_size" {
  default = "32G"
}

variable "k3s_master_data_disk_size" {
  default = "250G"
}

variable "k3s_master_disk_storage" {
  default = "vmdata"
}

variable "num_k3s_nodes" {
  default = 3
}

variable "k3s_worker_pve_node" {
  description = "The PVE node to target"
  type        = list(string)
  sensitive   = false
  default     = ["proxmox2", "proxmox1", "proxmox2"]
}

variable "k3s_worker_ip_addresses" {
  description = "List of IP addresses for master node(s)"
  type        = list(string)
  default     = ["xxx.xxx.xxx.204/24", "xxx.xxx.xxx.205/24", "xxx.xxx.xxx.206/24"]
}

variable "num_k3s_node_mem" {
  default = "8192"
}

variable "k3s_node_cores" {
  default = "4"
}

variable "k3s_node_root_disk_size" {
  default = "32G"
}

variable "k3s_node_data_disk_size" {
  default = "250G"
}

variable "k3s_node_disk_storage" {
  default = "vmdata"
}

variable "k3s_gateway" {
  type    = string
  default = "xxx.xxx.xxx.1"
}

variable "k3s_vlan" {
  default = 33
}

variable "template_vm_name" {
  default = "k3s-template"
}

variable "k3s_nameserver_domain" {
  type    = string
  default = "domain.ld"
}

variable "k3s_nameserver" {
  type    = string
  default = "xxx.xxx.xxx.1"
}

variable "k3s_user" {
  default = "ansible"
}

terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = ">=2.8.0"
    }
  }
}

provider "proxmox" {
  pm_api_url          = var.pve_api_url
  pm_api_token_id     = var.pve_token_id
  pm_api_token_secret = var.pve_token_secret
  pm_log_enable       = false
  pm_log_file         = "terraform-plugin-proxmox.log"
  pm_parallel         = 1
  pm_timeout          = 600
  pm_log_levels = {
    _default    = "debug"
    _capturelog = ""
  }
}

resource "proxmox_vm_qemu" "proxmox_vm_master" {
  count = var.num_k3s_masters
  name  = "k3s-master-${count.index}"
  desc  = "K3S Master Node"
  ipconfig0   = "gw=${var.k3s_gateway},ip=${var.k3s_master_ip_addresses[count.index]}"
  target_node = var.k3s_master_pve_node[count.index]
  onboot      = true
  hastate     = "started"
  # Same CPU as the Physical host, possible to add cpu flags
  # Ex: "host,flags=+md-clear;+pcid;+spec-ctrl;+ssbd;+pdpe1gb"
  cpu        = "host"
  numa       = false
  clone      = "${var.template_vm_name}-${var.k3s_master_pve_node[count.index]}"
  os_type    = "cloud-init"
  agent      = 1
  ciuser     = var.k3s_user
  memory     = var.num_k3s_master_mem
  cores      = var.k3s_master_cores
  nameserver = var.k3s_nameserver
 
  network {
    model  = "virtio"
    bridge = "vmbr0"
    tag    = var.k3s_vlan
  }

  serial {
    id   = 0
    type = "socket"
  }

  vga {
    type = "serial0"
  }

  disk {
    size    = var.k3s_master_root_disk_size
    storage = var.k3s_master_disk_storage
    type    = "scsi"
    format  = "qcow2"
    backup  = 1
  }

  disk {
    size    = var.k3s_master_data_disk_size
    storage = var.k3s_master_disk_storage
    type    = "scsi"
    format  = "qcow2"
    backup  = 1
  }

  lifecycle {
    ignore_changes = [
      network, disk, sshkeys, target_node
    ]
  }
}

resource "proxmox_vm_qemu" "proxmox_vm_workers" {
  count = var.num_k3s_nodes
  name  = "k3s-worker-${count.index}"
  ipconfig0   = "gw=${var.k3s_gateway},ip=${var.k3s_worker_ip_addresses[count.index]}"
  target_node = var.k3s_worker_pve_node[count.index]
  onboot      = true
  hastate     = "started"
  # Same CPU as the Physical host, possible to add cpu flags
  # Ex: "host,flags=+md-clear;+pcid;+spec-ctrl;+ssbd;+pdpe1gb"
  cpu   = "host"
  numa  = false
  clone = "${var.template_vm_name}-${var.k3s_worker_pve_node[count.index]}"
  os_type    = "cloud-init"
  agent      = 1
  ciuser     = var.k3s_user
  memory     = var.num_k3s_node_mem
  cores      = var.k3s_node_cores
  nameserver = var.k3s_nameserver

  network {
    model  = "virtio"
    bridge = "vmbr0"
    tag    = var.k3s_vlan
  }

  serial {
    id   = 0
    type = "socket"
  }

  vga {
    type = "serial0"
  }

  disk {
    size    = var.k3s_node_root_disk_size
    storage = var.k3s_node_disk_storage
    type    = "scsi"
    format  = "qcow2"
    backup  = 1
  }

  disk {
    size    = var.k3s_node_data_disk_size
    storage = var.k3s_node_disk_storage
    type    = "scsi"
    format  = "qcow2"
    backup  = 1
  }

  lifecycle {
    ignore_changes = [
      network, disk, sshkeys, target_node
    ]
  }
}

data "template_file" "k3s" {
  template = file("./templates/k3s.tpl")
  vars = {
    k3s_master_ip = "${join("\n", [for instance in proxmox_vm_qemu.proxmox_vm_master : join("", [instance.name, " ansible_host=", instance.default_ipv4_address])])}"
    k3s_node_ip   = "${join("\n", [for instance in proxmox_vm_qemu.proxmox_vm_workers : join("", [instance.name, " ansible_host=", instance.default_ipv4_address])])}"
  }
}

resource "local_file" "k3s_file" {
  content  = data.template_file.k3s.rendered
  filename = "../../inventory/k3s"
}

output "Master-IPS" {
  value = ["${proxmox_vm_qemu.proxmox_vm_master.*.default_ipv4_address}"]
}
output "worker-IPS" {
  value = ["${proxmox_vm_qemu.proxmox_vm_workers.*.default_ipv4_address}"]
}
Copy
Most of this is pretty self-explantory, but the lifecycle section tells Terraform to ignore any changes to network configuration, disk, sshkeys, and the node that the VMs is on. This allows configuration drift in those areas without triggering Terraform to destroy the VM and recreate it with the “correct” configuration.

The last section creates an Ansible inventory file with each of the hosts and their IP addresses. The template, k3s.tpl, is in a templates directory:

[master]
${k3s_master_ip}

[node]
${k3s_node_ip}

[k3s_cluster:children]
master
node
Ansible Playbooks 
The playbook k3s-provision.yaml will decrypt the vars.tf file and execute Terraform:

---
- hosts: localhost
  name: Create K3S infrastructure
  vars:
    terraform_dir: terraform/k3s

  pre_tasks:
    - name: Decrypt variables file
      ansible.builtin.copy:
        src: "{{ terraform_dir }}/var.tf-encrypted"
        dest: "{{ terraform_dir }}/var.tf"
        decrypt: yes
                                           
  tasks:
    - name: Creating k3s VMs
      community.general.terraform:
          project_path: "{{ terraform_dir }}"
          state: present
      register: outputs
    - name: Dump oututs
      ansible.builtin.debug:
        var=outputs                           
Copy
The k3s-destroy.yaml playbook will destroy the cluster:

---
- hosts: localhost
  name: Remove K3S infrastructure
  vars:
    terraform_dir: terraform/k3s

  pre_tasks:
    - name: Decrypt variables file
      ansible.builtin.copy:
        src: "{{ terraform_dir }}/var.tf-encrypted"
        dest: "{{ terraform_dir }}/var.tf"
        decrypt: yes
                                           
  tasks:
    - name: Removing k3s VMs
      community.general.terraform:
          project_path: "{{ terraform_dir }}"
          state: absent
      register: outputs
    - name: Dump oututs
      ansible.builtin.debug:
        var=outputs                             
```
Next Steps 
In my next post, I will share the Terraform project and Ansible roles to create a highly-available loadbalancer which routes traffic to multiple Kubernetes cluster based on the domain name in the request. This allowed me to move applications from the old cluster to the new cluster one at a time.

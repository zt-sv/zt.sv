+++
date = '2026-02-26T16:30:16+03:00'
draft = false
title = 'Building a Base Virtual Machine Image with Packer and Ansible'
keywords = [
  "Packer",
  "Ansible",
  "VMware",
  "Rocky",
  "Linux",
  "Hashicorp",
  "CIS Benchmark",
  "automation",
  "DevOps",
  "Infrastructure as Code",
  "Immutable infrastructure"
]
description = 'Automating the build of a Rocky Linux virtual machine image using Packer and Ansible for VMware, with CIS Benchmark compliance'
images = ['author.png']
+++

Imagine a scenario: you need to deploy a new virtual machine for development or testing. You start with a clean distribution image, install the necessary packages, configure the network, and apply security policies — this takes a significant amount of time. Doing this once is manageable. But what if you need many identical machines? And what if, over time, you need to make slight adjustments to the initial configuration?
Packer and Ansible provide the answers to these questions.

HashiCorp Packer is a tool for automated virtual machine image building. Instead of manually creating and configuring a VM, the entire process is described as code.
Ansible, in the context of Packer, is responsible for installing and configuring software within the created templates.

In this guide, we will look at building a virtual machine using Packer and subsequently configuring it with Ansible.

## Rocky Linux for VMware

Let's assume the following task: we need to prepare a base Rocky Linux image, from which virtual machines will be deployed in a private cloud based on VMware Cloud Director (VCD).

Template requirements:

* Compliance with the [CIS Benchmark](https://www.cisecurity.org/cis-benchmarks);
* Pre-installed [Prometheus Node Exporter](https://prometheus.io/docs/guides/node-exporter/);
* The ability to build Rocky Linux 8 and 9;
* The ability to perform reproducible builds.

### Tools

To solve this task, we will need the following tools:

* [Hashicorp Packer](https://developer.hashicorp.com/packer/install);
* [Ansible](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html);
* One of the [VMware desktop hypervisors](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion);
* [VMware OVFTool](https://customerconnect.vmware.com/downloads/get-download?downloadGroup=OVFTOOL441);
* [Python](https://www.python.org/downloads/).

### Project Structure

When running the `packer build` command, Packer reads all `*.pkr.hcl` files in the execution directory and merges them into a single configuration. This allows us to split the configuration into multiple files, improving readability and future maintenance.

We will create a separate directory `rocky` for the template, and split the template configuration into individual files.

Everything related to Ansible will be placed in its own dedicated directory.

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv" >}}
$ tree packer-templates 
packer-templates
├── ansible
│   ├── collections # Ansible collections
│   ├── roles # Ansible roles
│   ├── ansible.cfg # Ansible configuration
│   ├── playbook.yaml
│   └── requirements.yaml # Ansible dependencies
├── artifacts # Directory for the built virtual machines
├── requirements.txt # Python dependencies
└── rocky
    ├── build.pkr.hcl # Build blocks
    ├── packer.pkr.hcl # Packer dependencies block
    ├── source.pkr.hcl # Builder configuration
    └── variables.pkr.hcl # Variable definitions
{{< /terminal >}}

### Installing Ansible

It is most convenient to install Ansible via `pip` into an isolated project virtual environment.

Our task requires building images on Rocky 8. It should be noted that this version uses Python 3.6, which is incompatible with newer versions of [ansible-core](https://pypi.org/project/ansible-core). In this case, an Ansible version < 2.17 must be used.

We pin the Ansible version in the `requirements.txt` file:

```requirements.txt
ansible-core==2.16.16
```

Create and activate a new `venv`:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/packer-templates" >}}
$ python3 -m venv $(pwd)/venv  
$ source $(pwd)/venv/bin/activate
{{< /terminal >}}

And install the dependencies:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/packer-templates" >}}
$ pip3 install -r requirements.txt
{{< /terminal >}}

Since we plan to build multiple images across different OS versions, we must ensure `Ansible facts` saves in memory rather than on disk. Otherwise, cached facts from another system might be reused, leading to errors.

To do this, specify the following in `ansible.cfg`:

{{< highlight cfg "linenos=inline" >}}
[defaults]
gathering = smart  
fact_caching = memory
{{< /highlight >}}

### Defining and Installing Packer Dependencies

The following plugins are required to build the template:

* [VMware](https://developer.hashicorp.com/packer/integrations/vmware/vmware/) - builders for VMware hypervisors;
* [Ansible](https://developer.hashicorp.com/packer/integrations/hashicorp/ansible) - for running Ansible playbooks from Packer;
* [Git](https://developer.hashicorp.com/packer/integrations/ethanmdavidson/git) - for fetching git information.

We add them to the `rocky/packer.pkr.hcl` file and pin the current versions:

{{< highlight hcl "linenos=inline" >}}
packer {  
  required_version = ">= 1.14.0, < 2.0.0"
  required_plugins {
    vmware = {
      version = "= 1.2.0"
      source  = "github.com/hashicorp/vmware"
    }
    ansible = {
      version = "= 1.1.4"
      source  = "github.com/hashicorp/ansible"
    }
    git = {
      version = "= 0.6.5"
      source  = "github.com/ethanmdavidson/git"
    }
  }
}
{{< /highlight >}}

To install the plugins, run the [`init`](https://developer.hashicorp.com/packer/docs/commands/init) command in the template directory:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/packer-templates" >}}
$ packer init rocky
{{< /terminal >}}

### VMware ISO Builder

[VMware ISO](https://developer.hashicorp.com/packer/integrations/hashicorp/vmware/latest/components/builder/iso) allows creating virtual machines from an ISO file in VMware desktop hypervisors.

After creating the virtual machine, the builder passes the kickstart file URL to the OS bootloader via Packer's built-in HTTP server. It then waits for an SSH connection to hand over control to the `build` block and execute the provisioning scripts.

Finally, the built virtual machine is exported to the OVA format, which can be imported into VCD.

For image versioning, a combination of the current date and the git commit hash is used. This helps us understand exactly when and from which commit the image was built.

We will describe the entire builder configuration in the `rocky/source.pkr.hcl` file.

{{< highlight hcl "linenos=inline" >}}
data "git-commit" "cwd-head" {}

locals {
  truncated_sha = substr(data.git-commit.cwd-head.hash, 0, 8)
  build_date    = formatdate("YYYY-MM-DD", timestamp())
  vm_name       = "Rocky${var.rocky_version}-${local.build_date}-${local.truncated_sha}"
  iso_url       = "${var.rocky_repository_url}/${var.rocky_version}/isos/x86_64/Rocky-${var.rocky_version}-x86_64-minimal.iso"
  iso_checksum  = "file:${local.iso_url}.CHECKSUM"
  export_path   = "${abspath(path.cwd)}/artifacts/${local.vm_name}"
  repo_list     = [
    {
      name : "BaseOS",
      url : "${var.rocky_repository_url}/${var.rocky_version}/BaseOS/x86_64/os/",
    },
    {
      name : "AppStream",
      url : "${var.rocky_repository_url}/${var.rocky_version}/AppStream/x86_64/os/",
    },
    {
      name : "extras",
      url : "${var.rocky_repository_url}/${var.rocky_version}/extras/x86_64/os/",
    },
    {
      name : "devel",
      url : "${var.rocky_repository_url}/${var.rocky_version}/devel/x86_64/os/",
    },
  ]
  kickstart = templatefile("${abspath(path.root)}/templates/kickstart.pkrtpl", {
    build_username : var.provisioner_username,
    build_password : var.provisioner_password,
    lvm_layout : var.lvm_layout,
    repo_list : local.repo_list,
    guest_timezone : var.guest_timezone,
    install_packages : var.install_packages,
    uninstall_packages : var.uninstall_packages,
  })
}

source "vmware-iso" "golden-image" {
  vm_name       = local.vm_name
  version       = var.vmware_virtual_hardware_version
  guest_os_type = var.vmware_guest_os_type

  headless = true

  # Hardware configuration
  cpus                 = var.cpus
  cores                = var.cores
  memory               = var.memory
  disk_size            = var.disk_size
  network              = "nat"
  network_adapter_type = "VMXNET3"

  disk_adapter_type = "pvscsi"

  # ISO Configuration
  iso_url      = local.iso_url
  iso_checksum = local.iso_checksum

  # Http directory configuration
  http_content = {
    "/kickstart.ks" = local.kickstart,
  }

  # Communicator configuration
  communicator           = "ssh"
  ssh_username           = var.provisioner_username
  ssh_password           = var.provisioner_password
  ssh_timeout            = "60m"
  ssh_handshake_attempts = 10

  # Boot Configuration
  boot_wait    = "5s"
  boot_command = [
    "<up><tab> inst.text inst.ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/kickstart.ks<enter><wait><enter>",
  ]
  shutdown_command = "sudo /sbin/halt -h -p"

  # Export configuration
  format           = "ova"
  output_directory = "${local.export_path}/vmware"
}
{{< /highlight >}}

Most configuration parameters are self-explanatory, but I would like to specifically mention the `version` and `guest_os_type` parameters.

The `version` parameter defines the VMware VM Hardware Version — it determines the virtualization capabilities and depends on the hypervisor version used. Information on supported versions can be found in the article "[Virtual machine hardware versions](https://knowledge.broadcom.com/external/article?articleNumber=315655)".

You must also set the correct `guest_os_type`. This parameter defines the type of operating system inside the virtual machine and plays a crucial role in performance and the optimal operation of the hypervisor and VMware Tools. Available options can be found in the article "[Determine the guest OS from a VM configuration file](https://knowledge.broadcom.com/external/article?articleNumber=321876)".

### Kickstart

A [kickstart](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/automatically_installing_rhel/starting-kickstart-installations_rhel-installer) file is used for automated OS installation and configuration.

It handles:

* Disk partitioning;
* Setting up the timezone and time synchronization servers;
* Adding a user;
* Installing packages;
* Initial system hardening.

The file is templated using variables defined in the builder. The template file itself is placed in `rocky/templates/kickstart.pkrtpl`.

{{< highlight hcl "linenos=inline" >}}
###########################
# Kickstart Configuration #
###########################

cdrom

### Setup repositories
%{ for r in repo_list ~}
repo --name="${r.name}" --baseurl="${r.url}"
%{ endfor ~}

### Performs the kickstart installation in text mode
text

### System language
lang en_US.utf8

### Sets the default keyboard type
keyboard --vckeymap=us

### Configure network information for target system
network --bootproto=dhcp

### Configure firewall settings for the system
firewall --enabled

### Sets up the authentication options for the system. The SSDD profile sets sha512 to hash passwords. Passwords are shadowed by default
authselect select sssd

### Skip EULA (include this for non-interactive install of subcription based EL)
eula --agreed

### Do not configure the X Window System
skipx

### System services
services --enabled="NetworkManager,chronyd,sshd"

### System timezone with ntp servers
timezone ${guest_timezone} --utc --ntpservers=0.pool.ntp.org,1.pool.ntp.org,2.pool.ntp.org,3.pool.ntp.org

### Lock root - user will not be able to log in from the console
rootpw --lock

### Add a user that can login and escalate privileges.
user --name=${build_username} --plaintext --password=${build_password} --groups=wheel

### Ensure SELinux State is Enforcing
selinux --enforcing

##########################
###   Disk Partition   ###
##########################
clearpart --all --initlabel
part /boot --fstype xfs --size=512 --ondisk sda
part pv.01 --size=1 --grow --ondisk sda
volgroup vg_root pv.01

%{ for lv in lvm_layout ~}
logvol ${lv.mountpoint} --fstype ${lv.fstype} --vgname vg_root --name ${lv.name} --size=${lv.size} %{ if length(lv.fsoptions) > 0 }--fsoptions='${join(",", lv.fsoptions)}'%{ endif }
%{ endfor ~}


##########################
### Packages selection ###
##########################
%packages --ignoremissing --excludedocs
@core --nodefaults
%{ for package in install_packages ~}
${package}
%{ endfor ~}
%{ for package in uninstall_packages ~}
-${package}
%{ endfor ~}
%end

# Disable KDump Kernel Crash Analyzer
%addon com_redhat_kdump --disable
%end

########################
###   Post install   ###
########################
%post --log=/root/ks-post.log
#!/bin/bash

# CCE-83857-3, CCE-83881-3, CCE-83891-2 - Add noexec, nodev, nosuid options to /dev/shm
echo "none    /dev/shm    tmpfs   nosuid,nodev,noexec   0 0" >> /etc/fstab

echo "LANG=en_US.utf-8" >> /etc/environment
echo "LC_ALL=en_US.utf-8" >> /etc/environment

# Disable AllowZoneDrifting
replace_or_append '/etc/firewalld/firewalld.conf' '^AllowZoneDrifting' 'no' '' '%s=%s'
# Remove cockpit service
firewall-offline-cmd --remove-service=cockpit

echo "${build_username} ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers.d/${build_username}
sed -i "s/^.*requiretty/#Defaults requiretty/" /etc/sudoers

%end

# Reboot after install
reboot --eject
{{< /highlight >}}

### The Build Block

The `build` block defines which builders will be run, which provisioning scripts will be executed, and what post-processing operations will be applied.
In this scenario, we use only one builder - `vmware-iso.golden-image`.

Before exporting the image, an Ansible playbook is executed. It brings the virtual machine into compliance with the CIS Benchmark, cleans the image of unnecessary logs, and resets the machine ID.

We will place the configuration for this block in a separate file `rocky/build.pkr.hcl`:

{{< highlight hcl "linenos=inline" >}}
build {
  name    = "golden-image"
  sources = [
    "sources.vmware-iso.golden-image",
  ]
  
  provisioner "ansible" {
    playbook_file    = "${abspath(path.cwd)}/ansible/playbook.yaml"
    galaxy_file      = "${abspath(path.cwd)}/ansible/requirements.yaml"
    roles_path       = "${abspath(path.cwd)}/ansible/roles"
    collections_path = "${abspath(path.cwd)}/ansible/collections"
    sftp_command     = "/usr/libexec/openssh/sftp-server -e" 
    ansible_env_vars = [
      "ANSIBLE_CONFIG=${abspath(path.cwd)}/ansible/ansible.cfg",  
    ]
    extra_arguments = [
      "--scp-extra-args",
      "'-O'",
    ]
  }
}
{{< /highlight >}}

### Ansible Configuration

After Packer successfully installs the base OS using the Kickstart file and establishes an SSH connection, it hands over control to Ansible. At this stage, the final system configuration (provisioning) takes place.

As stated in the initial template requirements, we need to bring the system into compliance with CIS Benchmark policies and pre-install the Prometheus Node Exporter to collect metrics. Performing these tasks via bash scripts inside Kickstart is tedious and hard to maintain, which is why we delegate this to Ansible.

The main entry point for Packer is the `ansible/playbook.yaml` file. Here we apply the required roles:

{{< highlight yaml "linenos=inline" >}}
- name: Configure VM for CIS Benchmark
  hosts: all
  become: true
  gather_facts: true
  vars:
    __authselect_custom_profile_name: custom-profile
    __warning_banner: |
      -- WARNING --
      This system is for the use of authorized users only. Individuals
      using this computer system without authority or in excess of their
      authority are subject to having all their activities on this system
      monitored and recorded by system personnel. Anyone using this
      system expressly consents to such monitoring and is advised that
      if such monitoring reveals possible evidence of criminal activity
      system personal may provide the evidence of such monitoring to law
      enforcement officials.
    __auditd_max_log_file_action: rotate
    __syslog: journald
    __journald_systemmaxuse: 50M
    __journald_systemkeepfree: 500M
    __journald_runtimemaxuse: 50M
    __journald_runtimekeepfree: 500M
    __journald_maxfilesec: 5day
    __sudo_timestamp_timeout: 5
    __shell_session_timeout: 600
  tasks:
    - name: CIS Benchmark RHEL 8
      vars:
        # Disable rules
        rhel8cis_rule_1_2_1: false # Ensure GPG keys are configured
        rhel8cis_rule_1_3_1: false # Ensure bootloader password is set
        rhel8cis_rule_4_3_4: false # Ensure users must provide password for escalation
    
        rhel8cis_authselect_custom_profile_name: "{{ __authselect_custom_profile_name }}"
        rhel8cis_warning_banner: "{{ __warning_banner }}"
        rhel8cis_auditd_max_log_file_action: "{{ __auditd_max_log_file_action }}"
        rhel8cis_syslog: "{{ __syslog }}"
        rhel8cis_journald_systemmaxuse: "{{ __journald_systemmaxuse }}"
        rhel8cis_journald_systemkeepfree: "{{ __journald_systemkeepfree }}"
        rhel8cis_journald_runtimemaxuse: "{{ __journald_runtimemaxuse }}"
        rhel8cis_journald_runtimekeepfree: "{{ __journald_runtimekeepfree }}"
        rhel8cis_journald_maxfilesec: "{{ __journald_maxfilesec }}"
        rhel8cis_sudo_timestamp_timeout: "{{ __sudo_timestamp_timeout }}"
        rhel8cis_shell_session_timeout: "{{ __shell_session_timeout }}"
      when:
        - ansible_facts.os_family == 'RedHat' or ansible_facts.os_family == "Rocky"
        - ansible_facts.distribution_major_version is version_compare('8', '==')
      ansible.builtin.include_role:
          name: rhel8_cis

    - name: CIS Benchmark RHEL 9
      vars:
        # Disable rules
        rhel9cis_rule_1_1_1_9: false # Ensure unused filesystems kernel modules are not available
        rhel9cis_rule_1_2_1_1: false # Ensure GPG keys are configured
        rhel9cis_rule_1_4_1: false # Ensure bootloader password is set
        rhel9cis_rule_5_2_4: false # Ensure users must provide password for escalation

        rhel9cis_authselect_custom_profile_name: "{{ __authselect_custom_profile_name }}"
        rhel9cis_warning_banner: "{{ __warning_banner }}"
        rhel9cis_auditd_max_log_file_action: "{{ __auditd_max_log_file_action }}"
        rhel9cis_syslog: "{{ __syslog }}"
        rhel9cis_journald_systemmaxuse: "{{ __journald_systemmaxuse }}"
        rhel9cis_journald_systemkeepfree: "{{ __journald_systemkeepfree }}"
        rhel9cis_journald_runtimemaxuse: "{{ __journald_runtimemaxuse }}"
        rhel9cis_journald_runtimekeepfree: "{{ __journald_runtimekeepfree }}"
        rhel9cis_journald_maxfilesec: "{{ __journald_maxfilesec }}"
        rhel9cis_sudo_timestamp_timeout: "{{ __sudo_timestamp_timeout }}"
        rhel9cis_shell_session_timeout: "{{ __shell_session_timeout }}"
      when:
        - ansible_facts.os_family == 'RedHat' or ansible_facts.os_family == "Rocky"
        - ansible_facts.distribution_major_version is version_compare('9', '==')
      ansible.builtin.include_role:
        name: rhel9_cis

    - name: Install Prometheus Node Exporter
      hosts: all
      become: true
      gather_facts: true
      tags:
        - node-exporter
      tasks:
        - name: Install node exporter
          ansible.builtin.include_role:
            name: prometheus.prometheus.node_exporter
          vars:
            node_exporter_version: 1.10.2
            node_exporter_web_listen_address: "0.0.0.0:9100"
            node_exporter_enabled_collectors:
             - systemd
             - processes
      post_tasks:
        - name: Collect facts about system services
          ansible.builtin.service_facts:
          register: services_state
          changed_when: false

        - name: Open node_exporter port
          ansible.posix.firewalld:
          port: 9100/tcp
          zone: public
          permanent: true
          immediate: true
          state: enabled
          when:
          - services_state.ansible_facts.services['firewalld.service'].state == 'running'

    - name: Cleanup VM
      hosts: all
      become: true
      gather_facts: true
      roles:
        - cleanup
{{< /highlight >}}

### Variables Block

All variables that will be used to build the template are defined in the `rocky/variables.pkr.hcl` file. We provide default values for most variables. This allows us to start a build without having to define a huge number of variables, while still preserving the flexibility to customize the template.

{{< highlight hcl "linenos=inline" >}}
variable "rocky_version" {
  type    = string
  default = "9.7"
}

variable "rocky_repository_url" {
  description = "Repo list"
  type        = string
  default     = "https://dl.rockylinux.org/pub/rocky"
}

# See https://knowledge.broadcom.com/external/article?articleNumber=315655
variable "vmware_virtual_hardware_version" {
  description = "The virtual machine hardware version"
  type        = number
  default     = 21
}

# See https://knowledge.broadcom.com/external/article?articleNumber=321876
variable "vmware_guest_os_type" {
  description = "The guest operating system identifier for the virtual machine"
  type        = string
  default     = "rhel9_64Guest"
}

# Hardware configuration
variable "cpus" {
  description = "The number of virtual CPUs cores for the virtual machine"
  type        = number
  default     = 1
}

variable "cores" {
  description = "The number of virtual CPU cores per socket for the virtual machine"
  type        = number
  default     = 1
}

variable "memory" {
  description = "The amount of memory to use for building the VM in megabytes"
  type        = number
  default     = 2048
}

variable "disk_size" {
  description = "The size, in megabytes, of the hard disk to create for the VM"
  type        = number
  default     = 20480
}

variable "provisioner_username" {
  description = "Setup provisioner username"
  type        = string
  sensitive   = true
  default     = "packer"
}

variable "provisioner_password" {
  description = "Setup provisioner password"
  type        = string
  sensitive   = true
}

# Template configuration
variable "lvm_layout" {
  description = "Root disk LVM layout"
  type        = list(object({
    mountpoint = string
    name       = string
    size       = number
    fstype     = string
    grow       = bool
    fsoptions  = list(string)
  }))

  default = [
    {
      mountpoint : "/",
      name : "root",
      size : "5120",
      fstype : "xfs",
      grow : false,
      fsoptions : [],
    },

    {
      mountpoint : "swap",
      name : "swap",
      size : "1024",
      fstype : "swap",
      grow : false,
      fsoptions : [],
    },

    # CCE-83468-9 - Ensure /home Located On Separate Partition
    {
      mountpoint : "/home",
      name : "home",
      size : "1024",
      fstype : "xfs",
      grow : true,
      fsoptions : [
        # CCE-83871-4 - Add nodev Option to /home
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_home_nodev
        "nodev",
        # CCE-83894-6 - Add nosuid Option to /home
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_home_nosuid
        "nosuid",
      ],
    },

    # CCE-90845-9 - Ensure /tmp Located On Separate Partition
    # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_partition_for_tmp
    {
      mountpoint : "/tmp",
      name : "tmp",
      size : "2048",
      fstype : "xfs",
      grow : false,
      fsoptions : [
        # CCE-83869-8 - Add nodev Option to /tmp
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_tmp_nodev
        "nodev",
        # CCE-83872-2 - Add nosuid Option to /tmp
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_tmp_nosuid
        "nosuid",
        # CCE-83885-4 - Add noexec Option to /tmp
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_tmp_noexec
        "noexec",
      ],
    },

    # CCE-83466-3 - Ensure /var Located On Separate Partition
    # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_partition_for_var
    {
      mountpoint : "/var",
      name : "var",
      size : "2048",
      fstype : "xfs",
      grow : false,
      fsoptions : [
        # CCE-83868-0 - Add nodev Option to /var
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_var_nodev
        "nodev",
        # CCE-83865-6 - Add noexec Option to /var
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_var_noexec
        "noexec",
        # CCE-83867-2 - Add nosuid Option to /var
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_var_nosuid
        "nosuid"
      ],
    },

    # CCE-90848-3 - Ensure /var/log Located On Separate Partition
    # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_partition_for_var_log
    {
      mountpoint : "/var/log",
      name : "log",
      size : "4096",
      fstype : "xfs",
      grow : false,
      fsoptions : [
        # CCE-83886-2 - Add nodev Option to /var/log
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_var_log_nodev
        "nodev",
        #  CCE-83887-0 - Add noexec Option to /var/log
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_var_log_noexec
        "noexec",
        # CCE-83870-6 - Add nosuid Option to /var/log
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_var_log_nosuid
        "nosuid",
      ],
    },

    # CCE-90847-5 - Ensure /var/log/audit Located On Separate Partition
    # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_partition_for_var_log_audit
    {
      mountpoint : "/var/log/audit",
      name : "audit",
      size : "1024",
      fstype : "xfs",
      grow : false,
      fsoptions : [
        # CCE-83882-1 - Add nodev Option to /var/log/audit
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_var_log_audit_nodev
        "nodev",
        # CCE-83878-9 - Add noexec Option to /var/log/audit
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_var_log_audit_noexec
        "noexec",
        # CCE-83893-8 - Add nosuid Option to /var/log/audit
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_var_log_audit_nosuid
        "nosuid",
      ],
    },

    # CCE-83487-9 - Ensure /var/tmp Located On Separate Partition
    # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_partition_for_var_tmp
    {
      mountpoint : "/var/tmp",
      name : "vartmp",
      size : "2048",
      fstype : "xfs",
      grow : false,
      fsoptions : [
        # CCE-83864-9 - Add nodev Option to /var/tmp
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_var_tmp_nodev
        "nodev",
        # CCE-83866-4 - Add noexec Option to /var/tmp
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_var_tmp_noexec
        "noexec",
        # CCE-83863-1 - Add nosuid Option to /var/tmp
        # Ref: https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_mount_option_var_tmp_nosuid
        "nosuid",
      ],
    },
  ]
}

variable "install_packages" {
  type        = list(string)
  description = "Packages list ensure to be installed"
  default     = [
    "bash-completion",
    "tar",
    "vim",
    "htop",

    # VMware tools and dependencies
    "open-vm-tools",
    "perl",
    
    # CCE-90843-4 Install AIDE https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_aide_installed
    "aide",
    # CCE-83523-1 Install sudo Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_sudo_installed
    "sudo",
    # CCE-83649-4 Ensure the audit Subsystem is Installed https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_audit_installed
    "audit",
    # CCE-84063-7 Ensure rsyslog is Installed https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_rsyslog_installed
    "rsyslog",
    # CCE-84021-5 Install firewalld Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_firewalld_installed
    "firewalld",
    # CCE-84069-4 Install libselinux Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_libselinux_installed
    "libselinux",
  ]
}

variable "uninstall_packages" {
  type        = list(string)
  description = "Packages list ensure to be uninstalled"
  default     = [
    # unnecessary firmware
    "aic94xx-firmware",
    "atmel-firmware",
    "b43-openfwwf",
    "bfa-firmware",
    "ipw2100-firmware",
    "ipw2200-firmware",
    "ivtv-firmware",
    "iwl*-firmware",
    "libertas-usb8388-firmware",
    "ql*-firmware",
    "rt61pci-firmware",
    "rt73usb-firmware",
    "xorg-x11-drv-ati-firmware",
    "zd1211-firmware",
    "cockpit",
    "alsa-*",
    "fprintd-pam",
    "intltool",
    "microcode_ctl",

    # CCE-84072-8 Uninstall mcstrans Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_mcstrans_removed
    "mcstrans",
    # CCE-84073-6 Uninstall setroubleshoot Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_setroubleshoot_removed
    "setroubleshoot",
    # CCE-84159-3 Uninstall vsftpd Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_vsftpd_removed
    "vsftpd",
    # CCE-85974-4 Uninstall httpd Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_httpd_removed
    "httpd",
    # CCE-85977-7 Uninstall dovecot Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_dovecot_removed
    "dovecot",
    # CCE-90831-9 Ensure LDAP client is not installed https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_openldap-clients_removed
    "openldap-clients",
    # CCE-84155-1 Uninstall xinetd Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_xinetd_removed
    "xinetd",
    # CCE-84151-0 Remove NIS Client https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_ypbind_removed
    "ypbind",
    # CCE-84152-8 Uninstall ypserv Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_ypserv_removed
    "ypserv",
    # CCE-84142-9 Uninstall rsh Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_rsh_removed
    "rsh",
    # CCE-84157-7 Uninstall talk Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_talk_removed
    "talk",
    # CCE-84149-4 Uninstall telnet-server Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_telnet-server_removed
    "telnet-server",
    # CCE-84146-0 Remove telnet Clients https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_telnet_removed
    "telnet",
    # CCE-84154-4 Uninstall tftp-server Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_tftp-server_removed
    "tftp-server",
    # CCE-84153-6 Remove tftp Daemon https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_tftp_removed
    "tftp",
    # CCE-84238-5 Uninstall squid Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_squid_removed
    "squid",
    # CCE-85981-9 Uninstall net-snmp Package https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_net-snmp_removed
    "net-snmp",
    # CCE-84104-9 Remove the X Windows Package Group https://static.open-scap.org/ssg-guides/ssg-rhel9-guide-cis.html#xccdf_org.ssgproject.content_rule_package_xorg-x11-server-common_removed
    "xorg-x11-server-common",
  ]
}

variable "guest_timezone" {
  type    = string
  default = "UTC"
}
{{< /highlight >}}

## Running the Build

To start the build, you only need to pass the value of the `provisioner_password` variable:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/packer-templates" >}}
$ packer build \
    -var "provisioner_password=REPLACE_ME" \
    rocky
{{< /terminal >}}

This command will read all `*.hcl` files in the `rocky` directory and merge them into a single configuration.

After successful execution, the `artifacts` directory will contain the built Rocky 9.7 virtual machine image in `ova` format, which can be imported into VMware Cloud Director or launched in any other VMware hypervisor.

If you need to build a different version of Rocky Linux, simply change the value of the `rocky_version` variable and, if necessary, the `vmware_guest_os_type`. Example for Rocky 8.10:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/packer-templates" >}}
$ packer build \
    -var "provisioner_password=REPLACE_ME" \
    -var "rocky_version=8.10" \
    -var "vmware_guest_os_type=rhel8_64Guest" \
    rocky
{{< /terminal >}}

Older versions of Rocky are moved to the archive, so if we need to build, for example, Rocky 9.4, we also need to pass the archive repository address to the `rocky_repository_url` variable:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/packer-templates" >}}
$ packer build \
    -var "provisioner_password=REPLACE_ME" \
    -var "rocky_version=9.4" \
    -var "rocky_repository_url=https://dl.rockylinux.org/vault/rocky" \
    rocky
{{< /terminal >}}

The full source code for this example is available in the [repository](https://github.com/zt-sv/packer-templates).

# What's Next?

This example allows building various versions of Rocky Linux and ensuring their compliance with the CIS Benchmark.
If a new version of the CIS Benchmark or a new version of Rocky Linux is released, you will not have to rebuild the template from scratch. It will be enough to make changes to the template code and run Packer again.

Building upon this foundation, you can:
* Build images for other hypervisors;
* Perform subsequent packaging of templates into Vagrant Boxes;
* Scale the approach to other operating systems;
* Add various software to the template via Ansible.

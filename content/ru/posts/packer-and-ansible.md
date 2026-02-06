+++
date = '2026-02-04T17:57:16+03:00'
draft = true
title = 'Сборка базового образа виртуальной машины при помощи Packer и Ansible'
keywords = [
"Packer",
"Ansible",
"VMWare",
"Virtualbox",
"Rocky",
"Linux",
"Vagrant",
"Hashicorp",
"CIS Benchmark"
]
description = 'Как автоматизировать сборку образа виртуальной машины на примере Rocky Linux с использованием Packer и Ansible для VMware с учётом CIS Benchmark'
images = ['author.png']
toc = true
+++

Представьте ситуацию: вам нужно развернуть новую виртуальную машину для разработки или тестирования. Вы начинаете с
чистого образа дистрибутива, устанавливаете необходимые пакеты, настраиваете сеть, применяете политики безопасности -
это займет большое количество времени. Выполнить такую операцию единожды можно. Но что если одинаковых машин нужно
много? А если спустя время в первоначальной конфигурации нужно внести небольшие изменения?
Ответы на эти вопросы дают Packer и Ansible.

HashiCorp Packer — инструмент для автоматизированной сборки образов виртуальных машин. Вместо того чтобы вручную
создавать и конфигурировать ВМ, весь процесс описывается кодом.
Ansible, в контексте Packer, будет поставлять и конфигурировать программное обеспечение в создаваемых шаблонах.

В данном гайде я рассмотрю несколько вариантов сборки виртуальных машин, постепенно увеличивая требования и сложность.
Для повторения примеров вам понадобятся установленные инструменты:

* [Hashicorp Packer](https://developer.hashicorp.com/packer/install);
* [Hashicorp Vagrant](https://developer.hashicorp.com/vagrant/install);
* [Ansible](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html) - рекомендую
  устанавливать через `pip` в изолированный `venv`;
* Один из [VMWare desktop hypervisors](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion);
* [VMware OVFTool](https://customerconnect.vmware.com/downloads/get-download?downloadGroup=OVFTOOL441);
* [VirtualBox](https://www.virtualbox.org/manual/topics/installation.html#installation);
* [Python](https://www.python.org/downloads/).

## Rocky Linux для VMWare

Представим что поставлена задача: необходимо подготовить базовый образ на Rocky Linux 9, из которого будут
разворачиваться
виртуальные машины в частном облаке на базе VMWare Cloud Director (VCD).

Требования к шаблону:

* соответствие [CIS Benchmark](https://www.cisecurity.org/cis-benchmarks);
* предустановленный [Prometheus Node Exporter](https://prometheus.io/docs/guides/node-exporter/);
* возможность повторяемой сборки и версионирования.

Создаем структуру проекта:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv" >}}
$ tree packer-templates 
packer-templates
├── ansible
│   ├── collections # Ansible коллекции
│   ├── roles # Ansible роли
│   ├── ansible.cfg # конфигурация Ansible
│   ├── playbook.yaml
│   └── requirements.yaml # зависимости Ansible
├── artifacts # директория для собранных виртуальных машин
├── requirements.txt # Python зависимости
└── rocky
    ├── build.pkr.hcl # блоки build
    ├── packer.pkr.hcl # блок зависимостей Packer
    ├── source.pkr.hcl # конфигурация билдеров
    └── variables.pkr.hcl # описание переменных
{{< /terminal >}}

### Установка Ansible

Ansible удобнее всего устанавливать через pip в изолированное виртуальное окружение проекта.

Если планируется сборка образов Rocky 8, необходимо учитывать, что в этой версии используется Python 3.6, который
несовместим с новыми версиями [ansible-core](https://pypi.org/project/ansible-core). В этом случае следует использовать
Ansible версии < 2.17.

Версию Ansible фиксируем в файле `requirements.txt`:

```requirements.txt
ansible-core==2.16.16
```

Создаем и активируем новый `venv`:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/packer-templates" >}}
$ python3 -m venv $(pwd)/venv  
$ source $(pwd)/venv/bin/activate
{{< /terminal >}}

И устанавливаем зависимости:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/packer-templates" >}}
$ pip3 install -r requirements.txt
{{< /terminal >}}

Так как планируется сборка несколько образов на разных версиях ОС, то нужно убедиться, что Ansible сохраняет факты в
память, а не на диск. В противном случае возможно повторное использование кешированных фактов от другой системы, что
приведёт к ошибкам.

Для этого в `ansible.cfg` указываем:

{{< highlight cfg "linenos=inline" >}}
[defaults]
gathering = smart  
fact_caching = memory
{{< /highlight >}}

### Шаблон Packer

При запуске команды `packer build` Packer считывает все `*.pkr.hcl` файлы в директории запуска и объединяет их в единую
конфигурацию. Это позволяет разделить конфигурацию на файлы, для повышения читаемости и дальнейшего сопровождения.
Каждый шаблон Packer состоит как минимум из трех блоков:

* `packer` - блок конфигурации Packer и зависимостей;
* `source` - определяет конфигурацию билдеров образов;
* `build` - определяет какие именно сборщики должны быть запущены, как должны быть сконфигурированы и какие действия
  после сборки должны быть выполнены для получения конечного образа.

#### Определение и установка зависимостей Packer

Для сборки шаблона потребуются плагины:

* [VMware](https://developer.hashicorp.com/packer/integrations/vmware/vmware/) - билдеры для гипервизиоров VMWare;
* [Ansible](https://developer.hashicorp.com/packer/integrations/hashicorp/ansible) - запуск Ansible плейбуков из Packer;
* [Git](https://developer.hashicorp.com/packer/integrations/ethanmdavidson/git) - получение информации из git.

Добавляем их в файл `rocky/packer.pkr.hcl` и фиксируем текущие версии:

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

Для установки плагинов выполняем команду [`init`](https://developer.hashicorp.com/packer/docs/commands/init) в
директории с шаблоном:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/packer-templates" >}}
$ packer init rocky
{{< /terminal >}}

#### VMWare ISO Builder

Виртуальную машину будет создаваться из ISO-файла, поэтому в качестве билдера
используется [VMware ISO](https://developer.hashicorp.com/packer/integrations/hashicorp/vmware/latest/components/builder/iso).
С его помощью будет создан образ в `ova` формате, который можно импортировать в VCD.

Для версионирования образа используем текущую дату и hash коммита.

Всю конфигурацию билдера опишем в файле `rocky/source.pkr.hcl`.

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

Большая часть параметров конфигурации не требует дополнительных объяснений, но я бы хотел отдельно упомянуть
параметры `version` и `guest_os_type`.

Параметр `version` определяет VMware VM Hardware Version - он необходим для определения возможностей виртуализации и
зависит от версии используемого гипервизора. Информацию о поддерживаемых версиях можно найти в
статье "[Virtual machine hardware versions](https://knowledge.broadcom.com/external/article?articleNumber=315655)".

Так же необходимо выставить корректный `guest_os_type`. Данный параметр определяет тип операционной системы внутри
виртуальной машины и играет важную роль в производительности и оптимальной работе гипервизора и VMWare Tools с такой
виртуальной машиной. Доступные варианты можно найти
статье "[Determine the guest OS from a VM configuration file](https://knowledge.broadcom.com/external/article?articleNumber=321876)".

#### Kickstart

Для автоматической установки и конфигурации ОС
используется [kickstart](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/automatically_installing_rhel/starting-kickstart-installations_rhel-installer)
файл.

С его помощью выполняются:

* разметка диска;
* настройка таймзоны и серверов синхронизации времени;
* добавление пользователя;
* установка пакетов;
* первичный hardening системы.

Файл шаблонизируется при помощи переменных, которые заданы в билдере. Сам файл шаблона размещаем в
файле `rocky/templates/kickstart.pkrtpl`.

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
echo "none    /dev/shm    tmpfs   nosuid,nodev,noexec	  0 0" >> /etc/fstab

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

#### Блок build

Определение какие именно билдеры должны быть запущены, а так же provision скрипты и post-processing операции, которые
должны быть выполнены - все это описывается в блоке конфигурации `build`.
В данном сценарии используется всего один билдер - `vmware-iso.golden-image`.

Перед экспортом образа выполняется Ansible плейбук. При его помощи виртуальная машина будет
приводиться в соответствие с CIS Benchmark, будет выполнена чистка образа от лишних логов и сброс идентификатора
машины.

Конфигурацию данного блока вынесем в отдельный файл `rocky/build.pkr.hcl`:

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

#### Блок переменных

Все переменные, которые будут использованы для сборки шаблона, определяем в файле `rocky/variables.pkr.hcl`. Большей
части переменных задаем значение по-умолчанию. Это позволит запускать сборку, не определяя огромное количество переменных. При этом остается возможность гибкой настройки шаблона.

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

### Запуск сборки

Для запуска сборки достаточно только передать значение переменной `provisioner_password`:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/packer-templates" >}}
$ packer build \
    -var "provisioner_password=123" \
    rocky
{{< /terminal >}}

Данная команда прочитает все `*.hcl` файлы в директории `rocky` и склеит их в единую конфигурацию.

После успешного выполнения в директории `artifacts` будет собранный образ виртуальной машины на Rocky 9.7 в
формате `ova`, который можно импортировать в VMWare Cloud Director или запустить в любом другом гипервизоре VMWare.

Если требуется собрать другую версию Rocky, достаточно изменить значение переменной `rocky_version` и, при необходимости,
`vmware_guest_os_type`. Пример для Rocky 8.10:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/packer-templates" >}}
$ packer build \
    -var "provisioner_password=123" \
    -var "rocky_version=8.10" \
    -var "vmware_guest_os_type=rhel8_64Guest" \
    rocky
{{< /terminal >}}

Полный код данного примера доступен в [репозитории](https://github.com/zt-sv/packer-templates).

---
title: Setting Up a Linux Kernel Development Environment with QEMU and libvirt
description: Notes and lessons learned while setting up a Linux Kernel development environment using QEMU and libvirt.
date: 2026-03-13 15:25:00 +/-TTTT
categories: [Kernel Development]
tags: [kernel, linux, ubuntu, qemu, virtualization]
author: <author_id> # for single entry
---

# QEMU + libvirt setup (ordem de execução)

## 1. Instalar dependências

### Debian/Ubuntu
sudo apt update
sudo apt install qemu-system libvirt-daemon-system virtinst libguestfs-tools wget

### Arch
sudo pacman -Syy
sudo pacman -S qemu-full libvirt virt-install guestfs-tools wget

### Fedora
sudo dnf update
sudo dnf install qemu libvirt-daemon virt-install guestfs-tools wget


## 2. Criar diretório base e permissões

sudo mkdir /home/lk_dev
sudo chown -R libvirt-qemu:libvirt-qemu /home/lk_dev
sudo chmod -R 2770 /home/lk_dev
sudo usermod -aG libvirt-qemu "$USER"

# logout/login depois disso


## 3. Criar script de ambiente

touch /home/lk_dev/activate.sh
chmod +x /home/lk_dev/activate.sh


## 4. Criar diretório da VM

mkdir "${LK_DEV_DIR}/vm"


## 5. Baixar imagem do sistema

wget --directory-prefix="${VM_DIR}" http://cdimage.debian.org/cdimage/cloud/bookworm/daily/20250217-2026/debian-12-nocloud-arm64-daily-20250217-2026.qcow2

mv "${VM_DIR}/debian-12-nocloud-arm64-daily-20250217-2026.qcow2" "${VM_DIR}/base_arm64_img.qcow2"


## 6. Copiar kernel e initrd

virt-copy-out --add "${VM_DIR}/arm64_img.qcow2" /boot/<kernel> "$BOOT_DIR"
virt-copy-out --add "${VM_DIR}/arm64_img.qcow2" /boot/<initrd> "$BOOT_DIR"


## 7. Iniciar libvirt

sudo systemctl start libvirtd
systemctl status libvirtd

# opcional
sudo systemctl enable libvirtd


## 8. Iniciar rede do libvirt

sudo virsh net-start default

# opcional
sudo virsh net-autostart default


## 9. Comandos úteis do virsh

sudo virsh list --all
sudo virsh dominfo arm64
sudo virsh start --console arm64
sudo virsh shutdown arm64
sudo virsh destroy arm64
sudo virsh undefine arm64


## 10. Configurar SSH dentro da VM

# editar dentro da VM:
/etc/ssh/sshd_config

PermitRootLogin yes
PermitEmptyPasswords yes

# dentro da VM:
dpkg-reconfigure openssh-server
systemctl restart sshd
systemctl status sshd


## 11. Descobrir IP da VM (host)

sudo virsh net-dhcp-leases default


## 12. Acessar via SSH

ssh root@<VM-IP-address>


## 13. Transferência de arquivos

scp root@<VM-IP-address>:/root/foo /tmp/bar
scp /tmp/bar root@<VM-IP-address>:/root/foo


## 14. Listar módulos do kernel

# dentro da VM
lsmod > vm_mod_list


## 15. Editar VM (opcional - compartilhamento)

sudo EDITOR=vim
sudo virsh edit iio-arm64
# TP1 Provisionnement programmable

L'objectif de ce projet est d'automatiser la crÃ©ation de machines virtuelles. Je vais utiliser Virtualbox comme hyperviseur et Vagrant pour piloter l'hyperviseur.

## 1. Une premiÃ¨re VM

### DÃ©marrage facile

Pour pouvoir utiliser Vagrant, je vais crÃ©er un rÃ©pertoire pour chaque nouvelle machine virtuelle. Je vais utiliser Rocky 9 comme systÃ¨me d'exploitation.

```console
$ mkdir vagrant0
$ cd vagrant0/

$ cat Vagrantfile
```

```
Vagrant.configure("2") do |config|
  config.vm.box = "generic/rocky9"
end
```

Voici Ã  quoi ressemble un Vagrantfile. Avec ce fichier, je suis capable de lancer une VM simple sans aucune configuration.

Pour le faire :
```console
$ vagrant up
```

Pour se connecter :
```console
$ vagrant ssh
```

Pour dÃ©truire la machine :
```console
$ vagrant halt
$ vagrant destroy -f
```

### Un peu de configuration

Ce nouveau Vagrantfile va :
- Attribuer une adresse IP
- Attribuer un nom d'hÃ´te
- Changer le nom de la machine Vagrant
- Ajouter 2 Go de RAM
- Avoir une taille de disque de 20 Go

```
Vagrant.configure("2") do |config|
  config.vm.box = "generic/rocky9"
  config.vm.hostname = "ezconf.tp1.efrei"
  config.vm.network "private_network", ip: "10.1.1.11"
  config.vm.disk :disk, size: "20GB", primary: true
  config.vm.provider "virtualbox" do |vb|
    vb.name = "machine1.tp1.efrei"
    vb.memory = 2048
  end
end
```

Voici les rÃ©sultats :

```console
[vagrant@cauchemar ~]$ ip a
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:03:97:6a brd ff:ff:ff:ff:ff:ff
    altname enp0s8
    inet 10.1.1.11/24 brd 10.1.1.255 scope global noprefixroute eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe03:976a/64 scope link 
       valid_lft forever preferred_lft forever

[vagrant@cauchemar ~]$ cat /proc/meminfo
MemTotal:        2002653 kB
```

En modifiant la configuration par dÃ©faut de mon Vagrantfile, je peux crÃ©er des machines virtuelles personnalisÃ©es.

## 2. Script d'initialisation

Le Vagrantfile suivant fait rÃ©fÃ©rence Ã  une VM Ã  laquelle a Ã©tÃ© attribuÃ© un script bash Ã  exÃ©cuter au dÃ©marrage.

```
Vagrant.configure("2") do |config|
  config.vm.box = "generic/rocky9"
  config.vm.hostname = "machine1.tp1.efrei"
  config.vm.provision "shell", path: "script.sh"
  config.vm.provider "virtualbox" do |vb|
    vb.name = "machine1.tp1.efrei"
    vb.memory = 2048
  end
end
```

Mon script.sh ressemble Ã  ceci :

```bash
#!/bin/bash

sudo dnf install -y vim
sudo dnf install -y python3
sudo dnf update -y
```

Ce script va installer vim, python3, puis mettre Ã  jour la machine :

```console
$ vagrant up
$ vagrant ssh

[vagrant@cauchemar ~]$ cat bonjour cauchemar
bonjour cauchemar

[vagrant@cauchemar ~]$ python3
Python 3.9.18 (main, Jan  4 2024, 00:00:00) 
[GCC 11.4.1 20230605 (Red Hat 11.4.1-2)] on linux
Type "help", "copyright", "credits" or "license" for more information.
```

## 3. Packager

Packager signifie crÃ©er une image personnalisÃ©e Ã  partir d'une VM crÃ©Ã©e. Cela signifie que nous pouvons stocker une configuration dÃ©sirÃ©e et crÃ©er toutes les VM que nous voulons.

```console
$ vagrant package --output custom-efrei.box

$ vagrant box add custom-efrei custom-efrei.box

$ vagrant box list
custom-efrei   (virtualbox, 0)
```

Et pour packager cette machine dans un nouveau dossier :

```console
$ vagrant init custom-efrei
```

## 4. Plusieurs VM

Nous pouvons crÃ©er autant de machines que nous le souhaitons avec un seul Vagrantfile. Ici, je lance deux machines sur le mÃªme rÃ©seau, avec des noms d'hÃ´te et des mÃ©moires RAM diffÃ©rents :

```
Vagrant.configure("2") do |config|
  config.vm.define "machine1" do |m1|
    m1.vm.box = "generic/rocky9"
    m1.vm.hostname = "machine1.tp1.efrei"

    m1.vm.network :private_network, ip: "10.1.1.101"

    m1.vm.provider :virtualbox do |v1|
      v1.memory = 2048
      v1.customize ["modifyvm", :id, "--name", "machine.tp1.efrei"]
    end
  end
  config.vm.define "machine2" do |m2|
    m2.vm.box = "generic/rocky9"
    m2.vm.hostname = 'machine2.tp2.efrei'

    m2.vm.network :private_network, ip: "10.1.1.102"

    m2.vm.provider :virtualbox do |v2|
      v2.memory = 1024
      v2.name = "machine2.tp1.efrei"
    end
  end
end
```

Une fois mes machines dÃ©marrÃ©es, je vais vÃ©rifier que ces machines sont sur le mÃªme rÃ©seau.

```console
$ vagrant ssh machine2

[vagrant@machine2 ~]$ ping 10.1.1.102
PING 10.1.1.102 (10.1.1.102) 56(84) bytes of data.
64 bytes from 10.1.1.102: icmp_seq=1 ttl=64 time=0.200 ms
```

## 5. Cloud-init

Cloud-init est utilisÃ© pour effectuer une configuration automatique d'une VM dÃ¨s son dÃ©marrage. Je vais lancer une VM et installer le service *cloud-init* :

```console
[vagrant@machine2 ~]$ sudo dnf -y install cloud-init
[vagrant@machine2 ~]$ sudo systemctl enable cloud-init
```

Ensuite, la machine sera repackagÃ©e :

```console
$ vagrant box list
cloud-efrei    (virtualbox, 0)
```

Pour crÃ©er une nouvelle VM avec une configuration *cloud-init* spÃ©cifique, il existe deux mÃ©thodes :
- GÃ©nÃ©rer manuellement le fichier ISO :

```console
genisoimage -output cloud-init.iso -volid cidata -joliet -r meta-data user-data
```

Cette commande fait rÃ©fÃ©rence aux fichiers meta-data :

```cloud-init
#cloud-init
local-hostname: cloud-init.tp1.efrei
```

Et Ã  user-data :
```cloud-init
#cloud-config
users:
  - name: fils
    primary_group: fils
    groups: teiko 
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
    passwd: $5$ajra8rB1mQPgIqhh$i4ID0xdshbdolkshhsin8bHOy4fs3MVJBU.TTO9aRzmJKbeZWtDwlyqsjbziu5dld85prMoZj.XfrcEWVhe1ZEcM68d1
    ssh_authorized_keys:
      - ssh-ed28949 AAAAC3NzaC1l3R4CNTE5xttAIMO/JQ3AtA3k8iXJWlkdQJOHDh215OKyLR0vauzD7BgA 

```

Enfin, mon Vagrantfile ressemblera Ã  ceci :

```
Vagrant.configure("2") do |config|
  config.vm.box = "cloud-efrei"
  config.vm.disk :dvd, name: "installer", file: "./cloud-init.iso"
end
```

- Donner Ã  Vagrant un fichier cloud-init.yml

Mon fichier de donnÃ©es utilisateur ressemble Ã  ceci :

```cloud-init
#cloud-config
users:
  - name: fils
    cauch: Super adminsys
    primary_group: fils
    groups: teiko
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
    passwd: $5$ajra8rB1mQPgIqhh$i4ID0xdshbdolkshhsin8bHOy4fs3MVJBU.TTO9aRzmJKbeZWtDwlyqsjbziu5dld85prMoZj.XfrcEWVhe1ZEcM68d1
    ssh_authorized_keys:
      - ssh-ed28949 AAAAC3NzaC1l3R4CNTE5xttAIMO/JQ3AtA3k8iXJWlkdQJOHDh215OKyLR0vauzD7BgA 
```

Et mon Vagrantfile :

```
Vagrant.configure("2") do |config|
  config.vm.box = "test"
  config.vm.cloud_init :user_data, content_type: "text/cloud-config", path: "user_data.yml"
end
```

Pour vÃ©rifier que cela fonctionne, j'ai entrÃ© dans ma machine et recherchÃ© le nom d'utilisateur fils dans le fichier */etc/passwd* :

```console
$ vagrant ssh

[vagrant@rocky9 ~]$ cat /etc/passwd
fils:x:1001:1001:Super adminsys:/home/fils:/bin/bash

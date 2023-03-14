#ДЗ: Подготовить ISO образ с актуальным ядром. Успешено развернуть его через Vargant. Настроить VirtualBox Shared Folders

#1. /Vagrantfile
#Vagrantfile содержит параметры разворачиваемой вагрантом ВМ из собранного пакером образа CentOS8

MACHINES = {
#Указываем имя ВМ "su"
  :"su" => {
#Какой vm box будем использовать https://app.vagrantup.com/sarmat000/boxes/centos8-kernel6.2
              :box_name => "sarmat000/centos8-kernel6.2",
#Указываем box_version
              :box_version => "1.1",
#Указываем количество ядер ВМ
              :cpus => 2,
#Указываем количество ОЗУ в мегабайтах
              :memory => 1024,
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
#Таймаут ожидани подключения к серверу по SSH
    config.vm.boot_timeout = 4000
#Отключаем обновления vbguest
    config.vbguest.auto_update = false
# Включаем проброс общей папки в ВМ. Метод проброса sshfs согласно рекомендациям из https://blog.centos.org/2019/07/updated-centos-vagrant-images-available-v1905-01/
# Предварительно должен быть удановлен плагин
# vagrant plugin install vagrant-sshfs
# Дополнительнная инфомрация https://github.com/dustymabe/vagrant-sshfs
    config.vm.synced_folder ".", "/vagrant", type: "sshfs", sshfs_opts_append: "-o cache=no"
# Применяем конфигурацию ВМ
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s
      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
#Раскоменитровать если требуется увидеть процесс разворачивания ВМ
#      config.vm.provider :virtualbox do |vb|
#      vb.gui = true
#      end
    end
  end
end

#2. /packer/centos.json
#centos.json содержит информацию о характеристиках ВМ, с которой пакетр подготовит ISO образ. Образ можно залить в Vagrant cloud и использовать для разворачивания ВМ с уже выполнеными настройками, например обновленым ядром.


#Основная секция, в ней указываются характеристики нашей ВМ

{
    "builders": [
    {
#Указываем ссылку на файл автоматической конфигурации 
      "boot_command": [
        "<up><tab> inst.text inst.ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/ks.cfg<enter><wait>"
      ],
      "boot_wait": "10s",
#Указываем размер диска для ВМ
      "disk_size": "10240",
      "export_opts": [
        "--manifest",
        "--vsys",
        "0",
        "--description",
        "CentOS Stream 8 with kernel 6.x",
        "--version",
        "8"
      ],
#Указываем семейство ОС нашей ВМ
      "guest_os_type": "RedHat_64",
#Указываем каталог, из которого возьмём файл автоматической конфигурации
      "http_directory": "http",
#Контрольная сумма ISO-файла Проверяется после скачивания файла
      "iso_checksum": "b4bb35e2c074b4b9710419a9baa4283ce4a02f27d5b81bb8a714b576e5c2df7a",
#Ссылка на дистрибутив из которого будет разворачиваться наша ВМ
      "iso_url": "http://mirror.linux-ia64.org/centos/8-stream/isos/x86_64/CentOS-Stream-8-x86_64-20230209-boot.iso",
 #Hostname нашей ВМ
      "name": "centos-8",
      "output_directory": "builds",
      "shutdown_command": "echo 'vagrant' | sudo -S /sbin/halt -h -p",
      "shutdown_timeout": "5m",
#Пароль пользователя
      "ssh_password": "vagrant",
#Номер ssh-порта
      "ssh_port": 22,
      "ssh_pty": true,
#Тайм-аут подключения по SSH
#Если через 20 минут не получается подключиться, то сборка отменяется
      "ssh_timeout": "20m",
#Имя пользователя
      "ssh_username": "vagrant",
#Тип созданного образа (Для VirtualBox)
      "type": "virtualbox-iso",
#Параметры ВМ
#2 CPU и 1Гб ОЗУ
      "vboxmanage": [
        [
          "modifyvm",
          "{{.Name}}",
          "--memory",
          "1024"
        ],
        [
          "modifyvm",
          "{{.Name}}",
          "--cpus",
          "2"
        ]
      ],
#Имя ВМ в VirtualBox
      "vm_name": "packer-centos-vm"
    }
  ],
  "post-processors": [
    {
#Уровень сжатия
      "compression_level": "7",
#Указание пути для сохранения образа
#Будет сохранён в каталог packer
      "output": "centos-8-kernel-6-x86_64-Minimal.box",
      "type": "vagrant"
    }
  ],
#Настройка ВМ после установки
  "provisioners": [
    {
      "execute_command": "{{.Vars}}echo 'vagrant' | sudo -S -E bash '{{.Path}}'",
      "expect_disconnect": true,
      "override": {
        "centos-8": {
          "scripts": [
#Скрипты, которые будут запущены после установки ОС
#Скрипты выполняются в указанном порядке
            "scripts/stage-1-kernel-update.sh",
            "scripts/stage-2-clean.sh"
          ]
        }
      },
#Тайм-аут запуска скриптов, после того, как подключились по SSH
      "pause_before": "20s",
      "start_retry_timeout": "1m",
      "type": "shell"
    }
  ]
}

3. /packer/http/ks.cfg
#ks.cfg файл автоматической конфигурации ОС, применяется в json файле из пункта 2
#Включаем игнорирование недостающих пакетов если таковые есть https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/installation_guide/s1-kickstart2-packageselection
%packages --ignoremissing
# dnf group info minimal-environment
@^minimal-environment
%end
# Документация https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/installation_guide/s1-kickstart2-options
# Подтверждаем лицензионное соглашение
eula --agreed
# Указываем язык нашей ОС
lang en_US.UTF-8
# Раскладка клавиатуры
keyboard us
# Указываем часовой пояс
timezone UTC
# Включаем сетевой интерфейс и получаем ip-адрес по DHCP, задаем hostname ВМ
network --bootproto=dhcp --noipv6 --onboot=on --activate --device=link --hostname=otus-c8
# Задаём hostname otus-c8
rootpw vagrant
# Указываем пароль root пользователя
authselect --useshadow --passalgo=sha512
# Создаём пользователя vagrant, добавляем его в группу Wheel
user --groups=wheel --name=vagrant --password=vagrant --gecos="vagrant"
# Включаем SELinux в режиме enforcing
selinux --enforcing
# Выключаем штатный межсетевой экран
firewall --disabled
firstboot --disabled
# Выбираем установку в режиме командной строки
text
# Указываем адрес, с которого установщик возьмёт недостающие компоненты
url --url="http://mirror.centos.org/centos/8-stream/BaseOS/x86_64/os/"
# System bootloader configuration
bootloader --location=mbr --append="crashkernel=auto"
skipx
logging --level=info
zerombr
clearpart --all --initlabel
# Автоматически размечаем диск, создаём LVM
autopart --type=lvm
# Перезагрузка после установки
reboot

4. /packer/scripts
#Скрипты, которые применятся в пунте 2 для настройки ОС после ее установки
#stage-1-kernel-update.sh скрипт обнволения ядра
#!/bin/bash
# Установка репозитория elrepo
sudo yum install -y https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm 
# Установка нового ядра из репозитория elrepo-kernel
yum --enablerepo elrepo-kernel install kernel-ml -y
# Обновление параметров GRUB
grub2-mkconfig -o /boot/grub2/grub.cfg
grub2-set-default 0
echo "Grub update done."
# Перезагрузка ВМ
shutdown -r now


#stage-2-clean.sh настройка ключей vagrant и прав доступа
#!/bin/bash
# Обновление и очистка всех ненужных пакетов
yum update -y
yum clean all
# Добавление ssh-ключа и права для пользователя vagrant
mkdir -pm 700 /home/vagrant/.ssh
curl -sL https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub -o /home/vagrant/.ssh/authorized_keys
chmod 0600 /home/vagrant/.ssh/authorized_keys
chown -R vagrant:vagrant /home/vagrant/.ssh
echo %vagrant ALL=NOPASSWD:ALL > /etc/sudoers.d/vagrant
chmod 0440 /etc/sudoers.d/vagrant
# Удаление временных файлов
rm -rf /tmp/*
rm  -f /var/log/wtmp /var/log/btmp
rm -rf /var/cache/* /usr/share/doc/*
rm -rf /var/cache/yum
rm -rf /vagrant/home/*.iso
rm  -f ~/.bash_history
history -c

rm -rf /run/log/journal/*
sync
grub2-set-default 0
echo "###   Hi from second stage" >> /boot/grub2/grub.cfg

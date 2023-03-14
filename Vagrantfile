# -*- mode: ruby -*-
# vi: set ft=ruby :
MACHINES = {
  :"su" => {
              :box_name => "sarmat000/centos8-kernel6.2",
              :box_version => "1.1",
              :cpus => 2,
              :memory => 1024,
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.boot_timeout = 4000
    config.vbguest.auto_update = false
    config.vm.synced_folder ".", "/vagrant", type: "sshfs", sshfs_opts_append: "-o cache=no"
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s
      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
#      config.vm.provider :virtualbox do |vb|
#      vb.gui = true
#      end
    end
  end
end

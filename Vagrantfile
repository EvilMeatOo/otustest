# -*- mode: ruby -*-
# vi: set ft=ruby :
MACHINES = {
  :"su" => {
              :box_name => "centos/stream8",
              :box_version => "20210210.0",
              :cpus => 2,
              :memory => 1024,
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.synced_folder ".", "/vagrant", disabled: true
    if Vagrant.has_plugin?("vagrant-vbguest") then
      config.vbguest.auto_update = false
    end
    config.vm.boot_timeout = 4000000
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s
      box.vbguest.installer_options = { allow_kernel_upgrade: true }
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

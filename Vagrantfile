# -*- mode: ruby -*-
# vi: set ft=ruby :
#

VAGRANTFILE_API_VERSION = "2"

os = ENV['OS'] || 'centos'
if os == 'fedora'
  box_name = 'fedora/30-cloud-base'
else
  box_name = 'centos/7'
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    config.vm.box = box_name
    config.vm.box_check_update = false

    # Tweak VirtualBox configuration for GUI applications
    config.vm.provider :virtualbox do |v|
        v.name = os
        v.memory = 2048
        v.cpus = 2
    end

    config.vm.define :control, primary: true do |control|
      control.vm.network "private_network", ip: "192.168.111.1"
      control.vm.hostname = "control.kata"
      control.vm.synced_folder "./control", "/vagrant"
    end
  
    config.vm.define :target do |target|
      target.vm.network "private_network", ip: "192.168.111.2"
      target.vm.hostname = "target.kata"
      target.vm.synced_folder "./target", "/vagrant"
    end
  
    config.vm.provision :ansible_local do |ansible|
      ansible.playbook="setup.yml"
    end
  
    config.ssh.insert_key = false
  end
  
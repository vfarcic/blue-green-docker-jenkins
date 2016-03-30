# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.synced_folder ".", "/vagrant"
  # config.vm.synced_folder ".", "/vagrant", mount_options: ["dmode=700,fmode=600"]
  config.vm.define "swarm-master" do |d|
    d.vm.box = "ubuntu/wily64"
    d.vm.hostname = "swarm-master"
    d.vm.network "private_network", ip: "10.100.192.200"
    d.vm.provider "virtualbox" do |v|
      v.memory = 2048
    end
    d.vm.provision :shell, path: "bootstrap_ansible.sh"
  end
  (1..2).each do |i|
    config.vm.define "swarm-node-#{i}" do |d|
      d.vm.box = "ubuntu/wily64"
      d.vm.hostname = "swarm-node-#{i}"
      d.vm.network "private_network", ip: "10.100.192.20#{i}"
      d.vm.provider "virtualbox" do |v|
        v.memory = 1024
      end
    end
  end
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end
end

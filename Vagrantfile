# -*- mode: ruby -*-
# vi: set ft=ruby :

# Drew's note: This is a Vagrantfile used to stand up a Kubernetes cluster for
# local experimentation and learning. To find out more, see this repository:
#
# https://github.com/hybby/cheapk8s

# -----------------------------------------------------
# CUSTOMISE THE BELOW IF REQUIRED
# -----------------------------------------------------
windows_username = `powershell.exe '$env:UserName'`.strip

box = "ubuntu/focal64"  # CKE says Ubuntu 20.04, so who am I to argue?
cpus = "2"
memory = "2048"

cp_ip_addr = "10.0.0.175"
wrk1_ip_addr = "10.0.0.176"
wrk2_ip_addr = "10.0.0.177"
# ------------------
# END CUSTOMISATIONS
# ------------------

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  config.vm.boot_timeout = 900
  config.vm.synced_folder '.', '/vagrant', disabled: true

  config.vm.provider "virtualbox" do |vb|
    vb.memory = memory
    vb.cpus = cpus
    vb.customize [ "modifyvm", :id, "--uartmode1", "disconnected" ]
  end

  # controlplane
  config.vm.define "cp" do |server|
    server.vm.box = box
    server.vm.hostname = "cp.example.com"
    server.vm.post_up_message = "cp is up"
    server.vm.network :private_network, ip: cp_ip_addr

    # kubernetes api server port
    server.vm.network "forwarded_port", guest: 6443, host: 6443

    server.vm.provision "shell", inline: <<-SHELL
      echo "this is my kubernetes controlplane"
    SHELL
  end

  # workers
  config.vm.define "wrk1" do |server|
    server.vm.box = box
    server.vm.hostname = "wrk1.example.com"
    server.vm.post_up_message = "wrk1 is up"
    server.vm.network :private_network, ip: wrk1_ip_addr

    server.vm.provision "shell", inline: <<-SHELL
      echo "this is my kubernetes worker (number 1)"
    SHELL
  end

  config.vm.define "wrk2" do |server|
    server.vm.box = box
    server.vm.hostname = "wrk2.example.com"
    server.vm.post_up_message = "wrk2 is up"
    server.vm.network :private_network, ip: wrk2_ip_addr

    server.vm.provision "shell", inline: <<-SHELL
      echo "this is my kubernetes worker (number 2)"
    SHELL
  end
end

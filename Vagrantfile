# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.ssh.insert_key = false
  config.vm.box = "ubuntu/jammy64"
  config.vm.provider "virtualbox" do |box|
      box.memory = 512
      box.cpus = 1
  end
  
  config.vm.define "server" do |server|
      server.vm.hostname = "server.loc"
      server.vm.network "private_network", ip: "192.168.56.10"
  end

  config.vm.define "client" do |client|
      client.vm.hostname = "client.loc"
      client.vm.network "private_network", ip: "192.168.56.20"
  end

  config.vm.provision "shell", inline: <<-SHELL
    sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
    sudo systemctl restart sshd
  SHELL
  #config.vm.provision "ansible" do |ansible|
    #ansible.playbook = "ansible/playbook.yml"
    #ansible.inventory_path = "ansible/hosts"
    #ansible.host_key_checking = "false"
    #ansible.limit = "all"
  #end
end
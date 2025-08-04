# -*- mode: ruby -*-
# vi: set ft=ruby :
# Vagrantfile for multi-tier monitoring project

Vagrant.configure("2") do |config|
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true

  def configure_client(config, name, ip, memory, script_path)
    config.vm.define name do |client|
      client.vm.box = "eurolinux-vagrant/centos-stream-9"
      client.vm.hostname = name
      client.vm.network "private_network", ip: ip
      client.vm.provider "virtualbox" do |vb|
        vb.memory = memory
      end

      # Run service installation script
      client.vm.provision "shell", path: script_path, privileged: true

      # NRPE setup for monitoring
      client.vm.provision "shell", inline: <<-SHELL
        sudo yum install -y epel-release
        sudo yum update -y
        sudo yum install -y nrpe nagios-plugins-all --skip-broken
        echo "allowed_hosts=127.0.0.1,192.168.56.10" >> /etc/nagios/nrpe.cfg
        sudo systemctl enable nrpe && sudo systemctl start nrpe
        systemctl start firewalld.service
        firewall-cmd --add-port=5666/tcp --permanent
        firewall-cmd --reload
      SHELL
    end
  end

  # Define VMs with relevant scripts
  configure_client(config, "db01", "192.168.56.15", 2048, "scripts/mariadb.sh")
  configure_client(config, "mc01", "192.168.56.14", 900, "scripts/memcached.sh")
  configure_client(config, "rmq01", "192.168.56.13", 600, "scripts/rabbitmq.sh")
  configure_client(config, "app01", "192.168.56.12", 4200, "scripts/tomcat.sh")

  # Nginx frontend web server
  config.vm.define "web01" do |web01|
    web01.vm.box = "ubuntu/jammy64"
    web01.vm.hostname = "web01"
    web01.vm.network "private_network", ip: "192.168.56.11"
    web01.vm.provider "virtualbox" do |vb|
      vb.gui = true
      vb.memory = "800"
    end

    web01.vm.provision "shell", path: "scripts/nginx.sh", privileged: true
    web01.vm.provision "shell", inline: <<-SHELL
      apt update
      apt install -y nagios-nrpe-server nagios-plugins
      echo "allowed_hosts=127.0.0.1,192.168.56.10" >> /etc/nagios/nrpe.cfg
      systemctl enable nagios-nrpe-server && systemctl start nagios-nrpe-server
      ufw allow 5666
    SHELL
  end

  # Monitoring Server (Nagios renamed to monitor01)
  config.vm.define "monitor01" do |monitor01|
    monitor01.vm.box = "eurolinux-vagrant/centos-stream-9"
    monitor01.vm.hostname = "monitor01"
    monitor01.vm.network "private_network", ip: "192.168.56.10"
    monitor01.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
    end

    monitor01.vm.provision "shell", path: "scripts/nagios.sh"
  end
end

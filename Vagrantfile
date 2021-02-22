# -*- mode: ruby -*-
# vi: set ft=ruby :
# Vagrant 2.2.14
# VirtualBox 6.1.16
# Windows 10 2004

BRIDGE_NET="192.168.51."
INTERNAL_NET="192.168.50."
DOMAIN="logs.local"

servers=[
  {
    :hostname => "nginx." + DOMAIN,
    :ip => BRIDGE_NET + "101",
    :ip_int => INTERNAL_NET + "201",
	  :cpus => 1,
    :ram => 512	
  },
  {
    :hostname => "rsyslog." + DOMAIN,
    :ip => BRIDGE_NET + "102",
    :ip_int => INTERNAL_NET + "202",
    :cpus => 1,
    :ram => 512
  },
  {
    :hostname => "elk." + DOMAIN,
    :ip => BRIDGE_NET + "103",
    :ip_int => INTERNAL_NET + "203",
	  :cpus => 1,
    :ram => 4096,
    :hdd_name => "elk.vdi",
    :hdd_size => "10240"
  },
  {
    :hostname => "ansible." + DOMAIN,
    :ip => BRIDGE_NET + "10",
    :ip_int => INTERNAL_NET + "20",
	  :cpus => 1,
    :ram => 512
  },
]
 
Vagrant.configure(2) do |config|
    config.vm.synced_folder ".", "/vagrant", disabled: true
    servers.each do |machine|
        config.vm.define machine[:hostname] do |node|
            node.vm.box = "centos/7"
            node.vm.usable_port_range = (2231..2235)
            node.vm.hostname = machine[:hostname]
            node.vm.network "public_network", ip: machine[:ip], bridge: 'Realtek Gaming GbE Family Controller'
            node.vm.network "private_network", ip: machine[:ip_int], virtualbox__intnet: "logs"
            node.vm.provider "virtualbox" do |vb|
                vb.customize ["modifyvm", :id, "--cpus", machine[:cpus], "--memory", machine[:ram]]
                vb.name = machine[:hostname]
                if (!machine[:hdd_name].nil?)
                    unless File.exist?(machine[:hdd_name])
                        vb.customize ["createhd", "--filename", machine[:hdd_name], "--size", machine[:hdd_size]]
                    end
				          	vb.customize ["storagectl", :id, "--name", "SATA AHCI", "--add", "sata", "--controller", "IntelAHCI"]
                    vb.customize ["storageattach", :id, "--storagectl", "SATA AHCI", "--port", 1, "--device", 0, "--type", "hdd", "--medium", machine[:hdd_name]]
                end
            end
        end
    end
end

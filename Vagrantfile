# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
    config.vm.box = "juniper/ffp-12.1X47-D20.7"
    config.vm.define :firefly1 do | firefly1 |
        firefly1.vm.hostname = 'firefly1'
        firefly1.vm.network "private_network",ip: "192.168.34.16",netmask: "255.255.255.0"
        firefly1.vm.network "private_network",ip: "192.168.35.1",netmask: "255.255.255.252",virtualbox__intnet: "PNI"
    end  
    config.vm.define :firefly2 do | firefly2 |
        firefly2.vm.hostname = 'firefly2'
        firefly2.vm.network "private_network",ip: "192.168.34.17",netmask: "255.255.255.0"
        firefly2.vm.network "private_network",ip: "192.168.35.2",netmask: "255.255.255.252",virtualbox__intnet: "PNI"
    end  
end

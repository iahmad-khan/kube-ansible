require "yaml"
require "fileutils"

Vagrant.require_version ">= 1.7.0"

CONFIG = File.expand_path("./tools/.config.rb")
if File.exist?(CONFIG)
  require CONFIG
end

$os_image = (ENV['OS_IMAGE'] || "ubuntu16").to_sym

def set_vbox(vb, config)
  vb.gui = false
  vb.memory = $system_memory
  vb.cpus = $system_vcpus

  case $os_image
  when :centos7
    config.vm.box = "bento/centos-7.3"
  when :ubuntu16
    config.vm.box = "bento/ubuntu-16.04"
  end
end

def set_libvirt(lv, config)
  lv.nested = true
  lv.volume_cache = 'none'
  lv.uri = 'qemu+unix:///system'
  lv.memory = $system_memory
  lv.cpus = $system_vcpus

  case $os_image
  when :centos7
    config.vm.box = "centos/7"
  when :ubuntu16
    config.vm.box = "yk0/ubuntu-xenial"
  end
end

Vagrant.configure("2") do |config|
  config.vm.provider "virtualbox"
  config.vm.provider "libvirt"

  config.vm.provision "shell", inline: "sudo swapoff -a"
  config.vm.provision "shell", inline: "sudo cp /vagrant/hosts /etc/"

  count = $net_count
  (1..($master_count + $node_count)).each do |mid|
    name = (mid <= $node_count) ? "node" : "master"
    id   = (mid <= $node_count) ? mid : (mid - $node_count)

    config.vm.define "#{name}#{id}" do |n|
      n.vm.hostname = "#{name}#{id}"
      ip_addr = "#{$private_subnet}.#{count}"
      n.vm.network :private_network, ip: "#{ip_addr}",  auto_config: true
      if $bridge_enable && $bridge_eth.to_s != ''
        n.vm.network "public_network", bridge: $bridge_eth
      end

      # Configure virtualbox provider
      n.vm.provider :virtualbox do |vb, override|
        vb.name = "kube-#{n.vm.hostname}"
        set_vbox(vb, override)
      end

      # Configure libvirt provider
      n.vm.provider :libvirt do |lv, override|
        lv.host = "kube-#{n.vm.hostname}"
        set_libvirt(lv, override)
      end
      count += 1

      if mid == ($master_count + $node_count)
        n.vm.provision "cluster", type: "ansible" do |ansible|
          ansible.playbook = "cluster.yml"
          ansible.inventory_path = "inventory"
          ansible.limit = "all"
          ansible.host_key_checking = false
        end
        n.vm.provision "addon", type: "ansible" do |ansible|
          ansible.playbook = "addons.yml"
          ansible.inventory_path = "inventory"
          ansible.limit = "all"
          ansible.host_key_checking = false
        end
      end
    end
  end
end

# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'ipaddr'

unless Vagrant.has_plugin?("vagrant-hostmanager")
  raise 'vagrant-hostmanager is not installed! run "vagrant plugin install vagrant-hostmanager" to fix'
end

unless Vagrant.has_plugin?("vagrant-hosts")
  raise 'vagrant-hosts is not installed. run "vagrant plugin install vagrant-hosts" to fix'
end

VAGRANTFILE_API_VERSION = "2"

class IpAssigner

  def self.next_ip(current_ip)
    IPAddr.new(current_ip).succ.to_s
  end

  def self.generate(start_ip, number_of_addresses)
    number_of_addresses.times.inject([start_ip]) { |acc, i|
      nekst = next_ip(acc.last)
      acc + [nekst]
    }
  end

end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.box = 'hfm4/centos7'
  config.ssh.insert_key = false
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true
  config.vm.provision :hosts, :sync_hosts => true

  accept_oracle_licence = true # set to false if you don't agree (will not install Java8 for you)

  private_network_begin =  "192.168.5.100" # private ip will start incrementing from this
  zk_vm_memory_mb = 256
  zk_port = 2181
  kafka_vm_memory_mb = 512
  kafka_port = 9092

  # < ------- These need to be set in group vars if using Ansible w/o Vagrant -------

  # The follwing variables will need to be passed manually if you want to use the Ansible
  # playbooks w/o Vagrant. They are set in this Vagrantfile in this manner because it allows us to easily
  # increase or decrease the cluster sizes.

  # Note that zk_id must be unique for each host in the cluster. It should ideally not change
  # throughout the lifetime of the Zookeeper installation on a given machine.
  # Note that broker_id must be unique for each host in the cluster. It should ideally not change
  # throughout the lifetime of the Kafka installation on a given machine.
  zk_cluster_info = {
    'zk-node-1' => { :zk_id => 1, :broker_id => 1 },
    'zk-node-2' => { :zk_id => 2, :broker_id => 2 },
    'zk-node-3' => { :zk_id => 3, :broker_id => 3 }
  }

  ## ------- These need to be set in group vars if using Ansible w/o Vagrant ------- >

  # Helper to make new ips
  zk_ips = IpAssigner.generate(
    private_network_begin,
    zk_cluster_info.size)

  zk_cluster = Hash[zk_cluster_info.map.with_index { |(k, v), idx|
    [k, v.merge(
      :ip => zk_ips[idx],
      :memory => kafka_vm_memory_mb,
      :zk_client_port => zk_port,
      :zk_client_forward_to => zk_port + idx,
      :ka_client_port => kafka_port,
      :ka_client_forward_to => kafka_port + idx)]
  }]

  ## ------- Add Zookeeper and Kafka Machines ---------------->

  zk_cluster.each_with_index do |(short_name, info), idx|

    config.vm.define short_name do |host|
      host.vm.network :forwarded_port, guest: info[:zk_client_port], host: info[:zk_client_forward_to]
      host.vm.network :forwarded_port, guest: info[:ka_client_port], host: info[:ka_client_forward_to]
      host.vm.network :private_network, ip: info[:ip]
      host.vm.hostname = short_name
      host.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", info[:memory]]
      end

      # Setup for Kafka Manager on the first node
      if idx == 0
        host.vm.network :forwarded_port, guest: 9000, host: 9654
      end

      # This allows us to provision everything in one go, in parallel.
      if idx == (zk_cluster.size - 1)
          

         host.vm.provision "shell" do |s|
         s.args = [zk_port]
         s.inline = <<-SHELL
              sudo yum update -y
              sudo yum install -y epel-release
              sudo yum update -y
              sudo yum install -y ansible

              echo "Port is $1"

              ansible-galaxy install -r /vagrant/requirements.yml
              ansible-playbook -i /vagrant/hosts -e vagrant_zk_client_port=$1 /vagrant/site.yml
           SHELL
         end
      end
    end
  end
end

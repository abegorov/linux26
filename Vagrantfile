# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_EXPERIMENTAL'] = 'disks'

MACHINES = {
  :'backup' => {
    :domain => 'internal',
    :box => 'rockylinux/9/v4.0.0',
    :cpus => 2,
    :memory => 1024,
    :disks => {
      :backup => '2GB'
    },
    :networks => [
      [ :private_network, {:ip => '192.168.11.160',
          :virtualbox__intnet => 'backup'} ]
    ]
  },
  :'client' => {
    :domain => 'internal',
    :box => 'rockylinux/9/v4.0.0',
    :cpus => 2,
    :memory => 1024,
    :disks => {},
    :networks => [
      [ :private_network, {:ip => '192.168.11.150',
          :virtualbox__intnet => 'backup'} ]
    ]
  }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |host_name, host_config|
    config.vm.define host_name do |host|
      host.vm.box = host_config[:box]
      host.vm.host_name = host_name.to_s + '.' + host_config[:domain].to_s

      host.vm.provider :virtualbox do |vb|
        vb.cpus = host_config[:cpus]
        vb.memory = host_config[:memory]
      end

      host_config[:disks].each do |name, size|
        host.vm.disk :disk, name: name.to_s, size: size
      end

      host_config[:networks].each do |network|
        host.vm.network(network[0], **network[1])
      end

      if MACHINES.keys.last == host_name
        host.vm.provision :ansible do |ansible|
          ansible.playbook = 'playbook.yml'
          ansible.limit = 'all'
          ansible.compatibility_mode = '2.0'
          ansible.raw_arguments = ['--diff']
        end
      end
    end
  end
end

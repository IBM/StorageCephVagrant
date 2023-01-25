# Copyright contributors to the rhcs5-vagrant project.
# Based on vagrant-rhel8 code - Copyright (c) 2019 Andrew Garner

Vagrant.require_version ">= 2.1.0" # 2.1.0 minimum required for triggers

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'

# Number of worker nodes
N = 3

user = ENV['RH_SUBSCRIPTION_MANAGER_USER']
password = ENV['RH_SUBSCRIPTION_MANAGER_PW']
if !user or !password
  puts 'Required environment variables not found. Please set RH_SUBSCRIPTION_MANAGER_USER and RH_SUBSCRIPTION_MANAGER_PW'
  abort
end

register_script = %{
if ! subscription-manager status; then
  sudo subscription-manager register --username=#{user} --password=#{password} --auto-attach
fi
}

unregister_script = %{
if subscription-manager status; then
  sudo subscription-manager unregister
fi
}

timezone_script = %{
localectl set-keymap de
timedatectl set-timezone Europe/Berlin
}

Vagrant.configure("2") do |config|
  config.vm.box = "roboxes/rhel8"

  config.vm.provider :libvirt do |libvirt|
    # Do not use (user) session libvirt - VM networks will not work there on Fedora!
    libvirt.qemu_use_session = false
    libvirt.memory = 8192
    libvirt.cpus = 2
    libvirt.disk_bus = "virtio"
  end
  config.ssh.forward_agent = true
  config.ssh.insert_key = false
  config.hostmanager.enabled = true
  config.vm.synced_folder './', '/vagrant', type: 'rsync'

  # The Ceph client will be our client machine to mount volumes and interact with the cluster
  config.vm.define "ceph-client" do |client|
    client.vm.hostname = "ceph-client"
    client.vm.network "private_network", ip: "172.21.12.11"
    client.vm.provision "shell", inline: register_script
    client.vm.provision "shell", inline: timezone_script
    client.vm.provision "ansible" do |ansible|
      ansible.playbook = "ceph-prep/client-playbook.yml"
      ansible.extra_vars = {
        node_ip: "172.21.12.11",
        user: user,
        password: password
      }
    end
  end
 
  # We need one Ceph admin machine to manage the cluster
  config.vm.define "ceph-admin" do |admin|
    admin.vm.hostname = "ceph-admin"
    admin.vm.network "private_network", ip: "172.21.12.10"
    admin.vm.provision "shell", inline: register_script
    admin.vm.provision "shell", inline: timezone_script
    admin.vm.provision "ansible" do |ansible|
      ansible.playbook = "ceph-prep/admin-playbook.yml"
      ansible.extra_vars = {
        node_ip: "172.21.12.10",
        user: user,
        password: password
      }
    end
  end
 
  # We provision three nodes to be Ceph servers
  (1..N).each do |i|
    config.vm.define "ceph-server-#{i}" do |server|
      server.vm.hostname = "ceph-server-#{i}"
      server.vm.network "private_network", ip: "172.21.12.#{i+11}"
      # Attach disks for Ceph OSDs
      server.vm.provider "libvirt" do |libvirt|
        # 1 small disk
        num = 1
        (1..num).each do |disk|
          libvirt.storage :file, :size => '5G', :cache => 'none'
        end
      end
      server.vm.provision "shell", inline: register_script
      server.vm.provision "shell", inline: timezone_script
      server.vm.provision "ansible" do |ansible|
        ansible.playbook = "ceph-prep/common-playbook.yml"
        ansible.extra_vars = {
          node_ip: "172.21.12.#{i+11}",
          user: user,
          password: password
        }
      end
    end
  end
    
  config.trigger.before :destroy do |trigger|
    trigger.name = "Before Destroy trigger"
    trigger.info = "Unregistering this VM from RedHat Subscription Manager..."
    trigger.warn = "If this fails, unregister VMs manually at https://access.redhat.com/management/subscriptions"
    trigger.run_remote = {inline: unregister_script}
    trigger.on_error = :continue
  end # trigger.before :destroy
end # vagrant configure

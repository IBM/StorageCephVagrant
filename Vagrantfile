# Copyright contributors to the rhcs5-vagrant project.
# Based on vagrant-rhel8 code - Copyright (c) 2019 Andrew Garner

### Some parameters that can be adjusted

# Number of ceph-server nodes. If you change the default number of 3 you
# need to adjust the ceph-prep/config and ceph-prep/hosts files accordingly.
N = 6

# The libvirt storage pool to use
STORAGE_POOL = "default"

# Amount of Memory (RAM) per node. 8192 is recommended, but 3072 works as well.
RAM_SIZE = 4096

# IP prefix and start address for the VMs
# The ceph-a-1 node will get IP_PREFIX.IPSTART (default: 172.21.12.10)
# The ceph-c-1 and ceph-s-x nodes will get subsequent addresses.
IP_PREFIX = "172.22.12."
IP_START = 10

### Do not modify the code below unless you are developing

Vagrant.require_version ">= 2.1.0" # 2.1.0 minimum required for triggers

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'

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
  config.vm.box = "roboxes/rhel9"

  config.vm.provider :libvirt do |libvirt|
    # Do not use (user) session libvirt - VM networks will not work there on Fedora!
    libvirt.qemu_use_session = false
    libvirt.memory = RAM_SIZE
    libvirt.cpus = 2
    libvirt.disk_bus = "virtio"
    libvirt.storage_pool_name = STORAGE_POOL
    # To avoid USB device resource conflicts
    libvirt.graphics_type = "spice"
    (1..2).each do
      libvirt.redirdev :type => "spicevmc"
    end
  end
  config.ssh.forward_agent = true
  config.ssh.insert_key = false
  config.hostmanager.enabled = true
  config.vm.synced_folder './', '/vagrant', type: 'rsync'

  # The Ceph client will be our client machine to mount volumes and interact with the cluster
  config.vm.define "ceph-c-1" do |client|
    client.vm.hostname = "ceph-c-1"
    client.vm.network "private_network", ip: IP_PREFIX+"#{IP_START + 1}"
    client.vm.provision "shell", inline: register_script
    client.vm.provision "shell", inline: timezone_script
    client.vm.provision "ansible" do |ansible|
      ansible.playbook = "ceph-prep/client-playbook.yml"
      ansible.extra_vars = {
        node_ip: IP_PREFIX+"#{IP_START + 1}",
        user: user,
        password: password,
      }
    end
  end
 
  # We need one Ceph admin machine to manage the cluster
  config.vm.define "ceph-a-1" do |admin|
    admin.vm.hostname = "ceph-a-1"
    admin.vm.network "private_network", ip: IP_PREFIX+"#{IP_START}"
    admin.vm.provision "shell", inline: register_script
    admin.vm.provision "shell", inline: timezone_script
    admin.vm.provision "ansible" do |ansible|
      ansible.playbook = "ceph-prep/admin-playbook.yml"
      ansible.extra_vars = {
        node_ip: IP_PREFIX+"#{IP_START}",
        user: user,
        password: password,
      }
    end
  end
 
  # We provision three nodes to be Ceph servers
  (1..N).each do |i|
    config.vm.define "ceph-s-#{i}" do |server|
      server.vm.hostname = "ceph-s-#{i}"
      server.vm.network "private_network", ip: IP_PREFIX+"#{IP_START+i+1}"
      # Attach disks for Ceph OSDs
      server.vm.provider "libvirt" do |libvirt|
        # 2 small disks
        num = 2
        (1..num).each do |disk|
          libvirt.storage :file, :size => '5G', :cache => 'none'
        end
      end
      server.vm.provision "shell", inline: register_script
      server.vm.provision "shell", inline: timezone_script
      server.vm.provision "ansible" do |ansible|
        ansible.playbook = "ceph-prep/common-playbook.yml"
        ansible.extra_vars = {
          node_ip: IP_PREFIX+"#{IP_START+i+1}",
          user: user,
          password: password,
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

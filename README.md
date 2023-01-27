Vagrant / Ansible RHCS5 Deployment
==================================

## Scope


This is an opinionated automated deployment of a Red Hat Ceph Storage (RHCS) 5.x cluster
installation based on RHEL8 up to the point where you run the preflight Ansible playbook.

![RHCS5 Screenshot](./RHCS5-Vagrant.png "RHCS5 Screenshot")


## Acknowledgments


The code to facilitate automated subscription-manager registration was derived from the
[https://github.com/agarthetiger/vagrant-rhel8](https://github.com/agarthetiger/vagrant-rhel8)
project.

Copyright notice:
```
# Copyright contributors to the rhcs5-vagrant project.
# Based on vagrant-rhel8 code - Copyright (c) 2019 Andrew Garner
```


## Prerequisites

You need a Linux host that is ideally equipped with 64GB+ RAM and 8+ vCPUs.
The configuration can be adjusted to make the deployment work on less capable
hardware.

Fedora Linux (IBM OpenClient for Fedora) was used to develop this automation.
Of course the instructions below can be adopted to other Linux variants with
their respective package manager commands.

- Vagrant: `sudo dnf -y install vagrant`
- QEM/KVM/libvirt: `sudo dnf -y install qemu libvirt libvirt-devel ruby-devel gcc`
- [Vagrant libvirt](https://github.com/vagrant-libvirt/vagrant-libvirt) Provider:
For Fedora use `sudo dnf -y install vagrant-libvirt`. Check for a package available
for your Linux distribution before trying to install with `vagrant plugin install vagrant-libvirt`. 
- [Vagrant Hostmanager](https://github.com/devopsgroup-io/vagrant-hostmanager)
plugin: For Fedora use `sudo dnf -y install vagrant-hostmanager`.
Check for a package available for your Linux distribution before trying to install
with `vagrant plugin install vagrant-hostmanager`. 
- Ansible: `sudo dnf -y install ansible`.

The host needs internet connectivity to download the required packages and
container images.


Tested with

- Fedora 36: Vagrant 2.2.19, vagrant-libvirt 0.7.0 and Ansible 5.9.0
- Fedora 37: Vagrant 2.2.19, vagrant-libvirt 0.7.0 and Ansible 7.1.0

You need a subscription for RHEL and Red Hat Ceph Storage.

Finally you need to create a private/public key pair in the `ceph-prep` directory:

```
cd ceph-prep
# use empty password when prompted
ssh-keygen -t rsa -f ./id_rsa
cd ..
```

There are some options that you can set in the [Vagrantfile](./Vagrantfile), look
at the section marked with `Some parameters that can be adjusted`.

## Installation

Bring up the Cluster by setting the environment variables with your Red Hat
credentials then starting start the vagrant/ansible driven deployment:
```
export RH_SUBSCRIPTION_MANAGER_USER=<your Red Hat username>
export RH_SUBSCRIPTION_MANAGER_PW=<your Red Hat password>
# For debugging: vagrant up --no-parallel --no-destroy-on-error
vagrant up --no-parallel
```

The vagrant deployment will automatically subscribe the machines at Red Hat.

To finalize the installation, ssh into the `ceph-admin` node and execute the
preflight Ansible playbook:

```
vagrant ssh ceph-admin
cd /usr/share/cephadm-ansible/
ansible-playbook -i inventory/production/hosts cephadm-preflight.yml --extra-vars "ceph_origin=rhcs"
```

For the next step you need to set the environment variables again on the
ceph-admin node:

```
export RH_SUBSCRIPTION_MANAGER_USER=<your Red Hat username>
export RH_SUBSCRIPTION_MANAGER_PW=<your Red Hat password>
```

Then you can continue with bootstrapping the cluster:

```
sudo cephadm bootstrap --cluster-network 172.21.12.0/24 --mon-ip 172.21.12.10 --registry-url registry.redhat.io --registry-username $RH_SUBSCRIPTION_MANAGER_USER  --registry-password $RH_SUBSCRIPTION_MANAGER_PW --yes-i-know
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-server-1
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-server-2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-server-3
```

Note down the admin password that the `cephadm bootstrap` command printed. You
will need it for logging into the console the first time.


## Configuration

You can now complete the installation by logging into the
[RHCS GUI](https://172.21.12.10:8443) to

- change the password
- activate telemetry module
- add nodes (ceph-server-1, ceph-server-2, ceph-server-3)
- add OSDs (wait until all ceph-server nodes are active before starting that,
might take up to 10 minutes)


Most of the following is taken from the course [Hands-on with Red Hat Ceph Storage 5](https://training-lms.redhat.com/sso/saml/auth/rhopen?RelayState=deeplinkoffering%3D44428318).

Note: When creating an RBD image, ensure that you do not have the `Exclusive Lock`
option set, otherwise there might be access issues mapping the RBD volume on the
client.

The following sections provide additional commands (on the `ceph-admin` node)
to configure RBD access, CephFS and RGW S3 and Swift access.

### Configure Block (RBD) access

```
vagrant ssh ceph-admin
# Create RBD pool
sudo ceph osd pool create rbd 64
sudo ceph osd pool application enable rbd rbd
# Create RBD user
sudo ceph auth get-or-create client.harald -o /etc/ceph/ceph.client.harald.keyring
sudo ceph auth caps client.harald mon 'allow r' osd 'allow rwx'
sudo ceph auth list
# Create RBD image
sudo rbd create rbd/test --size=128M
sudo rbd ls
```

### Configure File (CephFS) access

```
# Create CephFS
sudo ceph fs volume create fs_name --placement=ceph-server-1,ceph-server-2
sudo ceph fs volume ls
cd
cat <<EOF >mds.yaml 
service_type: mds
service_id: fs_name
placement:
  count: 2
EOF
sudo ceph orch apply -i mds.yaml 
sudo ceph orch ls
sudo ceph orch ps
sudo ceph -s
sudo ceph df
sudo ceph fs status fs_name
# Use admin keyring for CephFS mount
sudo cat /etc/ceph/ceph.client.admin.keyring 
```

### Configure Object (RGW) access

```
# Create RGW

# Note that this is due to the low number of OSDs
# While we're setting it on global level to silence the warnings setting it
# on OSD and Mon level would have been sufficient. Setting it on Mon level as
# documented does not silence the warning.
sudo ceph config set global mon_max_pg_per_osd 512
sudo radosgw-admin realm create --rgw-realm=test_realm --default
sudo radosgw-admin zonegroup create --rgw-zonegroup=default --master --default
sudo radosgw-admin zone create --rgw-zonegroup=default --rgw-zone=test_zone --master --default
sudo radosgw-admin period update --rgw-realm=test_realm --commit
sudo ceph orch apply rgw test --realm=test_realm --zone=test_zone --placement="2 ceph-server-2 ceph-server-3"
sudo ceph orch ls
sudo ceph orch ps
sudo ceph -s
sudo radosgw-admin user create --uid='user1' --display-name='First User' --access-key='S3user1' --secret-key='S3user1key'
sudo radosgw-admin subuser create --uid='user1' --subuser='user1:swift' --secret-key='Swiftuser1key' --access=full
sudo radosgw-admin user info --uid='user1'
```


## Usage

Now use the created resources on the `ceph-client` node.

### RBD volume access

```
vagrant ssh ceph-client
# Add the configuration according to the admin node equivalent
sudo vi /etc/ceph/ceph.conf
# Add the client keyring according to the admin node equivalent
sudo vi /etc/ceph/ceph.client.harald.keyring
# Create a block device
sudo rbd --id harald ls
sudo rbd --id harald map rbd/test
sudo rbd --id harald showmapped
sudo mkfs.ext4 /dev/rbd0
sudo mkdir /mnt/rbd
sudo mount -o user /dev/rbd0 /mnt/rbd/
sudo chown vagrant:vagrant /mnt/rbd/
echo "hello world" > /mnt/rbd/hello.txt
```

### CephFS file access

```
# Add the admin keyring as created on the admin node /etc/ceph/client.admin.keyring
sudo vi /etc/ceph/ceph.client.admin.keyring
sudo mkdir /mnt/cephfs
sudo mount -t ceph ceph-server-1:6789:/ /mnt/cephfs -o name=admin,fs=fs_name
sudo chown vagrant:vagrant /mnt/cephfs
df -h
echo "hello world" > /mnt/cephfs/hello.txt
```

### RGW OpenStack Swift object access

```
sudo pip3 install python-swiftclient
sudo rados --id harald lspools
swift -A http://ceph-server-2:80/auth/1.0 -U user1:swift -K 'Swiftuser1key' list
swift -A http://ceph-server-2:80/auth/1.0 -U user1:swift -K 'Swiftuser1key' post container-1
swift -A http://ceph-server-2:80/auth/1.0 -U user1:swift -K 'Swiftuser1key' list
base64 /dev/urandom | head -c 10000000 >dummy_file1.txt
swift -A http://ceph-server-2:80/auth/1.0 -U user1:swift -K 'Swiftuser1key' upload container-1 dummy_file1.txt 
```

### RGW S3 object access

Here we are actually testing S3 client access to the same data that was stored
using the Swift protocol.
When asked for the access key, put in `S3user1`, for the secret key use
`S3user1key`.

```
sudo pip3 install awscli
echo "Put in S3user1 as access key and S3user1key as secret key"
aws configure --profile ceph
aws --profile ceph --endpoint http://ceph-server-2 s3 ls
aws --profile ceph --endpoint http://172.21.12.13 s3 ls s3://container-1
```

## Shutting down and restarting the VMs

Shutting down the VMs with `vagrant halt` then re-starting them at a later time
with `vagrant reload` works with this solution. RBD mappings on the ceph-client
VM need to be re-created after the reboot.

## Deleting the deployment

Ensure that you have the machines up and running when destroying the machines
using `vagrant destroy -f` because only then the machines will be automatically
unregistered at Red Hat.

## Contacting the Project Maintainers, Contributing

**NOTE: This repository has been configured with the [DCO bot](https://github.com/probot/dco).**

If you have any questions or issues you can create a new [issue here](https://github.com/IBM/rhcs5-vagrant/issues).

Pull requests are very welcome! Make sure your patches are well tested.
Ideally create a topic branch for every separate change you make. For
example:

1. Fork the repo
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Added some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

## License

All source files must include a Copyright and License header. The SPDX license header is 
preferred because it can be easily scanned.

If you would like to see the detailed LICENSE click [here](LICENSE).

```text
#
# Copyright 2022- IBM Inc. All rights reserved
# SPDX-License-Identifier: MIT
#
```

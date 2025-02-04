# Openstack Ansible

Following the instructions on https://docs.openstack.org/openstack-ansible/2024.2/user/aio/quickstart.html.

The compatibility matrix can be visulized on https://docs.openstack.org/openstack-ansible/2024.2/admin/upgrades/compatibility-matrix.html

Since I want to use less resources as possible with the fastest installation method I will install using the **distro** packages without containers (**metal**).

## Preparing the Virtual Machine

Since I work on Red Hat, I will use `CentOS`:

```bash
cd /var/lib/libvirt/images
wget https://cloud.centos.org/centos/9-stream/x86_64/images/CentOS-Stream-GenericCloud-x86_64-9-latest.x86_64.qcow2 -O centos9.qcow2
virt-customize -a centos9.qcow2 --root-password password:centos --uninstall cloud-init --install vim,git
qemu-img create -f qcow2 -F qcow2 -b centos9.qcow2 openstack.qcow2 500G
```

## Preparing the environment

Increase disk size, configure static network and allow ssh access:

```bash
growpart /dev/vda 4
xfs_growfs /dev/vda4

nmcli con
nmcli con modify 'System eth0' \
ipv4.method static \
ipv4.address 172.27.11.10/24 \
ipv4.gateway 172.27.11.1 \
ipv4.dns 172.27.11.1
nmcli con reload
nmcli dev reapply eth0

sed -i 's,#PermitRootLogin.*,PermitRootLogin yes,' /etc/ssh/sshd_config 
sed -i 's,#PasswordAuthentication.*,PasswordAuthentication yes,' /etc/ssh/sshd_config
systemctl restart sshd

grubby --update-kernel ALL --args selinux=0

reboot
```

Install Openstack stuff:

```bash
dnf install -y git vim unzip tar python3-chardet

git clone https://opendev.org/openstack/openstack-ansible /opt/openstack-ansible
cd /opt/openstack-ansible
git checkout stable/2024.2
git describe --abbrev=0 --tags
git checkout 30.0.1

scripts/bootstrap-ansible.sh

# Modify the resulting hostname to be "openstack" instead of "aio1"
grep -r aio1 | cut -d: -f1 | sort | uniq | xargs -I{} sed -i 's,aio1,openstack,g' {}

export INSTALL_METHOD=distro
export SCENARIO='aio_metal'

# Force swift to use a single replica and a single loopback disk
mkdir -p /etc/openstack_deploy/conf.d
cp etc/openstack_deploy/conf.d/swift.yml.aio /etc/openstack_deploy/conf.d/swift.yml
sed -Ei 's,(mount_point: /srv),\1\n    repl_number: 1,' /etc/openstack_deploy/conf.d/swift.yml
sed -Ei 's,- name: swift2.img|- name: swift3.img,,' /etc/openstack_deploy/conf.d/swift.yml

# Don't use loopback files for disks on these services
sed -i 's,bootstrap_host_loopback_nova: yes,bootstrap_host_loopback_nova: no,' tests/roles/bootstrap-host/defaults/main.yml
sed -i 's,bootstrap_host_loopback_manila: yes,bootstrap_host_loopback_manila: no,' tests/roles/bootstrap-host/defaults/main.yml
sed -i 's,bootstrap_host_loopback_machines: yes,bootstrap_host_loopback_machines: no,' tests/roles/bootstrap-host/defaults/main.yml

# Force Glance to use simple file storage
sed -Ei 's,(glance_default_store: .*),#\1\nglance_default_store: file,' etc/openstack_deploy/user_variables.yml
sed -Ei 's,(glance_default_store: .*),#\1\nglance_default_store: file,' inventory/group_vars/cinder_all/defaults.yml
sed -Ei 's,(glance_default_store: .*),#\1\nglance_default_store: file,' inventory/group_vars/glance_all/defaults.yml

scripts/bootstrap-aio.sh
openstack-ansible openstack.osa.setup_hosts
openstack-ansible openstack.osa.setup_infrastructure
openstack-ansible openstack.osa.setup_openstack
```

## Persisting Network

Since Networkmanger is masked after the baremetal distro installation, we should create a file for `systemd-networkd`, in this case the network is `172.27.11.0/24`:

```bash
cat > /etc/systemd/network/0-eth0.network <<EOF
[Match]
Name = eth0

[Network]
Address = 172.27.11.10/24
Gateway = 172.27.11.1
DNS = 172.27.11.1
EOF
```

## Interacting with Openstack

```bash
source /root/openrc

openstack server create --network public --flavor tempest1 --image 'cirros 0.6' cirros
openstack server delete cirros

openstack flavor create --id a1 --vcpus 1 --ram   256 --disk 120 1cpu-256mb
openstack flavor create --id a2 --vcpus 1 --ram   512 --disk 120 1cpu-512mb
openstack flavor create --id a3 --vcpus 1 --ram  1024 --disk 120 1cpu-1g
openstack flavor create --id b1 --vcpus 2 --ram  2048 --disk 120 2cpu-2gb
openstack flavor create --id b2 --vcpus 2 --ram  4096 --disk 120 2cpu-4gb
openstack flavor create --id b3 --vcpus 2 --ram  8192 --disk 120 2cpu-8gb
openstack flavor create --id c1 --vcpus 3 --ram  4096 --disk 120 3cpu-4gb
openstack flavor create --id c2 --vcpus 3 --ram  8192 --disk 120 3cpu-8gb
openstack flavor create --id d1 --vcpus 4 --ram  8192 --disk 120 4cpu-8gb
openstack flavor create --id d2 --vcpus 4 --ram 16384 --disk 120 4cpu-16gb
openstack flavor create --id e1 --vcpus 8 --ram 16384 --disk 120 8cpu-16gb
openstack flavor create --id e2 --vcpus 8 --ram 32768 --disk 120 8cpu-32gb

openstack floating ip create --floating-ip 172.29.248.10 --description 'Openshift API' public
openstack floating ip create --floating-ip 172.29.248.11 --description 'Openshift Ingress' public

openstack quota show --default
openstack quota set --cores -1 --instances -1 --ram -1 --volumes -1 --gigabytes -1 --floating-ips -1 --key-pairs -1 --routers -1 --networks -1 --subnets -1 --ports -1 --default
```

### From outside

```bash
sudo ip route add 172.29.248.0/22 via 172.27.11.10
sudo ip route add 172.29.236.0/22 via 172.27.11.10
```

# Openshift

```bash
mkdir openstack
cd openstack
scp root@172.27.11.10:/etc/openstack_deploy/pki/roots/ExampleCorpIntermediate/certs/ExampleCorpIntermediate-chain.crt ca-chain.crt
```

```yaml
# install-config.yaml
apiVersion: v1
baseDomain: openshift.openstack.local
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform: {}
  replicas: 2
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    openstack:
      type: 8cpu-32gb
  replicas: 3
metadata:
  creationTimestamp: null
  name: main
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  openstack:
    apiFloatingIP: 172.29.248.10
    apiVIPs:
    - 10.0.0.10
    cloud: epyc
    defaultMachinePlatform:
      type: 4cpu-16gb
    externalDNS: null
    externalNetwork: public
    ingressFloatingIP: 172.29.248.11
    ingressVIPs:
    - 10.0.0.11
publish: External
pullSecret: '{"auths":{"cloud.openshift.com":{"auth":"","email":"email@example.com"},"quay.io":{"auth":"","email":"email@example.com"},"registry.connect.redhat.com":{"auth":"","email":"email@example.com"},"registry.redhat.io":{"auth":"","email":"email@example.com"}}}'
sshKey: |
  ssh-ed25519 ... root@openstack
additionalTrustBundlePolicy: Always
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  ...
  -----END CERTIFICATE-----
```

```yaml
# clouds.yaml
clouds:
  epyc:
    auth:
      auth_url: http://172.29.236.101:5000/v3
      project_name: admin
      username: admin
      password: <password>
      user_domain_name: Default
      project_domain_name: Default
    cacert: /home/hector/openstack/ca-chain.crt
```

```bash
export OS_CLIENT_CONFIG_FILE=/home/hector/openstack/clouds.yaml
```

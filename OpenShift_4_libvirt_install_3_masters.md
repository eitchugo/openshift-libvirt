# OpenShift 4.3 installation with libvirt/kvm (development) - 1 master mode

This installation is used for development/testing/learning only. It uses the Bare Metal UPI mode (User Provided Infrastructure) and libvirt.

## Information Summary

* Cluster Name: `ocp4-ikw`
* Base domain: `localdomain`
* Servers Network: `192.168.139.0/24`
* Node quantity: 3 master and 2 workers
* Load Balancer: HAproxy running on host (`192.168.139.1`)
* Total Space Needed: 75GB
* Total Memory Needed: 28GB (temporary +4GB for bootstrap)

## Network and DNS

1. Install and enable libvirt

```
sudo yum install libvirt
sudo systemctl enable libvirtd --now
```

2. Create the network:

```
cat <<EOF > /tmp/net-ocp4-ikw.xml
<network>
  <name>ocp4-ikw</name>
  <uuid>ab33f496-5392-49be-99e2-b07c182bc027</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr139' stp='on' delay='0'/>
  <mac address='02:b3:80:f6:b2:27'/>
  <domain name='ocp4-ikw.localdomain' localOnly='yes'/>
  <dns>
    <forwarder domain='apps.ocp4-ikw.localdomain' addr='127.0.0.1'/>
    <host ip='192.168.139.1'>
      <hostname>api</hostname>
      <hostname>api-int</hostname>
    </host>
    <host ip='192.168.139.10'>
          <hostname>etcd-0</hostname>
    </host>
    <host ip='192.168.139.11'>
          <hostname>etcd-1</hostname>
    </host>
    <host ip='192.168.139.12'>
          <hostname>etcd-2</hostname>
    </host>
    <srv service='etcd-server-ssl' protocol='tcp' domain='ocp4-ikw.localdomain' target='etcd-0.ocp4-ikw.localdomain' port='2380' priority='0' weight='10'/>
    <srv service='etcd-server-ssl' protocol='tcp' domain='ocp4-ikw.localdomain' target='etcd-1.ocp4-ikw.localdomain' port='2380' priority='0' weight='10'/>
    <srv service='etcd-server-ssl' protocol='tcp' domain='ocp4-ikw.localdomain' target='etcd-2.ocp4-ikw.localdomain' port='2380' priority='0' weight='10'/>
  </dns>
  <ip address='192.168.139.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.139.5' end='192.168.139.254'/>
      <host mac='02:68:45:3c:3b:1d' name='bootstrap.ocp4-ikw.localdomain' ip='192.168.139.5'/>
      <host mac='02:fd:9d:56:3d:45' name='master-0.ocp4-ikw.localdomain' ip='192.168.139.10'/>
      <host mac='02:8f:c1:df:8a:0d' name='master-1.ocp4-ikw.localdomain' ip='192.168.139.11'/>
      <host mac='02:9a:f8:bb:9d:b3' name='master-2.ocp4-ikw.localdomain' ip='192.168.139.12'/>
      <host mac='02:1d:70:c7:ca:cf' name='worker-0.ocp4-ikw.localdomain' ip='192.168.139.20'/>
      <host mac='02:1d:70:c7:ca:cf' name='worker-1.ocp4-ikw.localdomain' ip='192.168.139.21'/>
    </dhcp>
  </ip>
</network>
EOF

sudo virsh net-destroy ocp4-ikw
sudo virsh net-undefine ocp4-ikw
sudo virsh net-define --file /tmp/net-ocp4-ikw.xml
sudo virsh net-start ocp4-ikw
sudo virsh net-autostart ocp4-ikw
```

3. Test DNS from libvirt network:

```
host api.ocp4-ikw.localdomain 192.168.139.1
host api-int.ocp4-ikw.localdomain 192.168.139.1
host etcd-0.ocp4-ikw.localdomain 192.168.139.1
host etcd-1.ocp4-ikw.localdomain 192.168.139.1
host etcd-2.ocp4-ikw.localdomain 192.168.139.1

host -t srv _etcd-server-ssl._tcp.ocp4-ikw.localdomain 192.168.139.1
```

Results:

```
api.ocp4-ikw.localdomain has address 192.168.139.1
api-int.ocp4-ikw.localdomain has address 192.168.139.1
etcd-0.ocp4-ikw.localdomain has address 192.168.139.10
etcd-1.ocp4-ikw.localdomain has address 192.168.139.11
etcd-2.ocp4-ikw.localdomain has address 192.168.139.12

_etcd-server-ssl._tcp.ocp4-ikw.localdomain has SRV record 0 10 2380 etcd-2.ocp4-ikw.localdomain.
_etcd-server-ssl._tcp.ocp4-ikw.localdomain has SRV record 0 10 2380 etcd-1.ocp4-ikw.localdomain.
_etcd-server-ssl._tcp.ocp4-ikw.localdomain has SRV record 0 10 2380 etcd-0.ocp4-ikw.localdomain.

```

4. Configure the domain inside the host. If you're using NetworkManager, first modify `/etc/NetworkManager/NetworkManager.conf`:

```
[main]
dns = dnsmasq
```

Add the entry telling dnsmasq to search in our libvirt's DNS server for the new domain, and the load balancer (host ip):

```
cat <<EOF | sudo tee /etc/NetworkManager/dnsmasq.d/ocp4-ikw.conf
server=/ocp4-ikw.localdomain/192.168.139.1
address=/.apps.ocp4-ikw.localdomain/192.168.139.1
EOF
```

Restart NetworkManager:

```
sudo systemctl restart NetworkManager
```

Test the DNS from the host:

```
host api.ocp4-ikw.localdomain 127.0.0.1
host etcd-2.ocp4-ikw.localdomain 127.0.0.1
host testing.apps.ocp4-ikw.localdomain 127.0.0.1
```

Output:

```
api.ocp4-ikw.localdomain has address 192.168.139.1
etcd-2.ocp4-ikw.localdomain has address 192.168.139.12
testing.apps.ocp4-ikw.localdomain has address 192.168.139.1
```

## Load-Balancer

We'll use an HAProxy in the host as a load balancer. Installation and usage is outside the scope of this tutorial.

Add the following to `/etc/haproxy/haproxy.cfg`:

```
listen ocp4-ikw-api-server-6443
    bind 192.168.139.1:6443
    mode tcp
    balance source
    server master0 192.168.139.10:6443 check inter 1s
    server master1 192.168.139.11:6443 check inter 1s
    server master2 192.168.139.12:6443 check inter 1s
    server bootstrap 192.168.139.5:6443 check inter 1s

listen ocp4-ikw-machine-config-server-22623
    bind 192.168.139.1:22623
    mode tcp
    balance source
    server master0 192.168.139.10:22623 check inter 1s
    server master1 192.168.139.11:22623 check inter 1s
    server master2 192.168.139.12:22623 check inter 1s
    server bootstrap 192.168.139.5:22623 check inter 1s
 
listen ocp4-ikw-ingress-router-80
    bind 192.168.139.1:80
    mode tcp
    balance source
    server master0 192.168.139.10:80 check inter 1s
    server master1 192.168.139.11:80 check inter 1s
    server master2 192.168.139.12:80 check inter 1s
    server worker0 192.168.139.20:80 check inter 1s
    server worker1 192.168.139.21:80 check inter 1s

listen ocp4-ikw-ingress-router-443
    bind 192.168.139.1:443
    mode tcp
    balance source
    server master0 192.168.139.10:443 check inter 1s
    server master1 192.168.139.11:443 check inter 1s
    server master2 192.168.139.12:443 check inter 1s
    server worker0 192.168.139.20:443 check inter 1s
    server worker1 192.168.139.21:443 check inter 1s
```

If SELinux is enabled on the host, allow HAProxy to bind to any port:

```
setsebool -P haproxy_connect_any 1
```

Restart HAPRoxy and check if it's running OK:

```
sudo systemctl restart haproxy
sudo systemctl status haproxy
```

If using a firewall on host, don't foget to allow connections to these ports on IP `192.168.139.1`: `6443`, `22623`, `80` and `443`.

## HTTP Server

An HTTP server is needed to serve files to new machines. For example, in the host configure an Apache Server:

```
sudo mkdir -p /var/www/html/ocp4

cat <<EOF | sudo tee /etc/httpd/conf.d/ocp4-ikw.conf
Listen 192.168.139.1:8080
<VirtualHost 192.168.139.1:8080>
  DocumentRoot "/var/www/html/ocp4"
</VirtualHost>
EOF

sudo systemctl restart httpd
```

When using this example, remember to allow firewall to port 8080 on IP `192.168.139.1`.

## SSH Keys

If you don't have an SSH key and/or didn't add to the ssh-agent, follow documentation:

https://docs.openshift.com/container-platform/4.3/installing/installing_bare_metal/installing-bare-metal.html#ssh-agent-using_installing-bare-metal

It's easy:

```
$ ssh-keygen
$ ssh-add
```

Public key will be stored on `$HOME/.ssh/id_rsa.pub`.

## Installation file

If you don't have the installer, grab from [this url](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/), decompress and it's ready to use. At the time of this writing, the file is `openshift-install-linux-4.3.0.tar.gz`

1. Create a file named `upi-install-config.yaml`:

```
cat <<EOF > upi-install-config.yaml
apiVersion: v1
baseDomain: localdomain
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4-ikw
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14 
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {} 
fips: false 
pullSecret: '{"auths": ...}' 
sshKey: 'ssh-ed25519 AAAA...' 
EOF
```

The `pullSecret` can be obtained from [Red Hat Cloud - Create New Cluster](https://cloud.redhat.com).

The `sshKey` is the public key you'll use to SSH into the hosts (if needed).

2. Create a directory for this installation and copy the `install-config.yaml` there:

```
mkdir -p install-ocp4-ikw
cp upi-install-config.yaml install-ocp4-ikw/install-config.yaml
```

3. Create the manifests:

```
./openshift-install create manifests --dir=install-ocp4-ikw
sed -i -r 's/(mastersSchedulable: ).*/\1False/' install-ocp4-ikw/manifests/cluster-scheduler-02-config.yml
```

4. Create the ignition files to boot up the new machines:

```
./openshift-install create ignition-configs --dir=install-ocp4-ikw
```

5. Copy the ignition files to the HTTP Server (check previous section):

```
sudo cp install-ocp4-ikw/*.ign /var/www/html/ocp4
sudo chmod 644 /var/www/html/ocp4/*.ign
```

6. Download the ISO and OS images to proper locations. At the time of this writing, these files are for 4.3.0. You may need to change them.

```
sudo curl https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.3/latest/rhcos-4.3.0-x86_64-installer.iso -o /var/lib/libvirt/images/rhcos-4.3.0-x86_64-installer.iso
sudo curl https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.3/latest/rhcos-4.3.0-x86_64-metal.raw.gz -o /var/www/html/ocp4/rhcos-4.3.0-x86_64-metal.raw.gz
```

## Virtual Machines 

The installation will begin!

1. Create servers. Pay attention to MAC addresses.

```
export VM_DIR=/var/lib/libvirt/images
```

### Bootstrap

```
sudo virt-install \
  --cdrom /var/lib/libvirt/images/rhcos-4.3.0-x86_64-installer.iso \
  --memory 4096 \
  --vcpus 2 \
  --cpu host \
  --disk path=${VM_DIR}/bootstrap.ocp4-ikw.localdomain.qcow2,size=10,bus=virtio,format=qcow2 \
  --os-type=linux \
  --os-variant=rhel8-unknown \
  --graphics spice \
  --network network=ocp4-ikw,mac=02:68:45:3c:3b:1d \
  --name bootstrap.ocp4-ikw.localdomain &
```

Inside the VM Console, press `TAB` and put the following options in the command line:

```
coreos.inst=yes
coreos.inst.install_dev=vda
coreos.inst.image_url=http://192.168.139.1:8080/rhcos-4.3.0-x86_64-metal.raw.gz
coreos.inst.ignition_url=http://192.168.139.1:8080/bootstrap.ign
ip=dhcp
```

### Masters

```
sudo virt-install \
  --cdrom /var/lib/libvirt/images/rhcos-4.3.0-x86_64-installer.iso \
  --memory 8192 \
  --vcpus 4 \
  --cpu host \
  --disk path=${VM_DIR}/master-0.ocp4-ikw.localdomain.qcow2,size=15,bus=virtio,format=qcow2 \
  --os-type=linux \
  --os-variant=rhel8-unknown \
  --graphics spice \
  --network network=ocp4-ikw,mac=02:fd:9d:56:3d:45 \
  --name master-0.ocp4-ikw.localdomain &

sudo virt-install \
  --cdrom /var/lib/libvirt/images/rhcos-4.3.0-x86_64-installer.iso \
  --memory 8192 \
  --vcpus 4 \
  --cpu host \
  --disk path=${VM_DIR}/master-1.ocp4-ikw.localdomain.qcow2,size=15,bus=virtio,format=qcow2 \
  --os-type=linux \
  --os-variant=rhel8-unknown \
  --graphics spice \
  --network network=ocp4-ikw,mac=02:8f:c1:df:8a:0d \
  --name master-1.ocp4-ikw.localdomain &

sudo virt-install \
  --cdrom /var/lib/libvirt/images/rhcos-4.3.0-x86_64-installer.iso \
  --memory 8192 \
  --vcpus 4 \
  --cpu host \
  --disk path=${VM_DIR}/master-2.ocp4-ikw.localdomain.qcow2,size=15,bus=virtio,format=qcow2 \
  --os-type=linux \
  --os-variant=rhel8-unknown \
  --graphics spice \
  --network network=ocp4-ikw,mac=02:9a:f8:bb:9d:b3 \
  --name master-2.ocp4-ikw.localdomain &
```

Inside the VM Console, press `TAB` and put the following options in the command line:

```
coreos.inst=yes
coreos.inst.install_dev=vda
coreos.inst.image_url=http://192.168.139.1:8080/rhcos-4.3.0-x86_64-metal.raw.gz
coreos.inst.ignition_url=http://192.168.139.1:8080/master.ign
ip=dhcp
```

### Workers

```
sudo virt-install \
  --cdrom /var/lib/libvirt/images/rhcos-4.3.0-x86_64-installer.iso \
  --memory 2048 \
  --vcpus 2 \
  --cpu host \
  --disk path=${VM_DIR}/worker-0.ocp4-ikw.localdomain.qcow2,size=10,bus=virtio,format=qcow2 \
  --os-type=linux \
  --os-variant=rhel8-unknown \
  --graphics spice \
  --network network=ocp4-ikw,mac=02:be:2d:7d:bf:af \
  --name worker-0.ocp4-ikw.localdomain &

sudo virt-install \
  --cdrom /var/lib/libvirt/images/rhcos-4.3.0-x86_64-installer.iso \
  --memory 2048 \
  --vcpus 2 \
  --cpu host \
  --disk path=${VM_DIR}/worker-1.ocp4-ikw.localdomain.qcow2,size=10,bus=virtio,format=qcow2 \
  --os-type=linux \
  --os-variant=rhel8-unknown \
  --graphics spice \
  --network network=ocp4-ikw,mac=02:1d:70:c7:ca:cf \
  --name worker-1.ocp4-ikw.localdomain &
```

Inside the VM Console, press `TAB` and put the following options in the command line:

```
coreos.inst=yes
coreos.inst.install_dev=vda
coreos.inst.image_url=http://192.168.139.1:8080/rhcos-4.3.0-x86_64-metal.raw.gz
coreos.inst.ignition_url=http://192.168.139.1:8080/worker.ign
ip=dhcp
```

## Cluster install

Run this and wait for it to complete.

```
./openshift-install --dir=install-ocp4-ikw wait-for bootstrap-complete
```

**Warning**: this takes a lot of time to complete since it has to download a lot of images from quay.io. Check your Internet bandwidth :) You may have to issue the command more than 1 time if it takes more than 30 minutes.

If you want to see what is happening, check:

```
ssh core@bootstrap.ocp4-ikw.localdomain "sudo journalctl -b -f -u bootkube.service"
```

After the `wait-for bootstrap` completes, you can delete the bootstrap machine.

## Workers

Approve new workers:

```
export KUBECONFIG=install-ocp4-ikw/auth/kubeconfig
oc get csr -o name | xargs oc adm certificate approve
```

Do this until you have all nodes Ready.

## Memory limits

We don't want so much memory used by prometheus:

```
cat <<EOF > /tmp/config.yaml
prometheusK8s:
  resources: 
    requests:
      memory: 256M
EOF

oc create configmap cluster-monitoring-config --from-file=config.yaml=/tmp/config.yaml -n openshift-monitoring
sleep 30
oc delete pod prometheus-k8s-0 oc delete pod prometheus-k8s-1 -n openshift-monitoring
```

## Finishing

Run this and wait for it to complete.

```
./openshift-install --dir=install-ocp4-ikw wait-for install-complete 
```

**Warning**: this takes a lot of time to complete since it has to download a lot of images from quay.io. Check your Internet bandwidth :) You may have to issue the command more than 1 time if it takes more than 30 minutes.

Congratulations.

# Notes

This works for automatically sending kernel args but enters an infinite loop since booting from the kernel files is permanent.

```
sudo virt-install \
  --memory 4096 \
  --vcpus 2 \
  --cpu host \
  --disk path=/var/lib/libvirt/images/bootstrap.ocp4-ikw.localdomain.qcow2,size=10,bus=virtio,format=qcow2 \
  --boot hd,kernel=/var/lib/libvirt/images/rhcos-4.3.0-x86_64-installer-kernel,initrd=/var/lib/libvirt/images/rhcos-4.3.0-x86_64-installer-initramfs.img,kernel_args="coreos.inst=yes coreos.inst.install_dev=vda coreos.inst.image_url=http://192.168.139.1:8080/rhcos-4.3.0-x86_64-metal.raw.gz coreos.inst.ignition_url=http://192.168.139.1:8080/bootstrap.ign ip=dhcp rd.neednet=1" \
  --os-type=linux \
  --os-variant=rhel8-unknown \
  --graphics spice \
  --network network=ocp4-ikw,mac=02:68:45:3c:3b:1d \
  --name bootstrap.ocp4-ikw.localdomain \
  --noreboot
```
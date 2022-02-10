# Ceph octupus in ubuntu focal.

# Requirement


- Install docker or podman in all node cluster.
- Install Docker Engine on Ubuntu

# Install using the repository

- SET UP THE REPOSITORY 

1. Update the apt package index and install packages to allow apt to use a repository over HTTPS:
```
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
2. Add Docker’s official GPG key:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
3. To add the nightly or test repository
```
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
- INSTALL DOCKER ENGINE
```
 sudo apt-get update
 sudo apt-get install docker-ce docker-ce-cli containerd.io
```
- Verify that Docker Engine
```
sudo docker run hello-world
```

# Deploy cephamd

- Install repo and download package resource
```
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x cephadm
```
```
wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
echo deb https://download.ceph.com/debian-octopus/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
sudo apt-get update
```
- install the ceph-common and the cephadm tools
```
sudo ./cephadm install cephadm ceph-common
```
- Bootstrap your Cluster
```
sudo mkdir -p /etc/ceph
sudo ./cephadm bootstrap --mon-ip <ip>
```
note: if issue pls skip network mon
```
sudo ./cephadm bootstrap --mon-ip <ip> --skip-mon-network
```
- CLI
```
./cephadm shell
ceph status
```
- ADD Network
```
sudo ceph config set mon public_network 10.0.0.0/16
```
# Expending the Cluster

- Copy the ceph ssh key from your manager node into your new server
```
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node02
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node03
```
- ADD host
```
sudo ceph orch host add node03
sudo ceph orch host add node03
```
- ADD monitor
```
sudo ceph orch apply mon 3 ( set monitor trên 3 node)
```
Đối add ceph mon nếu thực hiện command 
```
sudo ceph orch apply mon node01
sudo ceph orch apply mon node02
sudo ceph orch apply mon node03
```
Thì command trên đầu tiên sẽ add mon trên node01, thực hiện add mon02 thì ceph mon sẽ chuyển qua mon02, mon01 se bị out. tương tự cho ceph mon 03.

Thực hiện add ceph mon bang file YAML hoac dong thoi 3 node.
```
vim file.yaml
```
```
service_type: mon
placement:
  hosts:
   - host1
   - host2
   - host3
```
Thực hiện deloy
```
ceph orch apply -i file.yaml
```

- Adding Storage

Kiem tra cac device dang co san
```
sudo ceph orch device ls
``` 
add vào ceph To add a device of a cluster node run:
```
sudo ceph orch daemon add osd node1:/dev/[sdb]
```
Xóa data node trước khi add vào cụm
```
sudo ceph orch device zap node1 /dev/[sdb] --force
```
add tất cả OSD có trong cụm hiện tại
```
ceph orch apply osd --all-available-devices
```
- Verify Cluster Status
```
ceph -s
```

# Thực hiện tạo pool cho openebula cung cấp Block Storage cho Openebula

- Create pool
```
ceph osd pool create one 512 512 
ceph osd lspools
```
note
```
Đối với tạo pool cần tính toán số lượng PG tạo ra để tối ưu performance và tinh replicate để tối ưu ceph
Thương nếu có < 5 OSDs thì PG tạo là 128
5 < OSDs < 10 số PG tạo là 512
Lớn hơn  10 PG tạo 1024
```
Define a Ceph user to access the datastore pool; this user will also be used by libvirt to access the disk images

```
ceph auth get-or-create client.libvirt \
      mon 'profile rbd' osd 'profile rbd pool=one'
```
Get a copy of the key of this user to distribute it later to the OpenNebula nodes
```
ceph auth get-key client.libvirt | tee client.libvirt.key
ceph auth get client.libvirt -o ceph.client.libvirt.keyring
```
Although RBD format 1 is supported it is strongly recommended to use Format 2. Chỉnh sửa cepf.conf

```
[global]
rbd_default_format = 2
```
# Frontend and Node Setup
- The mon daemon must be defined in the ceph.conf for all the nodes, so hostname and port doesn’t need to be specified explicitly in any Ceph command.
- Copy the Ceph user keyring (ceph.client.libvirt.keyring) to the nodes under /etc/ceph, and the user key (client.libvirt.key) to the oneadmin home.
```
scp ceph.client.libvirt.keyring root@node02:/etc/ceph
scp ceph.client.libvirt.keyring root@node03:/etc/ceph
scp client.libvirt.key oneadmin@node02:
scp client.libvirt.key oneadmin@node03:
```
Generate a secret for the Ceph user and copy it to the nodes under oneadmin home. Write down the UUID for later use

```
$ UUID=`uuidgen`; echo $UUID
$ cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>$UUID</uuid>
  <usage type='ceph'>
          <name>client.libvirt secret</name>
  </usage>
</secret>
EOF
$ scp secret.xml oneadmin@node02:
$ scp secret.xml oneadmin@node03:
```
Define the a libvirt secret and remove key files in the nodes:
```
virsh -c qemu:///system secret-define secret.xml
virsh -c qemu:///system secret-set-value --secret $UUID --base64 $(cat client.libvirt.key)
rm client.libvirt.key
```
# Create a System Datastore

- Create a System Datastore in Sunstone or through the CLI
```
$ cat systemds.txt
NAME    = ceph_system
TM_MAD  = ceph
TYPE    = SYSTEM_DS

POOL_NAME   = one
CEPH_HOST   = "host1 host2:port2"
CEPH_USER   = libvirt
CEPH_SECRET = "6f88b54b-5dae-41fe-a43e-b2763f601cfc"

BRIDGE_LIST = cephfrontend

$ onedatastore create systemds.txt
```
# Create an Image Datastore
- example of datastore:
```
$ cat ds.conf
NAME = "cephds"
DS_MAD = ceph
TM_MAD = ceph

DISK_TYPE = RBD

POOL_NAME   = one
CEPH_HOST   = "host1 host2:port2"
CEPH_USER   = libvirt
CEPH_SECRET = "6f88b54b-5dae-41fe-a43e-b2763f601cfc"

BRIDGE_LIST = cephfrontend

$ onedatastore create ds.conf

```
# DONE, Kiem tra frontend và check lại


Các nguon tham khao :

https://docs.ceph.com/en/latest/cephadm/
https://docs.opennebula.io/5.12/deployment/open_cloud_storage_setup/ceph_ds.html

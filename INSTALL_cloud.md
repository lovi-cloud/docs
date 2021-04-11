# QuickStart lovi-cloud

## Goal of this document

- Launch [satelit](https://github.com/lovi-cloud/satelit) and [teleskop](https://github.com/lovi-cloud/teleskop)
- Create and Start Virtual Machine

## Prepare

- A server of controller and hypervisor
    - e.g.) bare-metal, virtual-machine with vmx(Intel) or svm(AMD)

- MySQL Server (recommend a version: > 8)

e.g.) using Docker

```bash
$ docker run -d -p 3306:3306 --restart always -e MYSQL_ROOT_PASSWORD="password" mysql:8 --default-authentication-plugin=mysql_native_password
```

### Servers

| Type | Name | IP Address | distribution | 
| --- | --- | --- | --- | 
| controller (satelit, MySQL) | admin001 | 192.0.2.20 | Ubuntu focal |
| agent (teleskop) | hv001 | 192.0.2.100 | Ubuntu focal |
| iSCSI target | storage001 | 192.0.2.200 | Ubuntu focal |

A distribution that we tested is here.

- Ubuntu xenial (16.04)
- Ubuntu focal (20.04)

## Install

### [open-iscsi/targetd](https://github.com/open-iscsi/targetd)

#### 1. Install [OpenZFS](https://github.com/openzfs/zfs)

```bash
user@storage001 $ sudo apt install zfsutils-linux
```

#### 2. Create ZFS pool and dataset

```bash
user@storage001 $ sudo zpool create tank
user@storage001 $ sudo zfs create tank/targetd
```

#### 3. Put targetd.yaml

for example, please read the newest documents in [repository of targetd](https://github.com/open-iscsi/targetd#configuring-targetd) 

```bash
user@storage001 $ cat /etc/target/targetd.yaml
user: "foo" # strings quoted, or not
password: bar
ssl: false
target_name: iqn.1992-01.com.example:targetd

zfs_block_pools: ["tank/targetd"]
block_pools: []
zfs_enable_copy: true

portal_addresses: ["0.0.0.0"]
```

#### 4. Build and Run with Docker

```bash
user@storage001 $ docker build -t targetd -f docker/Dockerfile.zfs .
user@storage001 $ sudo docker run -d --net=host --privileged -v /etc/target:/etc/target -v /sys/kernel/config:/sys/kernel/config -v /lib/modules:/lib/modules -v /dev:/dev targetd
```

### [satelit](https://github.com/lovi-cloud/satelit)

#### 1. Generate initiator name for iSCSI

```bash
user@admin001 $ sudo cat /etc/iscsi/initiatorname.iscsi
GenerateName=yes
user@admin001 $ sudo systemctl restart iscsid
user@admin001 $ sudo cat /etc/iscsi/initiatorname.iscsi
## DO NOT EDIT OR REMOVE THIS FILE!
## If you remove this file, the iSCSI daemon will not start.
## If you change the InitiatorName, existing access control lists
## may reject this initiator.  The InitiatorName must be unique
## for each iSCSI initiator.  Do NOT duplicate iSCSI InitiatorNames.
InitiatorName=iqn.1993-08.org.debian:01:admin001
```

#### 2. Install apt packages

```bash
user@admin001 $ apt install qemu-utils
```

#### 3. Put satelit.yaml

please read the newest sample in [repository of satelit](https://github.com/lovi-cloud/satelit/blob/master/configs/satelit.yaml.sample).

```bash
user@admin001 $ cat satelit.yaml
# config of listen ports
api:
  listen: "0.0.0.0:9262"
datastore:
  listen: "0.0.0.0:9263"

# list of hypervisor that installed teleskop.
# satelit will register hosts in boot sequence.
teleskop:
  endpoints:
    host1: "192.0.2.100:5000"

# config of MySQL Server as the backend of datastore
mysql:
  dsn: "root:password@tcp(127.0.0.1:3306)/satelit"
  max_idle_conn: 80
  conn_max_lifetime_second: 60

# config of targetd as the backend of europa
targetd:
  - api_endpoint: "http://192.0.2.200:18700"
    username: "foo"
    password: "bar"
    pool_name: "tank/targetd"
    backend_name: "targetd"
    portal_ip: "192.0.2.200"

# config of the log level
log_level: "debug"
```

#### 4. Create and apply datastore

Create database

```bash
user@admin001 $ sudo mysql  # connect to MySQL in localhost
mysql> CREATE DATABASE satelit;
Query OK, 1 row affected (0.01 sec)
```

apply `schema.sql`

```bash
user@admin001 $ curl -L -0 https://raw.githubusercontent.com/lovi-cloud/satelit/master/internal/mysql/schema.sql
user@admin001 $ cat schema.sql | sudo mysql -uroot -p
```

#### 5. Build satelit

```bash
user@admin001 $ git clone https://github.com/lovi-cloud/satelit
user@admin001 $ cd satelit
user@admin001 $ make build-linux
```

#### 6. Execute satelit

`satelit revision` is commit hash of satelit when build

```bash
user@admin001 $ sudo ./satelit-linux-amd64 -conf satelit.yaml
satelit revision: 4530d54
```

### [teleskop](https://github.com/lovi-cloud/teleskop)

#### 1. Generate initiator name for iSCSI

```bash
user@hv001 $ sudo cat /etc/iscsi/initiatorname.iscsi
GenerateName=yes
user@hv001 $ sudo systemctl restart iscsid
user@hv001 $ sudo cat /etc/iscsi/initiatorname.iscsi
## DO NOT EDIT OR REMOVE THIS FILE!
## If you remove this file, the iSCSI daemon will not start.
## If you change the InitiatorName, existing access control lists
## may reject this initiator.  The InitiatorName must be unique
## for each iSCSI initiator.  Do NOT duplicate iSCSI InitiatorNames.
InitiatorName=iqn.1993-08.org.debian:01:hv001
```

#### 2. Install apt packages

```bash
user@hv001 $ curl "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x5edb1b62ec4926ea" | sudo apt-key add -
user@hv001 $ sudo add-apt-repository 'deb http://ubuntu-cloud.archive.canonical.com/ubuntu focal-updates/wallaby main'
user@hv001 $ sudo apt update
user@hv001 $ sudo apt install qemu-system qemu-kvm qemu-utils bridge-utils ca-certificates libvirt-clients libvirt-daemon libvirt-daemon-system libvirt0
```

#### 3. Modify config of libvirtd

teleskop connect to libvirtd using tcp.

```bash
user@hv001 $ sudo cat /etc/libvirt/libvirtd.conf | grep -vE "^$|^#"
listen_tls = 0
listen_tcp = 1
tcp_port = "16509"
listen_addr = "0.0.0.0"
unix_sock_group = "libvirt"
unix_sock_ro_perms = "0777"
unix_sock_rw_perms = "0770"
auth_unix_ro = "none"
auth_unix_rw = "none"
auth_tcp = "none"
auth_tls = "none"

user@hv001 $ sudo vim /etc/systemd/system/multi-user.target.wants/libvirtd.service
# Add Wants=libvirtd-tcp.socket
Wants=libvirtd-ro.socket
+Wants=libvirtd-tcp.socket
Wants=libvirtd-admin.socket

user@hv001 $ sudo systemctl restart libvirtd
```

#### 4. Put teleskop

Put systemd unit file

```bash
user@hv001 $ cat /etc/systemd/system/teleskop.service
[Unit]
Description=Teleskop Agent

[Service]
User=root
ExecStart=/usr/local/bin/teleskop -satelit 192.0.2.20:9263 -intf eth0
Restart=always

[Install]
WantedBy=multi-user.target
user@hv001 $ sudo systemctl daemon-reload
```

Download the newest binary from [release page of teleskop](https://github.com/lovi-cloud/teleskop/releases)

```bash
user@hv001 $ ls /usr/local/bin/teleskop
/usr/local/bin/teleskop
```

#### 5. Execute teleskop

```bash
user@hv001 $ sudo systemctl start teleskop

user@hv001 $ sudo journalctl -f -u teleskop
Mar 00 00:00:00 hv001 systemd[1]: Started Teleskop Agent.
Mar 00 00:00:00 hv001 teleskop[15274]: connect to libvirtd version = 6000000
Mar 00 00:00:00 hv001 teleskop[15274]: listening on address 0.0.0.0:80
Mar 00 00:00:00 hv001 teleskop[15274]: listening on address :5000
Mar 00 00:00:00 hv001 teleskop[15274]: listening on address 0.0.0.0:67
```

## Start Virtual-Machine

satelit API implemented by [gRPC](https://grpc.io/).
You can call gRPC, a simple client is [here](https://github.com/lovi-cloud/satelit/tree/master/examples/client).

### Prepare

- a simple client
  - `client` is binary of simple client
- Image file in qcow2 for OS Image
  - You can get an image of cirros (testing OS Image) in [GitHub Releases](https://github.com/cirros-dev/cirros/releases).
  
e.g.)

```bash
user@your-workspace $ curl -L -O https://github.com/cirros-dev/cirros/releases/download/0.5.2/cirros-0.5.2-x86_64-disk.img
```

### Upload Image

satelit needs `Image` before creating a virtual machine.

`client upload-image` uploads OS image from your computer.

```bash
user@your-workspace $ client upload-image --address 192.0.2.20:9262 --backend targetd --image ./cirros-0.5.2-x86_64-disk.img 
UploadImage
id:"<Image UUID>" name:"cirros-0.5.2-x86_64-disk" volume_id:"<Volume UUID>" description:"md5:<md5 hash of qcow2 file>"
GetImages
id:"<Image UUID>" name:"cirros-0.5.2-x86_64-disk" volume_id:"<Volume UUID>" description:"md5:<md5 hash of qcoe2 file>"
```

please save `<Image UUID>` in your notepad.

### Create and Start the virtual machine

`client create-vm` create and start a virtual machine from `<Image UUID>`.

```bash
user@your-workspace $ client create-vm --address 192.0.2.20:9262 --backend targetd --name cirros-test --image <Image UUID> --hypervisor <hv001>
AddVirtualMachine
StartVirtualMachine
uuid:"<Virtual Machine UUID>" name:"cirros-test"
```

You can check virtual machine status by `virsh`.

```bash
user@hv001 $ sudo virsh list --all
 Id   Name          State
-----------------------------
 1    cirros-test   running
```

Gotcha!

## More parameters

please see: [terraform-provider-lovi](https://github.com/lovi-cloud/terraform-provider-lovi).

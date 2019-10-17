# OpenStack queens版本的kolla部署

### 确认系统版本信息
#cat /etc/redhat-release

CentOS Linux release 7.4.1708 (Core)

#uname -r
3.10.0-693.5.2.el7.x86_64
### 更新pip
#pip install --upgrade pip
### 网卡配置
#cd /etc/sysconfig/network-scripts
>> ens160:
        TYPE="Ethernet"<br>
        PROXY_METHOD="none"<br>
        BROWSER_ONLY="no"<br>
        BOOTPROTO="none"<br>
        DEFROUTE="yes"<br>
        IPV4_FAILURE_FATAL="no"<br>
        IPV6INIT="yes"<br>
        IPV6_AUTOCONF="yes"<br>
        IPV6_DEFROUTE="yes"<br>
        IPV6_FAILURE_FATAL="no"<br>
        IPV6_ADDR_GEN_MODE="stable-privacy"<br>
        NAME="ens160"<br>
        UUID="cb175c17-9c1a-40dc-a018-8dee550552c5"<br>
        DEVICE="ens160"<br>
        ONBOOT="yes"<br>
        IPADDR="10.89.127.124"<br>
        PREFIX="24"<br>
        GATEWAY="10.89.127.254"<br>
        DNS1="114.114.114.114"<br>
        IPV6_PRIVACY="no"<br>

>> ens192:
        TYPE=Ethernet<br>
        PROXY_METHOD=none<br>
        BROWSER_ONLY=no<br>
        BOOTPROTO=dhcp<br>
        DEFROUTE=yes<br>
        IPV4_FAILURE_FATAL=no<br>
        IPV6INIT=yes<br>
        IPV6_AUTOCONF=yes<br>
        IPV6_DEFROUTE=yes<br>
        IPV6_FAILURE_FATAL=no<br>
        IPV6_ADDR_GEN_MODE=stable-privacy<br>
        NAME=ens192<br>
        UUID=0d0220be-2879-40a9-8ad8-2020c8d59fd4<br>
        DEVICE=ens192<br><br>
        ONBOOT=no<br>
        
* 上面可以看出有两张网卡ens160和ens192，这里我用ens160做管理网，ens192做业务网，这里不需要配置ip，把ens160网卡up起来就好。

### 查看主机名
#hostname<br>
queens<br>
## 环境初始化
### 关闭NetworkManager,firewalld,selinux
#systemctl stop NetworkManager<br>
#systemctl disable NetworkManager<br>

>> Removed symlink /etc/systemd/system/multi-user.target.wants/NetworkManager.service.<br>
>> Removed symlink /etc/systemd/system/dbus-org.freedesktop.NetworkManager.service.<br>
>> Removed symlink /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service.<br>

#systemctl stop firewalld<br>
#systemctl disable firewalld<br>

>> Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.<br>
>> Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.<br>

#sed -i "s/SELINUX=enforcing/SELINUX=disabled/" /etc/selinux/config<br>

#setenforce 0<br>
#getenforce<br>

>> Permissive<br>

### 查看是否开启了虚拟化
#egrep "vmx|svm" /proc/cpuinfo<br>

### 配置配置epel源安装基础包
#yum install epel-release<br>
#yum install axel vim git curl wget lrzsz gcc  python-devel python-pip<br>
### 安装配置docker
### 安装docker
#wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo<br>
#yum install -y docker-ce<br>
### 配置docker
#mkdir /etc/systemd/system/docker.service.d<br>
#tee /etc/systemd/system/docker.service.d/kolla.conf << 'EOF'<br>

[Service]<br>
MountFlags=shared<br>
EOF<br>

#vim /usr/lib/systemd/system/docker.service<br>

#ExecStart=/usr/bin/dockerd<br>
>> ExecStart=/usr/bin/dockerd --registry-mirror=http://f2d6cb40.m.daocloud.io --storage-driver=overlay2<br>

* 这里docker的文件系统我用overlay2

### 启动docker
#systemctl daemon-reload<br>
#systemctl restart docker<br>
#systemctl enable docker<br>

>> Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.

#systemctl status docker<br>
#docker info<br>
### 安装ansible
ansible版本必须在2.0以上<br>
#yum -y install ansible -y<br>
### 下载kolla-ansible，并安装配置
#git clone https://github.com/openstack/kolla-ansible -b stable/queens<br>
#cd kolla-ansible/<br>
#cp -r etc/kolla/ /etc/kolla/<br>
#pip install . -i https://pypi.tuna.tsinghua.edu.cn/simple<br>
### 配置globals.yml文件
#vim /etc/kolla/globals.yml<br>

---<br>
kolla_base_distro: "centos"<br>
kolla_install_type: "source"<br>
openstack_release: "queens"<br>
kolla_internal_vip_address: "10.89.127.124"<br>
docker_namespace: "kolla"<br>
network_interface: "ens160"<br>
neutron_external_interface: "ens192"<br>
enable_haproxy: "no"<br>
nova_compute_virt_type: "qemu"<br>
ironic_dnsmasq_dhcp_range:<br>
tempest_image_id:<br>
tempest_flavor_ref_id:<br>
tempest_public_network_id:<br>
tempest_floating_network_name:<br>

freezer部分<br>
enable_freezer:"yes"<br>
enable_heat:"yes"<br>
enable_horizon_freezer:"{{ enable_freezer | bool }}"<br>


说明：这里我直接在docker hub上拉镜像。如果是在虚拟机里安装 Kolla，希望可以在 OpenStack 平台上创建虚拟机，<br>
那么你需要在 globals.yml 文件中把 nova_compute_virt_type 配置项设置为 qemu，默认是 KVM。<br>

### 安装kolla
生成密码文件<br>
#kolla-genpwd<br>
编辑 /etc/kolla/passwords.yml 文件，配置 keystone 管理员用户的密码。<br>
keystone_admin_password: 登陆密码<br>
同时，也是登录 Dashboard，admin 使用的密码，你可以根据自己需要进行修改。<br>

运行 prechecks 检查配置是否正确，如果有错误，可以先忽略。<br>
#kolla-ansible prechecks<br>
从docker hub上pull镜像<br>
#kolla-ansible pull<br>
部署openstack<br>
#kolla-ansible deploy<br>
创建环境变量文件<br>
#kolla-ansible post-deploy<br>
这样就创建了/etc/kolla/admin-openrc.sh 环境变量文件。<br>

### 安装 OpenStack Client 端<br>
#pip install --ignore-installed python-openstackclient<br>
编辑init-runonce文件,设置public network<br>
#vim /usr/share/kolla-ansible/init-runonce<br>

EXT_NET_CIDR='10.89.127.0/24'<br>
EXT_NET_RANGE='start=10.89.127.110,end=10.89.127.254'<br>
EXT_NET_GATEWAY='10.89.127.1'<br>

加载OpenStack CLI所需的环境变量<br>
#source /etc/kolla/admin-openrc.sh<br>
初始化部署<br>
#cd /usr/share/kolla-ansible/ && ./init-runonce<br>
登陆Dashboard<br>
用浏览器访问10.89.127.124登陆Dashboard<br>

## 部署额外组件
### swift部分<br> （存疑）
打swift标签<br>
#parted /dev/sdb -s -- mklabel gpt mkpart KOLLA_SWIFT_DATA 1 -1 <br>
生成rings，执行如下脚本<br>
STORAGE_NODES=(10.89.127.124) <br>
KOLLA_SWIFT_BASE_IMAGE="kolla/centos-source-swift-base:queens" <br>
mkdir -p /etc/kolla/config/swift <br>

#Object ring <br>
docker run \ <br>
  --rm \ <br>
  -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \ <br>
  $KOLLA_SWIFT_BASE_IMAGE \ <br>
  swift-ring-builder \ <br>
    /etc/kolla/config/swift/object.builder create 10 3 1 <br>
for node in ${STORAGE_NODES[@]}; do <br>
    for i in {0..2}; do <br>
      docker run \ <br>
        --rm \ <br>
        -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \ <br>
        $KOLLA_SWIFT_BASE_IMAGE \ <br>
        swift-ring-builder \ <br>
          /etc/kolla/config/swift/object.builder add r1z1-${node}:6000/d${i} 1; <br>
    done <br>
done <br>

#Account ring <br>
docker run \ <br>
  --rm \ <br>
  -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \ <br>
  $KOLLA_SWIFT_BASE_IMAGE \ <br>
  swift-ring-builder \ <br>
    /etc/kolla/config/swift/account.builder create 10 3 1 <br>
for node in ${STORAGE_NODES[@]}; do <br>
    for i in {0..2}; do <br>
      docker run \ <br>
        --rm \ <br>
        -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \ <br>
        $KOLLA_SWIFT_BASE_IMAGE \ <br>
        swift-ring-builder \ <br>
          /etc/kolla/config/swift/account.builder add r1z1-${node}:6001/d${i} 1; <br>
    done <br>
done <br>

#Container ring <br>
docker run \ <br>
  --rm \ <br>
  -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \ <br>
  $KOLLA_SWIFT_BASE_IMAGE \ <br>
  swift-ring-builder \ <br>
    /etc/kolla/config/swift/container.builder create 10 3 1 <br>
for node in ${STORAGE_NODES[@]}; do <br>
    for i in {0..2}; do <br>
      docker run \ <br>
        --rm \ <br>
        -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \ <br>
        $KOLLA_SWIFT_BASE_IMAGE \ <br>
        swift-ring-builder \ <br>
          /etc/kolla/config/swift/container.builder add r1z1-${node}:6002/d${i} 1; <br>
    done <br>
done <br>
for ring in object account container; do <br>
  docker run \ <br>
    --rm \ <br>
    -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \ <br>
    $KOLLA_SWIFT_BASE_IMAGE \ <br>
    swift-ring-builder \ <br>
      /etc/kolla/config/swift/${ring}.builder rebalance; <br>
done <br>
配置globals文件<br>
#vim /etc/kolla/globals.yml<br>

#enable_ceph: "no"<br>
#enable_ceph_rgw: "no"<br>
enable_swift: "yes"<br>
#enable_ceph_rgw_keystone: "no"<br>

执行部署<br>
#kolla-ansible deploy<br>

### ceph部分<br>
关闭所有swift的docker
创建 /etc/kolla/config/ceph.conf<br>
打cepf标签<br>
#parted /dev/sdc -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP 1 -1<br>
#vim /etc/kolla/globals.yml<br>

enable_ceph: "yes"<br>
enable_ceph_rgw: "yes"<br>
#enable_swift: "no"<br>
enable_ceph_rgw_keystone: "yes"<br>

执行部署<br>
#kolla-ansible deploy<br>

### cinder部分<br>
#vim /etc/kolla/globals.yml<br>

enable_cinder: "yes"<br>

执行部署<br>
#kolla-ansible deploy<br>

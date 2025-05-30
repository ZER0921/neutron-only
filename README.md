# 如何单独运行Neutron服务？

Neutron是OpenStack的网络组件，可以作为软网元的控制器，实现一套完整的软SDN方案。<br>
有时出于学习的目的，需要将Neutron跑起来，从而学习Neutron的内部实现细节和软SDN的底层实现原理，但又不想部署整套OpenStack系统。怎么办？<br>
Neutron本质上是一个微服务，只需要将其与其他微服务之间的依赖解耦开，即可单独运行和调试Neutron服务。

## 准备环境

本文使用的操作系统为Ubuntu 22.04，系统自带的Python版本为3.10

### 部署视图

- 控制节点
  - neutron-server + mysql-server + rabbitmq-server

- 计算节点
  - neutron-openvswitch-agent (允许开启安全组) + openvswitch + conntrack

- 网络节点
  - neutron-openvswitch-agent (禁止开启安全组) + openvswitch + conntrack
  - neutron-dhcp-agent + dnsmasq-base/dnsmasq-utils
  - neutron-l3-agent + keepalived + haproxy + iputils-arping

### 基础组件

可以使用如下命令安装和配置mysql-server/rabbitmq-server
> openvswitch/conntrack等组件也是需要安装的，此次不再赘述

- mysql-server

```bash
apt install -y mysql-server

mysql
> CREATE USER dbadmin IDENTIFIED BY '123456';
> GRANT ALL ON *.* TO dbadmin;
> CREATE DATABASE neutron;
> quit

mysql -u dbadmin -p123456 neutron
```

- rabbitmq-server

```bash
apt install -y rabbitmq-server

rabbitmqctl add_user mqadmin 123456
rabbitmqctl set_permissions -p / mqadmin '.*' '.*' '.*'
rabbitmqctl set_user_tags mqadmin administrator

rabbitmqctl list_users
```

## 代码适配

### 下载代码

下载如下版本的代码
- neutron: 15.3.4
- neutron-lib: 1.29.2
- neutron-fwaas: 15.0.1
- python-neutronclient: 6.14.1
> 都是Train版本下的小版本

假定相关的代码路径如下
- /opt/neutron-only/
- /opt/neutron/
- /opt/neutron-lib/
- /opt/neutron-fwaas/
- /opt/python-neutronclient/

### 打补丁

将[patches](patches)目录的补丁文件依次应用到neutron/neutron-lib/neutron-fwaas/python-neutronclient源码上<br>

以neutron为例
```bash
cd /opt/neutron
git apply -p1 /opt/neutron-only/patches/neutron/*
```

补丁修改内容主要包括
- 支持noauth
- 适配Python 3.10和SQLAlchemy 1.4
- bugfix
- 调试验证feature
- 配置文件
- 辅助脚本

### Noauth

API的token认证依赖于keystone服务，在仅包含neutron的场景下，需要配置noauth认证策略

- neutron-server适配noauth
  - 在配置文件neutron.conf中指定noauth认证策略
  - request_id适配noauth：配置noauth认证时，API响应头中的request_id与后台日志中的request_id不一致，影响问题定位

- neutronclient适配noauth
neutronclient出于安全原因，默认禁止noauth认证策略

- shell适配noauth
noauth认证场景以admin身份操作资源，因此创建资源接口必须显式指定project_id参数，表示资源的所有者

### 示例配置文件

配置文件是基于示例配置文件修改得到的

在激活虚拟环境并安装依赖包后，在neutron/neutron-fwaas的根目录下执行如下命令，即会在etc目录下生成示例配置文件(*.sample)
```bash
./tools/generate_config_file_samples.sh
```

## 安装服务

### Python虚拟环境

基于Python虚拟环境进行开发，避免影响系统环境

```bash
cd /opt
python -m venv app
source app/bin/activate
```

### 安装依赖包

neutron组件版本和依赖包版本之间有配套关系，因此需要固定版本号
- 安装neutron和neutron-fwaas组件的依赖即可
- 需要额外安装数据库driver，例如pymysql

```bash
pip install -r /opt/neutron-only/patches/requirements.txt
```

### 安装Neutron组件

安装顺序: neutron-lib, python-neutronclient, neutron, neutron-fwaas
<br>
以neutron-lib为例
```bash
cd /opt/neutron-lib
python setup.py install
```

### 初始化操作

```bash
ln -s /opt/neutron/etc/neutron /etc/neutron
mkdir -p /var/log/neutron

export OS_LOCAL_IP=<local-vtep-ip>
export OS_PROJECT_ID=<test-project-id>
export OS_URL=http://127.0.0.1:9696/v2.0
```

## 运行服务

### 配置文件

```conf
- neutron-server
/etc/neutron/api-paste.ini
/etc/neutron/neutron.conf
/etc/neutron/plugins/ml2/ml2_conf.ini

- neutron-*-agent
/etc/neutron/rootwrap.conf
/etc/neutron/neutron_*.conf
/etc/neutron/plugins/*/*_agent.ini
```

### 启动Server

- 升级数据库

```bash
neutron-db-manage --config-file /etc/neutron/neutron.conf --subproject neutron upgrade head
neutron-db-manage --config-file /etc/neutron/neutron.conf --subproject neutron-fwaas upgrade head
```

- 拉起进程

```bash
neutron-server --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini
```

### 启动Agent

- ovs-agent

```bash
ovs-vsctl add-br br-ext
neutron-openvswitch-agent --config-file /etc/neutron/neutron_ovs.conf --config-file /etc/neutron/plugins/ml2/openvswitch_agent.ini
```

- dhcp-agent

```bash
neutron-dhcp-agent --config-file /etc/neutron/neutron_dhcp.conf --config-file /etc/neutron/plugins/dhcp/dhcp_agent.ini
```

- l3-agent

```bash
neutron-l3-agent --config-file /etc/neutron/neutron_l3.conf --config-file /etc/neutron/plugins/l3/l3_agent.ini
```

### 使用CLI

```bash
# 命令选项会直接转换为REST请求参数
#   --flag-name         -> 'flag_name':true
#   --flag-name=false   -> 'flag_name':'false' -> neutron-server的api框架会自动将string转换成boolean
#   --op-name op-value  -> 'op_name':'op-value'
# 命令选项中的中划线-和下划线_是等价的
#   例如，--project-id和--project_id是等价的
# 帮助信息中并未列出全部选项，部分API请求字段可以直接添加为命令选项
#   例如，net-create命令的--router:external选项

neutron help
neutron agent-list
```

## 调试验证

### 辅助脚本

- neutron-appctl: 服务进程运行控制
- neutron-curl: 针对neutron资源抽象的curl命令封装
- neutron-ps: neutron相关进程ps
- neutron-iface: "虚拟机网卡"接入虚拟网桥模拟
- netns-run: 更便利的netns运行命令
- ovs-dump-flows: 更简洁的流表查询命令

### Layer 2 Networking

创建network和subnet

```bash
neutron net-create network1
neutron subnet-create --name subnet1 --ip-version 4 network1 192.168.1.0/24
```

创建port并绑定到主机

```bash
PORT=port1
NETWORK=network1
VM_ID=$(uuidgen)
HOST_ID=$(hostname)
SECURITY=false
neutron port-create --name $PORT --device-id $VM_ID --device-owner compute:nova --binding:host-id=$HOST_ID --port-security-enabled=$SECURITY $NETWORK
```

在OpenStack中，虚拟机网卡是由Nova添加到虚拟网桥上的，我们使用neutron-iface脚本来模拟这个行为

```bash
neutron-iface add vm1 <port-id> <port-mac>
```

确认port状态为ACTIVE，表示port上线成功

```bash
neutron port-show port1 -c status
```

### Layer 3 Networking

TODO

### Security

TODO


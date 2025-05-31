# 如何单独运行Neutron服务？

Neutron是OpenStack的网络组件，可以作为软网元的控制器，实现一套完整的软SDN方案。
有时出于学习的目的，需要将Neutron跑起来，从而学习Neutron的内部实现细节和软SDN的底层实现原理，但又不想部署整套OpenStack系统。怎么办？
Neutron本质上是一个微服务，只需要将其与其他微服务之间的依赖解耦开，即可单独运行和调试Neutron服务。

## 准备环境

安装Ubuntu 22.04系统
- python 3.10

下载Train版本的如下neutron组件代码
- neutron: 15.3.4
- neutron-lib: 1.29.2
- neutron-fwaas: 15.0.1
- python-neutronclient: 6.14.1

以下假定相关的代码路径如下
- /opt/neutron-only/
- /opt/neutron/
- /opt/neutron-lib/
- /opt/neutron-fwaas/
- /opt/python-neutronclient/

## 代码适配修改

```bash
git apply -p1 /opt/neutron-only/patches/neutron-lib/*
```

### Noauth
API的token认证依赖于keystone服务，在仅包含neutron的场景下，需要配置noauth认证策略

- neutron-server适配noauth

```
- noauth配置项
''' neutron.conf
auth_strategy = noauth

- request_id适配noauth
配置noauth认证时，API响应头中的request_id与后台日志中的request_id不一致，影响问题定位

''' api-paste.ini
[composite:neutronapi_v2_0]
noauth = ... noauthcontext ...

[filter:noauthcontext]
paste.filter_factory = neutron.auth:NeutronNoauthContext.factory

''' neutron/auth.py
from oslo_middleware import request_id

class NeutronNoauthContext(base.ConfigurableMiddleware):
    """Make a request context for noauth."""

    @webob.dec.wsgify
    def __call__(self, req):
        req_id = req.environ.get(request_id.ENV_REQUEST_ID)
        ctx = context.get_admin_context_with_request_id(req_id)
        req.environ['neutron.context'] = ctx

        return self.application

''' neutron_lib/context.py
def get_admin_context_with_request_id(request_id):
    return Context(user_id=None,
                   tenant_id=None,
                   is_admin=True,
                   request_id=request_id,
                   overwrite=False)
```

- neutronclient适配noauth
neutronclient出于安全原因，默认禁止noauth认证策略

- shell适配noauth
noauth认证场景以admin身份操作资源。因此，创建资源接口必须显式指定project_id参数，表示资源的所有者
export OS_PROJECT_ID=xxx


## Standalone

* Python虚拟环境
python -m venv app
source app/bin/activate

* 依赖包
pip install -r requirements.txt
> 安装neutron和neutron-fwaas仓中的依赖即可
> SQLAlchemy必须使用1.4.x
> 需要额外安装数据库driver，例如pymysql

* Neutron组件
安装顺序: neutron-lib, python-neutronclient, neutron, neutron-fwaas

* 配置文件
- sample
在neutron/neutron-fwaas的根目录下执行
bash tools/generate_config_file_samples.sh
- neutron-server
/etc/neutron/api-paste.ini
/etc/neutron/neutron.conf
/etc/neutron/plugins/ml2/ml2_conf.ini
- neutron-*-agent
/etc/neutron/rootwrap.conf
/etc/neutron/neutron_*.conf
/etc/neutron/plugins/*/*_agent.ini

* 部署视图
- 控制节点
neutron-server + mysql-server + rabbitmq-server
- 计算节点
neutron-openvswitch-agent (允许开启安全组) + openvswitch + conntrack
- 网络节点
neutron-openvswitch-agent (禁止开启安全组) + openvswitch + conntrack
neutron-dhcp-agent + dnsmasq-base/dnsmasq-utils
neutron-l3-agent + keepalived + haproxy + iputils-arping

* 初始化操作
ln -s /opt/src/neutron/etc/neutron /etc/neutron
mkdir -p /var/log/neutron

export OS_LOCAL_IP=<local-vtep-ip>
export OS_PROJECT_ID=<test-project-id>
export OS_URL=http://127.0.0.1:9696/v2.0

* 启动Server
- 升级数据库
neutron-db-manage --config-file /etc/neutron/neutron.conf --subproject neutron upgrade head
neutron-db-manage --config-file /etc/neutron/neutron.conf --subproject neutron-fwaas upgrade head
- 拉起进程
neutron-server --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini

* 启动Agent
- ovs-agent
neutron-openvswitch-agent --config-file /etc/neutron/neutron_ovs.conf --config-file /etc/neutron/plugins/ml2/openvswitch_agent.ini
- dhcp-agent
neutron-dhcp-agent --config-file /etc/neutron/neutron_dhcp.conf --config-file /etc/neutron/plugins/dhcp/dhcp_agent.ini
- l3-agent
neutron-l3-agent --config-file /etc/neutron/neutron_l3.conf --config-file /etc/neutron/plugins/l3/l3_agent.ini

* 使用CLI
neutron xxx
- 命令选项会直接转换为REST请求参数
  --flag-name         -> 'flag_name':true
  --flag-name=false   -> 'flag_name':'false' -> neutron-server的api框架会自动将string转换成boolean
  --op-name op-value  -> 'op_name':'op-value'
- 命令选项中的中划线-和下划线_是等价的
  例如，--project-id和--project_id是等价的
- 帮助信息中并未列出全部选项，部分API请求字段可以直接添加为命令选项
  例如，net-create命令的--router:external选项



**Keepalive+Nginx双活（多活）**
===
1.为什么需要nginx双活
  - 解决负载均衡反向代理层的单点问题，让系统拥有更高的可用性
  - 如果你还不知道nginx是什么？有什么用，请关掉这个md。先学习一下nginx
---
2.nginx安装
  - nginx安装很简单。你可以使用yum安装，源码包编译安装，docker容器安装都是可以的。这不是本篇的重点略过。
---
3.keepalive是什么，有什么用，简单说下原理
  - keepalive使用vrrp协议，将多台服务器虚拟出一个ip地址，然后所有的实体地址都会和虚拟地址绑定。
  - 当主节点故障时，从节点就会接替主节点工作。从而实现高可用。
  - 当主节点恢复时，虚拟地址将切换回主节点
  - 使用keepalive实现nginx的双活是非常合适的
---
4.keepalive安装
  - 准备两台机器，同步好时间，设置好网络，linux最大文件数，更新好yum源。这里我使用的是CentOS7.6
    - 192.168.90.201 主节点
    - 192.168.90.202 从节点
  - 在两台机器上都使用yum安装keepalive
    - ```yum install keepalived```
  - 如果是内网环境，可以先使用
    - ```yum install --downloadonly --downloaddir=下载到的目录 keepalived```下载rpm包
    - 再复制到内网中使用```rpm -ivh *.rpm```安装
---
5.keepalive配置
  - 主节点
  - ```
    #这些是警报邮箱设置，不需要的可以不设置
    global_defs {
        notification_email {
            abc@qq.com
            }
        notification_email_from Alexandre.Cassen@firewall.loc
        smtp_server 1.1.1.1
        smtp_connect_timeout 30
        router_id LVS_3
    }
    vrrp_instance VI_1 {
        #设置节点为主节点
        state MASTER
        #这里填此服务器的网卡名
        interface ens33
        #这里填写0-255中的任意数字，划分出256个虚拟路由，我们主从服务器都要填写一样的值
        virtual_router_id 7
        #权重，主节点权重要至少比从节点高50
        priority 150
        #检查的时间间隔，默认1s
        advert_int 1
        authentication {
            #认证方式，支持PASS和AH，官方建议使用PASS
            auth_type PASS
            #认证的密码，主从机器要一致
            auth_pass 1111
        }
        #设置VIP，可以设置多个，用于切换时的地址绑定。格式：#<IPADDR>/<MASK> brd <IPADDR> dev <STRING> scope <SCOPT> label <LABEL>
        virtual_ipaddress {
            #一般我们就设置ip/子网掩码，即可。多个虚拟地址换行就行，不要在后面加逗号
            192.168.90.254/24
        }
    }
    ```
  - 从节点
  - ```
    #这些是警报邮箱设置，不需要的可以不设置
    global_defs {
        notification_email {
            abc@qq.com
        }
        notification_email_from Alexandre.Cassen@firewall.loc
        smtp_server 1.1.1.1
        smtp_connect_timeout 30
        router_id LVS_6
    }
    vrrp_instance VI_1 {
        #设置节点为从节点
        state BACKUP
        #这里填此服务器的网卡名
        interface ens33
        #这里填写0-255中的任意数字，划分出256个虚拟路由，我们主从服务器都要填写一样的值
        virtual_router_id 7
        #权重，主节点权重要至少比从节点高50
        priority 100
        #检查的时间间隔，默认1s
        advert_int 1
        authentication {
            #认证方式，支持PASS和AH，官方建议使用PASS
            auth_type PASS
            #认证的密码，主从机器要一致
            auth_pass 1111
        }
        #设置VIP，可以设置多个，用于切换时的地址绑定。格式：#<IPADDR>/<MASK> brd <IPADDR> dev <STRING> scope <SCOPT> label <LABEL>
        virtual_ipaddress {
            #一般我们就设置ip/子网掩码，即可。多个虚拟地址换行就行，不要在后面加逗号
            192.168.90.254/24
        }
    }
    ```
  - 配置keepalive的开启启动
  ```
  systemctl start keepalived
  systemctl enable keepalived
  ```
---
6.测试
  - 在nginx的默认主页index.html中加一行，此服务器的真实ip，方便观察
  - 使用虚拟ip```192.168.90.254```访问nginx,成功访问，显示的是192.168.90.201
  - 将主服务器关机，接着访问虚拟ip，成功访问，显示的是192.168.90.202
  - 将主服务器开机，接着访问虚拟ip，不停刷新浏览器，访问成功，一会儿后，显示的是192.168.90.201

# 1、配置生效

```bash
firewall-cmd --reload		# 重载防火墙的配置,立即生效
--permanent					# 重启防火墙后，永久有效
```



# 2、区域相关

```bash
# 查看防火墙预定义的区域有哪些：
# block dmz drop external home internal public trusted work
firewall-cmd --get-zones						

firewall-cmd --get-default-zone					# 查看当前默认区域
firewall-cmd --set-default-zone=public			# 更换默认区域
firewall-cmd --new-zone=athos --permanent		# 添加自定义区域

# 查询是否在指定区域中被定义策略
firewall-cmd --zone=public --query-interface=eth0	
firewall-cmd --zone=public --query-service=https

firewall-cmd --list-all 						# 查看区域信息
firewall-cmd --list-all --zone=ZoneName			# 查看指定区域信息

[root@m01 ~]# firewall-cmd --list-all
public (active)
  target: default					# 标策略为默认设置
  icmp-block-inversion: no			# 没有启用对 ICMP 消息的全面阻止
  interfaces: eth0 eth1				# 当前区域监听的网卡
  sources: 							# 针对源IP做策略
  services: dhcpv6-client ssh		# 允许访问的服务
  ports: 							# 允许访问的端口
  protocols: 						# 运行访问的协议
  masquerade: no					# IP伪装
  forward-ports: 					# 端口转发
  source-ports: 					# 源端口
  icmp-blocks: 						# 控制ICMP消息过滤
  rich rules: 						# 富规则
```



# 3、服务相关

```bash
# 查看firewalld中所有预定义的服务
firewall-cmd --get-services						

# 列出指定区域中放行的服务
firewall-cmd --zone=public --list-services		
firewall-cmd --list-all			# 关注services: 表示放行的服务

firewall-cmd --zone=public --add-service=http		# 在特定区域内放行某个服务
firewall-cmd --zone=public --remove-service=http	# 在特定区域内禁止某个服务
```



# 4、端口相关

```bash
# 查看当前区域中放行的端口
firewall-cmd --list-ports
firewall-cmd --list-all			# 关注ports: 表示放行的端口

firewall-cmd --add-port=8080/tcp					# 放行当前区域中的端口	
firewall-cmd --zone=public --add-port=8080/tcp		# 在指定区域中放行端口
firewall-cmd --zone=public --remove-port=8080/tcp	# 在特定区域中禁止放行端口
```



# 5、网卡相关

```bash
# 查看当前区域中激活的网络接口
firewall-cmd --get-active-zones
firewall-cmd --list-all			# 关注interfaces: 表示启用的网卡

firewall-cmd --zone=public --add-interface=eth2		# 启用指定区域的网络接口
firewall-cmd --zone=public --remove-interface=eth2	# 移除指定区域的网络接口
```



# 6、防火墙自定义服务

**如何添加自定义服务(比如：sersync) 到firewalld中？**

首先，需要创建一个 XML 文件来定义您的自定义服务。通常，这些文件位于`/etc/firewalld/services/` 目录中。也可以参照该目录下的其他服务文件。

```bash
# 定义sersync服务
cat >  /usr/lib/firewalld/services/sersync.xml <<OEF
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>sersync</short>
  <description></description>
  <port protocol="tcp" port="873"/>
</service>
EOF

# 重载服务
firewall-cmd --reload

# 添加服务到区域
firewall-cmd --zone=public --add-service=sersync --permanent

# 验证服务添加结果
firewall-cmd --get-services | grep sersync
firewall-cmd --list-all
```



# 7、防火墙端口转发策略

```bash
# 启用IP转发
sysctl net.ipv4.ip_forward			# 检查 IP 转发状态
sysctl -w net.ipv4.ip_forward=1		# 临时启用 IP 转发
echo "net.ipv4.ip_forward = 1" | tee -a /etc/sysctl.conf	# 永久启用 IP 转发
sysctl -p

# 开启地址伪装
firewall-cmd --reload

# 添加端口转发规则
firewall-cmd --zone=public --add-rich-rule=' \
rule family="ipv4" \
source address="0.0.0.0/0" \
forward-port port="8888" \
protocol="tcp" \
to-port="80" \
to-addr="172.16.1.7" 
--permanent'
```



# 8、富规则语法

富规则用于创建复杂的防火墙规则。这些规则允许基于多种条件和操作对流量进行细粒度控制。

提供了更多的选项和逻辑组合，适用于需要更高安全性或特定网络策略的场景。

```bash
# 富规则相关命令
firewall-cmd --add-rich-rule='<RULE>' 		# 在指定区域添加一条富语言规则
firewall-cmd --remove-rich-rule='<RULE>' 	# 在指定的区删除一条富语言规则
firewall-cmd --query-rich-rule='<RULE>' 	# 找到规则返回0，找不到返回1
firewall-cmd --list-rich-rules 				# 列出指定区里的所有富语言规则

# '<RULE>'
rule family="ipv4"									# ipv4 或 ipv6
source address="address[/mask]" [invert="True"]		# 定义源地址,invert="True" 表示反选，匹配不在指定地址范围内的流量
service name="服务名称"						  
port port="端口号" protocol="协议名称"				   # 定要匹配的端口和协议
protocol value="协议"								   # 指定其他协议类型，适用于不通过服务或端口匹配的情况
forward-port port="要转发的端口号" protocol="tcp|udp" to-port="转发到的目标端口" toaddr="目标地址"		
accept | reject [type="reject type"] | drop			# 动作（accept接受，reject拒绝可响应指定类型ICMP消息，drop丢弃不响应ICMP消息）
```



# 案例1

允许10.0.0.1主机能够访问http服务 允许172.16.1.0/24能够访问22端口

```bash
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="10.0.0.1" service name="http" accept' --permanent
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="172.16.1.0/24" service name="ssh" accept' --permanent

firewall-cmd --reload
firewall-cmd --list-rich-rules
```



# 案例2

默认public区域对外开放所有人能通过ssh服务连接，但拒绝172.16.1.0/24网段通过ssh连接服务器

```bash
# 确保SSH服务已在public区域中被允许
firewall-cmd --zone=public --add-service=ssh --permanent

# 添加富规则以拒绝172.16.1.0/24网段访问 SSH
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="172.16.1.0/24" service name="ssh" reject' --permanent

firewall-cmd --reload
firewall-cmd --list-rich-rules
```



# 案例3

使用firewalld，允许所有人能访问http,https服务，但只有10.0.0.1主机可以访问ssh服务

```bash
# 添加HTTP和HTTPS服务到public区域
sudo firewall-cmd --zone=public --add-service=http --permanent
sudo firewall-cmd --zone=public --add-service=https --permanent

# 添加SSH服务到public区域
firewall-cmd --zone=public --add-service=ssh --permanent

# 添加富规则，允许10.0.0.1主机访问SSH服务，拒绝其他主机访问SSH服务
# 允许10.0.0.1主机访问SSH
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="10.0.0.1" service name="ssh" accept' --permanent

# 拒绝其他主机访问SSH
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" service name="ssh" reject' --permanent

firewall-cmd --reload
firewall-cmd --list-rich-rules
```


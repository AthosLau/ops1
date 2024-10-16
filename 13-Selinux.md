# Selinux

Slinux是针对用户、进程、目录预设的保护策略，是一套增强Linux系统安全的强制访问控制体系。

工作中，一般不启用

## 1、selinux工作模式

- 强制模式：enforcing
- 宽松模式：permissive
- 禁用模式：disabled



## 2、selinux修改配置

```bash
getenforce 			# 查看selinux工作模式

临时修改：
setenforce 1 		# 强制模式
setenforce 0 		# 宽松模式

永久设置：
sed -i "s/^SELINUX=.*/SELINUX=disabled/g" /etc/selinux/config
sed -i "s/^SELINUX=.*/SELINUX=disabled/g" /etc/sysconfig/selinux
```






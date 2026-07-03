# pfctl 速查表

## 通用 pfctl 命令

| 命令 | 说明 |
| --- | --- |
| `pfctl -d` | 禁用包过滤 |
| `pfctl -e` | 启用包过滤 |
| `pfctl -q` | 静默运行 |
| `pfctl -v` | 输出比默认更详细 |
| `pfctl -v -v` | 输出更详细 |

## 加载 PF 规则

| 命令 | 说明 |
| --- | --- |
| `pfctl -f /etc/pf.conf` | 加载 **/etc/pf.conf** |
| `pfctl -n -f /etc/pf.conf` | 测试规则（解析 **/etc/pf.conf** 但不加载） |
| `pfctl -R -f /etc/pf.conf` | 仅加载 FILTER 规则 |
| `pfctl -N -f /etc/pf.conf` | 仅加载 NAT 规则 |
| `pfctl -O -f /etc/pf.conf` | 仅加载 OPTION 规则 |

## 清除 PF 规则与计数器

清除规则不会影响任何已建立的有状态连接。

| 命令 | 说明 |
| --- | --- |
| `pfctl -F all` | 清除全部 |
| `pfctl -F rules` | 仅清除规则 |
| `pfctl -F queue` | 仅清除队列 |
| `pfctl -F nat` | 仅清除 NAT |
| `pfctl -F info` | 清除所有不属于任何规则的统计信息 |
| `pfctl -z` | 清零所有计数器 |

## 输出 PF 信息

| 命令 | 说明 |
| --- | --- |
| `pfctl -s rules` | 显示过滤信息 |
| `pfctl -sr` | 显示过滤信息（另一种写法） |
| `pfctl -v -s rules` | 显示过滤信息及命中次数 |
| `pfctl -vvsr` | 显示过滤信息及规则编号 |
| `pfctl -v -s nat` | 显示 NAT 信息及命中次数 |
| `pfctl -s nat -i xl1` | 显示 `xl1` 接口的 NAT 信息 |
| `pfctl -s queue` | 显示队列信息 |
| `pfctl -s label` | 显示 LABEL 信息 |
| `pfctl -s state` | 显示状态表内容 |
| `pfctl -s info` | 显示状态表与包规范化的统计信息 |
| `pfctl -s all` | 显示所有信息 |

## 维护 PF 表

| 命令 | 说明 |
| --- | --- |
| `pfctl -t addvhosts -T show` | 显示 `addvhosts` 表 |
| `pfctl -vvsTables` | 查看所有表的全局信息 |
| `pfctl -t addvhosts -T add 192.168.0.5` | 向 `addvhosts` 表添加条目 |
| `pfctl -t addvhosts -T add 192.168.0.0/16` | 向 `addvhosts` 表添加网络 |
| `pfctl -t addvhosts -T delete 192.168.0.0/16` | 从 `addvhosts` 表删除网络 |
| `pfctl -t addvhosts -T flush` | 移除 `addvhosts` 表中的所有条目 |
| `pfctl -t addvhosts -T kill` | 彻底删除 `addvhosts` 表 |
| `pfctl -t addvhosts -T replace -f /etc/addvhosts` | 动态重载 `addvhosts` 表 |
| `pfctl -t addvhosts -T test 192.168.0.140` | 在 `addvhosts` 表中查找 IP 地址 **192.168.0.140** |
| `pfctl -T load -f /etc/pf.conf` | 加载新的表定义 |
| `pfctl -t addvhosts -T show -vi` | 输出 `addvhosts` 表中每个 IP 地址的统计信息 |
| `pfctl -t addvhosts -T zero` | 重置 `addvhosts` 表的所有计数器 |

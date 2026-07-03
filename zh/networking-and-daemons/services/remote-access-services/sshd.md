# sshd

## 概述

`sshd` 是 OpenSSH 守护进程，接受传入的 SSH 连接，提供加密的远程 shell 访问与安全的文件传输。在 OpenBSD 上，`sshd` 属于基本系统，安装后默认启用。典型用途包括安全管理、通过 `sftp(1)` 与 `scp(1)` 传输文件、远程命令执行以及安全隧道。

服务管理使用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)。配置则编辑 [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)。守护进程受益于 [pledge(2)](https://man.openbsdhandbook.com/pledge.2/)、[unveil(2)](https://man.openbsdhandbook.com/unveil.2/) 以及 [pf(4)](https://man.openbsdhandbook.com/pf.4/) 的网络控制。

## 服务管理

使用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/) 在开机时启用并立即启动：

```sh
# rcctl enable sshd
# rcctl start sshd
# rcctl check sshd
```

主机密钥在安装时生成。若缺失，用 [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/) 创建：

```sh
# ssh-keygen -A
```

## 配置基础

守护进程读取 **/etc/ssh/sshd_config**。每行仅允许一条指令，以 `#` 开头的行视为注释。用 [sshd(8)](https://man.openbsdhandbook.com/sshd.8/) 验证更改，并通过 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/) 重启。

```sh
# sshd -t
# rcctl restart sshd
```

精简的基线配置：

```
Port 22
AddressFamily any
ListenAddress 0.0.0.0
ListenAddress ::

PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
PubkeyAuthentication yes

# 访问控制
AllowUsers admin

# 会话保活
ClientAliveInterval 300
ClientAliveCountMax 2

# 子系统（默认启用）
Subsystem sftp /usr/libexec/sftp-server
```

此配置禁用 root 登录与密码认证，启用公钥认证，将访问限制到指定账户，并设置保守的保活参数。按需调整 `ListenAddress`。

## 客户端配置

OpenBSD 基本系统包含客户端程序 [ssh(1)](https://man.openbsdhandbook.com/ssh.1/)、[sftp(1)](https://man.openbsdhandbook.com/sftp.1/) 与 [scp(1)](https://man.openbsdhandbook.com/scp.1/)。本节介绍密钥创建、密钥安装与常见客户端操作。客户端配置从 [ssh\_config(5)](https://man.openbsdhandbook.com/ssh_config.5/) 读取。

### 生成 SSH 密钥对

创建带标识注释的 Ed25519 密钥对。使用 [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/)。用 passphrase 保护私钥。

```sh
$ ssh-keygen -t ed25519 -C "[admin@example.com](mailto:admin@example.com)" -f \~/.ssh/id\_ed25519
```

对于硬件安全密钥（FIDO/U2F），生成 `ed25519-sk` 密钥：

```sh
$ ssh-keygen -t ed25519-sk -C "[admin@example.com](mailto:admin@example.com)" -f \~/.ssh/id\_ed25519\_sk
```

确保 `.ssh` 目录与文件具有适当权限：

```sh
$ chmod 700 \~/.ssh
$ chmod 600 \~/.ssh/id\_\* \~/.ssh/id\_\*.pub
```

### 将密钥载入 ssh-agent（可选）

agent 可在内存中保存已解密的密钥。使用 [ssh-agent(1)](https://man.openbsdhandbook.com/ssh-agent.1/) 与 [ssh-add(1)](https://man.openbsdhandbook.com/ssh-add.1/) 启动 agent 并添加密钥。

```sh
$ eval "\$(ssh-agent -s)"
$ ssh-add \~/.ssh/id\_ed25519
```

### 在服务器上安装公钥

若客户端系统上有 `ssh-copy-id` 辅助工具，它可将公钥追加到远程账户的 `~/.ssh/authorized_keys`。该工具不属于 OpenBSD 基本系统，安装后用法如下：

```sh
$ ssh-copy-id [admin@host.example.com](mailto:admin@host.example.com)
```

`ssh-copy-id` 不可用时，可通过 SSH 用标准工具追加公钥：

```sh
$ ssh [admin@host.example.com](mailto:admin@host.example.com) 'mkdir -p \~/.ssh && chmod 700 \~/.ssh && touch \~/.ssh/authorized\_keys && chmod 600 \~/.ssh/authorized\_keys'
$ cat \~/.ssh/id\_ed25519.pub | ssh [admin@host.example.com](mailto:admin@host.example.com) 'cat >> \~/.ssh/authorized\_keys'
```

首次连接时验证并记录服务器主机密钥，连接一次并确认指纹即可。若需非交互式获取，[ssh-keyscan(1)](https://man.openbsdhandbook.com/ssh-keyscan.1/) 可填充 `~/.ssh/known_hosts`：

```sh
$ ssh-keyscan -t ed25519 host.example.com >> \~/.ssh/known\_hosts
```

### 按主机定制客户端配置

在 `~/.ssh/config` 中创建主机段落，为目标定义默认值。参见 [ssh\_config(5)](https://man.openbsdhandbook.com/ssh_config.5/)。

```
Host prod
HostName host.example.com
User admin
Port 22
IdentityFile \~/.ssh/id\_ed25519
IdentitiesOnly yes
ForwardAgent no
ForwardX11 no
```

保存文件后，用简短别名连接：

```sh
$ ssh prod
```

### 交互式登录与远程命令执行

使用 [ssh(1)](https://man.openbsdhandbook.com/ssh.1/) 建立交互式会话或执行单条远程命令。

```sh
$ ssh [admin@host.example.com](mailto:admin@host.example.com)
$ ssh [admin@host.example.com](mailto:admin@host.example.com) 'uname -a'
```

### 安全复制（scp）

传输文件或目录。递归复制使用 `-r`。

```sh
$ scp ./report.txt [admin@host.example.com](mailto:admin@host.example.com):/home/admin/
$ scp -r ./site/ [admin@host.example.com](mailto:admin@host.example.com):/var/www/
$ scp [admin@host.example.com](mailto:admin@host.example.com):/var/log/daemon /tmp/daemon.log
```

### SFTP 文件传输

使用 [sftp(1)](https://man.openbsdhandbook.com/sftp.1/) 打开交互式 SFTP 会话或执行一次性传输。

```sh
$ sftp [admin@host.example.com](mailto:admin@host.example.com)
sftp> pwd
sftp> lpwd
sftp> put ./archive.tar.gz /home/admin/
sftp> get /etc/daily /tmp/daily
sftp> mkdir /home/admin/uploads
sftp> ls -l /home/admin
sftp> exit
```

非交互式 SFTP 可直接传输文件：

```sh
$ sftp [admin@host.example.com](mailto:admin@host.example.com):/etc/daily /tmp/daily
$ sftp /var/log/messages [admin@host.example.com](mailto:admin@host.example.com):/home/admin/
```

### 已知主机与首次使用验证

首次连接时，[ssh(1)](https://man.openbsdhandbook.com/ssh.1/) 会提示信任服务器主机密钥并将其追加到 `~/.ssh/known_hosts`。预验证时，将呈现的指纹与服务器上生成的指纹比对：

```sh
# 在服务器上（以 root 身份运行以读取主机密钥）：

# ssh-keygen -l -f /etc/ssh/ssh\_host\_ed25519\_key.pub
```

### 可靠性与性能的常用选项

在 `~/.ssh/config` 中控制连接复用与超时：

```
Host \*
ServerAliveInterval 60
ServerAliveCountMax 2
ControlMaster auto
ControlPersist 5m
ControlPath \~/.ssh/cm-%r@%h:%p
```

`ControlMaster` 支持在单条 TCP 连接上多路复用；同一主机的后续会话启动更快，且无需重复认证。

## 加固

### 结合 ssh-audit 整合算法策略

使用 `ssh-audit` 清点并评估提供的算法。参见配套章节：[/ssh-audit](ssh-audit.md)。根据审计结果，在 [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/) 中限制密钥交换（KEX）、主机密钥与 MAC 算法。

保守且兼容性广泛的策略：

```
# 密钥交换
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,\
diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,\
diffie-hellman-group-exchange-sha256

# MAC（encrypt-then-MAC 变体）
MACs umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com,\
hmac-sha2-512-etm@openssh.com

# 主机密钥算法（服务器认证）
HostKeyAlgorithms ssh-ed25519,rsa-sha2-256,rsa-sha2-512
```

缩减 `HostKeyAlgorithms` 时，确保 `HostKey` 文件指令仅引用保留的类型（例如，禁用 ECDSA 时省略 `ssh_host_ecdsa_key`）。验证并重新加载：

```sh
# sshd -t
# rcctl restart sshd
```

整理 Diffie–Hellman 组交换参数时，将 **/etc/ssh/moduli** 过滤为至少 3071 位的条目；参见 [moduli(5)](https://man.openbsdhandbook.com/moduli.5/)。

```sh
# awk '$5 >= 3071' /etc/ssh/moduli > /etc/ssh/moduli.safe
# mv -f /etc/ssh/moduli.safe /etc/ssh/moduli
# rcctl restart sshd
```

每次更改后再次运行 `ssh-audit`，检查是否有意外的回退。

### 访问控制

限制暴露面，缩小可认证的用户范围。

```
# 绑定到显式地址与非标准端口（如需要）
Port 22
ListenAddress 192.0.2.10
# ListenAddress 2001:db8::10

# 仅允许特定用户或组
AllowUsers admin opsuser
# 或：
# AllowGroups wheel ops
```

用 [pf(4)](https://man.openbsdhandbook.com/pf.4/) 将入站限制为可信来源。参见 PF 章节及 [/pf/cheat\_sheet/](../../pf/cheat_sheet.md) 中的 pfctl 速查表。

```pf
# PF 片段示例（视上下文而定）
block in on egress proto tcp to port ssh
pass  in on egress proto tcp from 198.51.100.0/24 to port ssh
```

用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/) 重新加载：

```sh
# pfctl -f /etc/pf.conf
```

### 认证与多因子选项

OpenSSH 支持多种多因子认证方式，无需外部 PAM 模块。

1. **公钥 + 密码（两个独立因子）。** 使用 [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/) 的 `AuthenticationMethods` 同时要求私钥与密码。

   ```
   PubkeyAuthentication yes
   PasswordAuthentication yes
   AuthenticationMethods publickey,password
   ```

   此设置在两个因子均成功时才允许访问。
2. **安全密钥（FIDO/U2F）。** 安全密钥支撑的密钥（如 `sk-ssh-ed25519@openssh.com`）提供持有加触碰验证。在客户端用 [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/) 生成：

   ```sh
   $ ssh-keygen -t ed25519-sk -f ~/.ssh/id_ed25519_sk
   ```

   确保服务器策略允许相应的公钥算法（若限制了 `PubkeyAcceptedAlgorithms`，不要排除 `sk-ssh-ed25519@openssh.com`），并保持公钥认证启用。
3. **通过 BSD Authentication 的键盘交互第二因子。** OpenBSD 使用 bsd\_auth(3) 而非 PAM。附加的 BSD Auth 后端可通过 `keyboard-interactive` 提供一次性验证码（如 TOTP）。安装并配置合适的后端后，用以下配置要求第二因子：

   ```
   KbdInteractiveAuthentication yes
   AuthenticationMethods publickey,keyboard-interactive
   ```

   后端的选择与配置因后端而异，不在本章范围之内。

附加加固措施：

```
# 减少暴力破解面
MaxAuthTries 3
LoginGraceTime 30
MaxStartups 10:30:100

# 禁用不需要的功能
PermitTunnel no
AllowAgentForwarding no
AllowTcpForwarding no
X11Forwarding no
GatewayPorts no
PrintMotd no
UseDNS no
```

### 子系统隔离与 chroot 的 SFTP

用 chroot 与内部 SFTP 子系统约束仅限 SFTP 的账户：

```
Match User sftpjailed
    ChrootDirectory /home/sftpjailed
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

chroot 目录必须由 `root` 拥有，且不可被目标账户写入；详情参见 [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)。

## 监控与故障排查

检查服务状态与监听套接字：

```sh
# rcctl check sshd
# sockstat -4 -6 | grep sshd
```

查看认证日志（facility 为 `auth`）：

```sh
# tail -f /var/log/authlog
```

常见问题包括语法错误（`sshd -t`）、`~/.ssh` 或 `authorized_keys` 权限不正确、公钥缺失，或防火墙策略阻断了端口。

修复用户密钥权限：

```sh
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/authorized_keys
$ chown "$USER":"$USER" ~/.ssh ~/.ssh/authorized_keys
```

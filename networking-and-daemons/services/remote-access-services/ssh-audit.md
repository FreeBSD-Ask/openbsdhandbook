# 审计 OpenSSH

## 概述

1. 使用 [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/) 安装 `ssh-audit`。
2. 对目标运行 `ssh-audit`，清点支持的算法。
3. 在 [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/) 中限制密钥交换、主机密钥与 MAC 算法。
4. 用 [sshd(8)](https://man.openbsdhandbook.com/sshd.8/) 验证配置，并通过 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/) 重启。
5. 再次运行 `ssh-audit` 确认策略生效。
6. 可选：按 [moduli(5)](https://man.openbsdhandbook.com/moduli.5/) 移除较小的 Diffie–Hellman 模数。
7. 可选：用 [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/) 轮换主机密钥。

## 概览

本章介绍如何使用 `ssh-audit` 工具评估并加固 OpenBSD 上的 OpenSSH 服务器。工作流程为：安装并运行 `ssh-audit`，在 [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/) 中限制算法，用 [sshd(8)](https://man.openbsdhandbook.com/sshd.8/) 验证守护进程，通过 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/) 重启，并可选地按 [moduli(5)](https://man.openbsdhandbook.com/moduli.5/) 整理 Diffie–Hellman 模数、用 [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/) 轮换主机密钥。

**密钥交换算法（KEX）** 用于建立共享密钥。**主机密钥算法** 用于认证服务器身份。**消息认证码（MAC）** 用于验证消息完整性。限制这些算法集合可缩小攻击面，同时维持所需的客户端兼容性。

## 安装并运行 ssh-audit

使用 [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/) 安装软件包，并执行本地审计。

```sh
# pkg_add ssh-audit
$ ssh-audit localhost
```

审计远程主机时，指定目标与可选端口。IPv6 地址字面量需用方括号包裹。

```sh
$ ssh-audit example.org
$ ssh-audit -p 2222 example.org
$ ssh-audit -p 22 '[2001:db8::1]'
```

该工具会报告服务器横幅及提供的 KEX、主机密钥、加密算法与 MAC，并标出弱项或已弃用的选项。

## 在 sshd\_config 中加固算法策略

编辑 **/etc/ssh/sshd_config**，仅允许一组精简的现代算法。下面的命令会追加适用于广泛客户端的保守设置，请根据部署的兼容性需求审查并调整。指令语义参见 [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)。

**密钥交换。** 优先使用 Curve25519 与强有限域 Diffie–Hellman 组。若服务器与客户端均支持，可加入混合算法 `sntrup761x25519-sha512@openssh.com`。

```sh
# sh -c 'printf "\n# Restrict key exchange\nKexAlgorithms \
curve25519-sha256,curve25519-sha256@libssh.org,\
diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,\
diffie-hellman-group-exchange-sha256\n" >> /etc/ssh/sshd_config'
```

**MAC。** 优先使用 encrypt-then-MAC 变体，避免 SHA-1。

```sh
# sh -c 'printf "# Restrict MACs\nMACs \
umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com,\
hmac-sha2-512-etm@openssh.com\n" >> /etc/ssh/sshd_config'
```

**主机密钥。** 优先使用 Ed25519 与采用 SHA-2 签名的 RSA。

```sh
# sh -c 'printf "# Restrict HostKey algorithms\nHostKeyAlgorithms \
ssh-ed25519,rsa-sha2-256,rsa-sha2-512\n" >> /etc/ssh/sshd_config'
```

缩减 `HostKeyAlgorithms` 时，确保 **/etc/ssh/sshd_config** 中的 `HostKey` 文件路径仅引用保留的密钥类型。例如，禁用 ECDSA 时应省略 `ssh_host_ecdsa_key`。参见 [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)。

## 验证并应用配置

测试守护进程配置并重启服务。用 [sshd(8)](https://man.openbsdhandbook.com/sshd.8/) 配合 `-t` 验证设置，用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/) 重启。

```sh
# sshd -t
# rcctl restart sshd
```

再次运行审计，确认提供了预期的算法集合。

```sh
$ ssh-audit localhost
```

## 可选：移除较小的 Diffie–Hellman 模数

组交换参数存放在 **/etc/ssh/moduli**。过滤掉小于 3071 位的条目，以达到至少约 128 位强度。字段详情参见 [moduli(5)](https://man.openbsdhandbook.com/moduli.5/)。

```sh
# awk '$5 >= 3071' /etc/ssh/moduli > /etc/ssh/moduli.safe
# mv -f /etc/ssh/moduli.safe /etc/ssh/moduli
# rcctl restart sshd
```

## 可选：轮换主机密钥

弃用算法或加强参数时，应轮换主机密钥。备份现有密钥，移除不需要的类型，并生成 Ed25519 与 RSA（3072 位）密钥。使用 [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/)。

```sh
# cd /etc/ssh
# install -d -m 0700 ./backup-keys && mv ssh_host_* ./backup-keys/
# ssh-keygen -t ed25519 -N '' -f /etc/ssh/ssh_host_ed25519_key
# ssh-keygen -t rsa -b 3072 -o -a 100 -N '' -f /etc/ssh/ssh_host_rsa_key
```

确保 `HostKey` 指令引用新文件：

```
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
```

验证并重启：

```sh
# sshd -t
# rcctl restart sshd
```

## 客户端兼容性注意事项

收紧策略可能使不支持 Curve25519 或 RSA SHA-2 签名的旧客户端无法连接。若无法避免旧客户端访问，可在 `sshd_config` 中使用条件 match 块限定例外，或另设一台采用宽松策略的堡垒机。参见 [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/) 与 [ssh(1)](https://man.openbsdhandbook.com/ssh.1/)。

## 验证

使用 `ssh-audit` 确认已移除 SHA-1 MAC、小 DH 组或已禁用的主机密钥类型等弱项。部署后通过系统的 [syslog(3)](https://man.openbsdhandbook.com/syslog.3/) 设施监控认证与协商日志。

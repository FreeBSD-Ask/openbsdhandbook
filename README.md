# OpenBSD 手册

你现在看到的是 OpenBSD 手册中文翻译项目。原作者项目地址：<https://www.openbsdhandbook.com/>，作者佚名。本项目采用 2 条款 BSD 协议发布。部分内容可能已经过时，欢迎提交 PR 更新。

---

> **“Linux 世界如此行事，是因为他们讨厌微软；我们如此行事，是因为我们热爱 Unix。”**
>
> —— Theo de Raadt（OpenBSD 创始人）

OpenBSD 手册是面向系统管理员的实用参考。本书注重讲解清晰、假设最少，并提供可直接复制粘贴的示例。目标是在服务器、嵌入式设备与工作站上稳定地安装、加固运行 OpenBSD 7.8。

## OpenBSD 的不同之处

OpenBSD 是一款完整、自洽的类 Unix 操作系统，作为整体来维护。项目把简洁、正确与安全放在首位。

- **自洽的基本系统。** 内核、用户态、工具链与核心守护进程一同开发。诸如 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/) 之类的管理接口在各服务之间保持一致。
- **安全工程。** 主动代码审计、权限分离与漏洞缓解是标准做法。应用沙箱 [pledge(2)](https://man.openbsdhandbook.com/pledge.2/) 与 [unveil(2)](https://man.openbsdhandbook.com/unveil.2/) 内置于基本系统。
- **实用的默认配置。** PF 防火墙通过 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/) 配置。精简的 Web 服务器（[httpd(8)](https://man.openbsdhandbook.com/httpd.8/)）使用 [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)。其他基本组件还包括 [relayd(8)](https://man.openbsdhandbook.com/relayd.8/)、[smtpd(8)](https://man.openbsdhandbook.com/smtpd.8/)、[bgpd(8)](https://man.openbsdhandbook.com/bgpd.8/) 与 OpenSSH。
- **文档优先。** 手册页权威且完整，从 [man(1)](https://man.openbsdhandbook.com/man.1/) 起步。包管理从 [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/) 开始。
- **可预期的发布。** 大约每六个月发布新版本。二进制补丁通过 [syspatch(8)](https://man.openbsdhandbook.com/syspatch.8/) 应用。版本之间的升级可用 [sysupgrade(8)](https://man.openbsdhandbook.com/sysupgrade.8/) 自动完成。

## 优势与权衡

OpenBSD 偏好安全、可理解的设计，而非功能堆砌。基本系统小巧，刻意保持保守。硬件支持广泛但并不全面，某些专用或专有设备可能缺少驱动。Ports 与软件包提供了大量第三方软件，重点在于可维护性。性能固然重要，但不会以牺牲安全或清晰为代价去追求。

## 如何使用本手册

本手册在工作流、背景说明与注意事项方面补充手册页。各章节会引用相关手册页，并附上经过测试的示例。约定如下：

- Shell 提示符：`$` 表示普通用户；`#` 表示超级用户（通常通过 `doas` 获得，参见 [doas(1)](https://man.openbsdhandbook.com/doas.1/)）。
- 命令输出与配置片段按实际输入或生成的样子呈现。
- 首次出现命令、守护进程、配置文件或系统调用时，会链接到对应的手册页。

如需整体了解与范围说明，请参阅 Introduction。

继续阅读 -> [Introduction](install-and-configure/introduction.md)

## 从这里开始

常见任务与入口：

- **安装与初始化。** 磁盘布局、网络、用户、首次登录。
  -> [/install/](install-and-configure/installation.md)
- **加固基本系统。** 权限模型、`doas`、文件权限、密码学、用 syspatch 打补丁、用 sysupgrade 升级版本。
  -> [/security/](system-administration/security.md)
- **网络。** 网络接口、自动配置、网桥、VLAN、无线、路由。
  -> [/networking/](install-and-configure/networking.md)
- **PF 防火墙。** 策略设计、NAT、重定向、表、锚点、用 `pfctl` 测试（参见 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)）。
  -> [/pf/](networking-and-daemons/pf/README.md)
- **Web 与代理。** httpd、TLS、OCSP、反向代理、用 [relayd(8)](https://man.openbsdhandbook.com/relayd.8/) 做健康检查。
  -> [/httpd/](networking-and-daemons/services/web-services/httpd.md)
  · [/relayd/](networking-and-daemons/services/web-services/relayd.md)
- **邮件传输。** OpenSMTPD 基础（参见 [smtpd(8)](https://man.openbsdhandbook.com/smtpd.8/)）、本地投递、中继。
  -> [/mail/](networking-and-daemons/services/mail-services/README.md)
- **存储。** 磁盘、分区、softraid、加密、文件系统、备份。
  -> [/storage/](system-administration/storage.md)
- **虚拟化。** 用 [vmd(8)](https://man.openbsdhandbook.com/vmd.8/) 与 [vmm(4)](https://man.openbsdhandbook.com/vmm.4/) 设备实现原生虚拟化。
  -> [/vmm/](system-administration/virtualization.md)
- **软件包与 ports。** 搜索、安装、版本锁定、用 [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/) 从 ports 构建。
  -> [/packages/](install-and-configure/ports.md)
- **升级与维护。** 版本规划、备份、升级后检查。
  -> [/upgrading/](system-administration/upgrade.md)
- **工作站主题。** X Window System、图形、音频、日常工具。
  -> [/x11/](install-and-configure/x11.md)

如需更深入的参考，请查阅手册页：
Manuals -> <https://man.openbsdhandbook.com/>

## 范围与读者

本手册面向偏好显式、可复现流程与保守默认值的技术型管理员。内容避免不必要的抽象，点明常见陷阱，偏好简单、可审计的方案，而非复杂的叠加方案。

在注重可预期性与安全性的环境中，OpenBSD 提供了坚实的基础，权衡清晰。本手册提供将其用对、用好所需的操作流程。

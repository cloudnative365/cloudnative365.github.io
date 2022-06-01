---
title: windows_exporter
keywords: keynotes, senior, metrics, windows_exporter
permalink: keynotes_L3_senior_5_metrics_10_windows_exporter.html
sidebar: keynotes_L3_senior_sidebar
typora-copy-images-to: ./pics/3_exporter
typora-root-url: ../../../../../cloudnative365.github.io
---

## 1. 概述

微软系的产品其实我一直没怎么研究，主要是我个人非常喜欢开源产品，对于收费的产品基本没什么研究。这里说的不到位的地方希望大家谅解。

Windows的exporter很有意思，因为他不仅可以监控Windows系统本身，连上面一些常用的服务也可以监控起来，比如ad，sql-server，iis，hyperv，exchange, .NET信息等等

## 2. windows_exporter

和node_exporter类似，windows也是使用了collector的方式，默认打开一些选项，如果我们有额外要求再打开别的选项

### 2.1. Collectors

摘自[官网](https://github.com/prometheus-community/windows_exporter)

| Name                                                         | Description                                                  | Enabled by default |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------ |
| [ad](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.ad.md) | Active Directory Domain Services                             |                    |
| [adfs](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.adfs.md) | Active Directory Federation Services                         |                    |
| [cpu](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.cpu.md) | CPU usage                                                    | ✓                  |
| [cs](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.cs.md) | "Computer System" metrics (system properties, num cpus/total memory) | ✓                  |
| [container](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.container.md) | Container metrics                                            |                    |
| [dhcp](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.dhcp.md) | DHCP Server                                                  |                    |
| [dns](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.dns.md) | DNS Server                                                   |                    |
| [exchange](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.exchange.md) | Exchange metrics                                             |                    |
| [fsrmquota](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.fsrmquota.md) | Microsoft File Server Resource Manager (FSRM) Quotas collector |                    |
| [hyperv](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.hyperv.md) | Hyper-V hosts                                                |                    |
| [iis](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.iis.md) | IIS sites and applications                                   |                    |
| [logical_disk](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.logical_disk.md) | Logical disks, disk I/O                                      | ✓                  |
| [logon](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.logon.md) | User logon sessions                                          |                    |
| [memory](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.memory.md) | Memory usage metrics                                         |                    |
| [msmq](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.msmq.md) | MSMQ queues                                                  |                    |
| [mssql](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.mssql.md) | [SQL Server Performance Objects](https://docs.microsoft.com/en-us/sql/relational-databases/performance-monitor/use-sql-server-objects#SQLServerPOs) metrics |                    |
| [netframework_clrexceptions](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.netframework_clrexceptions.md) | .NET Framework CLR Exceptions                                |                    |
| [netframework_clrinterop](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.netframework_clrinterop.md) | .NET Framework Interop Metrics                               |                    |
| [netframework_clrjit](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.netframework_clrjit.md) | .NET Framework JIT metrics                                   |                    |
| [netframework_clrloading](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.netframework_clrloading.md) | .NET Framework CLR Loading metrics                           |                    |
| [netframework_clrlocksandthreads](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.netframework_clrlocksandthreads.md) | .NET Framework locks and metrics threads                     |                    |
| [netframework_clrmemory](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.netframework_clrmemory.md) | .NET Framework Memory metrics                                |                    |
| [netframework_clrremoting](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.netframework_clrremoting.md) | .NET Framework Remoting metrics                              |                    |
| [netframework_clrsecurity](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.netframework_clrsecurity.md) | .NET Framework Security Check metrics                        |                    |
| [net](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.net.md) | Network interface I/O                                        | ✓                  |
| [os](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.os.md) | OS metrics (memory, processes, users)                        | ✓                  |
| [process](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.process.md) | Per-process metrics                                          |                    |
| [remote_fx](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.remote_fx.md) | RemoteFX protocol (RDP) metrics                              |                    |
| [service](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.service.md) | Service state metrics                                        | ✓                  |
| [system](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.system.md) | System calls                                                 | ✓                  |
| [tcp](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.tcp.md) | TCP connections                                              |                    |
| [thermalzone](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.thermalzone.md) | Thermal information                                          |                    |
| [terminal_services](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.terminal_services.md) | Terminal services (RDS)                                      |                    |
| [textfile](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.textfile.md) | Read prometheus metrics from a text file                     | ✓                  |
| [vmware](https://github.com/prometheus-community/windows_exporter/blob/master/docs/collector.vmware.md) | Performance counters installed by the Vmware Guest agent     |                    |

### 2.2. 启动选项

如果想要启动某个选项，就使用`--collectors.enabled`，比如

``` bash
.\windows_exporter.exe --collectors.enabled "service" --collector.service.services-where "Name='windows_exporter'"
```

启用二级选项，比如只监控firefox

``` bash
.\windows_exporter.exe --collectors.enabled "process" --collector.process.whitelist="firefox.+"
```

以进程形式启动

``` bash
msiexec /i C:\Users\Administrator\Downloads\windows_exporter.msi ENABLED_COLLECTORS="ad,iis,logon,memory,process,tcp,thermalzone" TEXTFILE_DIR="C:\custom_metrics\"
```


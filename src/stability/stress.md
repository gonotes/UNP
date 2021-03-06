## 系统加压

---

### 简介

#### UnitedStack知识库相关文章

 - [给你的系统加点料—Stress](https://confluence.ustack.com/pages/viewpage.action?pageId=16106956)

#### 使用场景

 我们在进行各种测试时，可以给系统加压，测试系统在高负载情况下各个服务的运行状态。
为什么会进行 Stress 测试？我们在运维了大量客户的生产环境后，发现当 Neutron 各个
服务在系统负载较低时，运行良好，API 较快，系统调用成功率高。但是当系统负载升高，
会产生很多意外的情况，比如 API 反应异常缓慢，由于系统升级而重启服务，时间变长，
系统调用生效慢等一系列问题。因此，我们认为在系统高负载的情况下进行各类测试是
十分有必要的。

 下面给出Stress的使用方法和截图：

 ```
 [root@server-233 ~(keystone_admin)]# stress
 `stress' imposes certain types of compute stress on your system
 Usage: stress [OPTION [ARG]] ...
  -?, --help         show this help statement
      --version      show version statement
  -v, --verbose      be verbose
  -q, --quiet        be quiet
  -n, --dry-run      show what would have been done
  -t, --timeout N    timeout after N seconds
      --backoff N    wait factor of N microseconds before work starts
  -c, --cpu N        spawn N workers spinning on sqrt()
  -i, --io N         spawn N workers spinning on sync()
  -m, --vm N         spawn N workers spinning on malloc()/free()
      --vm-bytes B   malloc B bytes per vm worker (default is 256MB)
      --vm-stride B  touch a byte every B bytes (default is 4096)
      --vm-hang N    sleep N secs before free (default is none, 0 is inf)
      --vm-keep      redirty memory instead of freeing and reallocating
  -d, --hdd N        spawn N workers spinning on write()/unlink()
      --hdd-bytes B  write B bytes per hdd worker (default is 1GB)
      --hdd-noclean  do not unlink files created by hdd workers
  
 stress --cpu 8 --io 4 --vm 50 --vm-bytes 1280M --timeout 10d
 ```

 ![stress][1]


#### 系统在不同负载下测试Neutron L3 服务所需重启时间

##### 测试说明

   本次测试通过观察重启 Neutron L3 Agent 时 Neutron 日志输出`L3 agent started`为准，确定重启所需时间。
   Neutron L3 Agent 主要负责虚拟路由器生命周期的管理，其中也包括了绑定 FloatingIP 时产生的 Namespace
   和虚拟路由器开启公网网关产生的 snat Namespace。
   本次测试使用到的 Neutron L3 Agent 详细信息如下：

```
[root@server-233 ~(keystone_admin)]# neutron agent-show d66f00ba-e665-4152-93c3-53b6c4a06a35
+---------------------+-------------------------------------------------------------------------------+
| Field               | Value                                                                         |
+---------------------+-------------------------------------------------------------------------------+
| admin_state_up      | True                                                                          |
| agent_type          | L3 agent                                                                      |
| alive               | True                                                                          |
| binary              | neutron-vpn-agent                                                             |
| configurations      | {                                                                             |
|                     |      "router_id": "",                                                         |
|                     |      "agent_mode": "dvr_snat",                                                |
|                     |      "gateway_external_network_id": "",                                       |
|                     |      "handle_internal_only_routers": true,                                    |
|                     |      "use_namespaces": true,                                                  |
|                     |      "routers": 2,                                                            |
|                     |      "interfaces": 3,                                                         |
|                     |      "floating_ips": 2,                                                       |
|                     |      "interface_driver": "neutron.agent.linux.interface.OVSInterfaceDriver",  |
|                     |      "log_agent_heartbeats": false,                                           |
|                     |      "external_network_bridge": "br-ex",                                      |
|                     |      "ex_gw_ports": 2                                                         |
|                     | }                                                                             |
| created_at          | 2016-04-15 10:02:50                                                           |
| description         |                                                                               |
| heartbeat_timestamp | 2016-05-23 08:16:15                                                           |
| host                | server-233                                                                    |
| id                  | d66f00ba-e665-4152-93c3-53b6c4a06a35                                          |
| started_at          | 2016-05-23 08:12:15                                                           |
| topic               | l3_agent                                                                      |
+---------------------+-------------------------------------------------------------------------------+
```

`由以上数据可知，本 L3 Agent 管理两个虚拟路由器（routers），三个网关设备，两个 FloatingIP。`


##### 测试步骤
1. 通过 stress 给系统达到不同的负载

2. 在各种负载下进行重启 Neutron L3 Agent 服务，观察重启时间的测试

3. 通过[uptime](http://linux.die.net/man/1/uptime)命令得到系统负载。进行本次测试使用到的系统负载如下：


  |--|系统负载|
  |:--:|:--:|
  |0 works|2.53|
  |5 works|31.26|
  |20 works|47.27|
  |30 works|53.99|
  |50 works|67.46|
  |80 works|100.96|

  `说明：Stress命令：stress --cpu 20 --io 4 --vm [workers] -vm-bytes 1280M --timeout 10d 。
本次使用到的测试机器是 32 核心，当系统负载为 31.26 时，每颗 CPU 接近满负载（31.26 / 32 * 100% = 97%）。`


##### 测试结果

 ![stress_restart_agent_time][2]


##### 测试结论

 Neutron L3 Agent重启所需时间随着系统负载的增大而变长。


 [1]: ../../images/stability/stress.png
 [2]: ../../images/stability/l3_agent_restart_time.png

结论：`dropping packet-in due to queue overflow` 不是 BFD 状态机本身的错误，而是 `ovs-vswitchd -> ovn-controller` 的 OpenFlow `packet-in` 队列满了。`queue-size=512` 已经接近 OVS 的上限，继续增大通常不可行，也不能解决持续速率过高的问题。

120 条会话、100 ms 周期的理论接收速率约为：

```text
120 / 0.1 = 1200 packet-in/s
```

512 个包的队列在控制器暂时不消费时，只能覆盖约：

```text
512 / 1200 = 0.43 s
```

`multi=5` 只影响故障检测时间，约为 500 ms，不会降低 BFD 报文速率。

OVN 的 BFD 流通常执行：

```text
handle_bfd_msg();
```

该动作把 BFD 报文送到 `ovn-controller` 处理，而不是完全在 OVS datapath 中终结。OVN northd 还可以通过 CoPP 的 `bfd` meter 对这类 packet-in 限速。参考：

- [OVN BFD schema](https://github.com/ovn-org/ovn/blob/main/ovn-nb.xml)
- [OVN BFD logical flow](https://github.com/ovn-org/ovn/blob/main/northd/northd.c)
- [OVS packet-in queue handling](https://github.com/openvswitch/ovs/blob/main/ofproto/connmgr.c)
- [OVS controller queue/rate-limit 配置](https://github.com/openvswitch/ovs/blob/main/vswitchd/vswitch.xml)

**优先处理方式**

1. 降低 BFD 报文速率

如果故障检测要求允许，首先把 `rx/tx` 从 100 ms 调大到 200 ms、500 ms 或 1 s，并同步调整 `multi`。例如：

```text
200 ms × 3 = 600 ms
500 ms × 3 = 1.5 s
```

实际协商间隔取双方参数中的较大值，因此 `bfdd` 与 OVN 两端都要检查。

2. 为 BFD packet-in 配置合理的 CoPP meter

1200 pps 不应直接作为限速值，建议先留 25% 至 50% 余量，例如：

```bash
ovn-nbctl meter-add bfd-meter drop 2000 pktps 500
ovn-nbctl copp-add copp-bfd bfd bfd-meter
ovn-nbctl lr-copp-add copp-bfd <logical-router>
```

如果已有 CoPP 配置，应修改现有配置，而不是重复创建。

注意：该 meter 是“主动丢弃超额 BFD 报文”。如果限速值低于正常 BFD 速率，BFD 可能被误判为 down。因此 meter 的速率必须高于实际总速率，并通过 meter 统计确认没有持续丢包。

3. 检查是否配置了 OVS 自身的 controller rate limit

```bash
ovs-vsctl --columns=name,other_config,status list Bridge
ovs-vsctl --columns=target,is_connected,status list Controller
```

重点检查：

```text
other_config:controller-queue-size
controller-rate-limit
controller-burst-limit
```

如果配置了很低的 `controller-rate-limit` 或 `controller-burst-limit`，可能是 OVS 自己先把 packet-in 排队并丢弃。对于 1200 pps 的场景，限速值必须至少覆盖 BFD 聚合速率；`controller-burst-limit` 也要能覆盖启动或重连时的瞬时突发。

`controller-queue-size` 的上限通常是 512；OVS 文档也提示增大该值会增加延迟。因此不要把“继续加队列”作为主要方案。

**建议的 debug 顺序**

1. 确认实际 BFD 报文速率

```bash
timeout 10 tcpdump -ni any 'udp dst port 3784'
```

分别在物理接口、相关 bridge 和可能的 patch/tunnel 接口观察，确认是否确实接近 1200 pps，或者存在重复报文、广播风暴、错误目的地址等问题。

2. 检查 OVN BFD 记录和状态

```bash
ovn-nbctl list BFD
ovn-sbctl list BFD
```

关注：

```text
logical_port
dst_ip
min_tx
min_rx
detect_mult
status
```

确认是否存在重复 BFD row、错误的 `logical_port`、错误 peer IP，或者大量会话处于 `down`/反复切换状态。

3. 确认 BFD logical flow 是否真的走 `handle_bfd_msg`

```bash
ovn-sbctl lflow-list <logical-router> | grep -E '3784|handle_bfd_msg|controller_meter'
```

应能看到类似：

```text
udp.dst == 3784
handle_bfd_msg()
```

同时确认 `controller_meter` 是否为空，以及是否绑定到了预期的 BFD meter。

4. 查看转换后的 OpenFlow 流和计数器

```bash
ovs-ofctl -O OpenFlow13 dump-flows br-int | \
  grep -E '3784|controller'
```

每隔几秒采样一次 `n_packets`，计算 BFD controller flow 的增长速率。还要检查是否有其他大量 `actions=CONTROLLER` 或 table-miss 流，因为日志中的 packet-in queue 是连接级队列，不一定全部来自 BFD。

5. 打开临时调试日志

```bash
ovs-appctl -t ovs-vswitchd vlog/set connmgr:dbg
ovs-appctl -t ovn-controller vlog/set pinctrl:dbg
```

OVN controller 中应能看到类似：

```text
rx BFD packets from ...
```

同时查看 `ovs-vswitchd` 是否出现：

- OpenFlow controller 重连
- TLS/Socket 写阻塞
- `packet-in` 持续 overflow
- `last_error` 或连接状态异常

完成后恢复日志级别：

```bash
ovs-appctl -t ovs-vswitchd vlog/set connmgr:info
ovs-appctl -t ovn-controller vlog/set pinctrl:info
```

6. 查看 CoPP/meter 统计

```bash
ovs-appctl -t ovn-controller meter-table-list
ovs-ofctl -O OpenFlow13 meter-stats br-int
```

如果 BFD meter 的 drop counter 持续增加，说明是 CoPP 限速；如果 meter 没有丢包但仍有 OVS overflow，则重点查 `ovn-controller` CPU、OpenFlow 连接写阻塞和其他 packet-in 来源。

**如何判断根因**

- tcpdump 约 1200 pps，`ovn-controller` CPU 高：控制器消费能力不足。
- tcpdump 速率正常，但存在大量其他 `CONTROLLER` 流：不是 BFD 独占问题，应修复额外 packet-in 源。
- `controller-rate-limit` 很低、burst 很小：调整 OVS controller 限速。
- BFD meter drop 持续增加：提高 meter 或降低 BFD 会话/频率。
- OpenFlow 连接反复重连或 socket 写阻塞：先解决网络、TLS、CPU 调度或控制器阻塞。
- 报文速率突然远高于 1200 pps：检查 bfdd 是否重复发送、会话是否重复配置，或同一个 BFD 报文是否被多个 logical flow 重复送入 controller。

实际操作上，推荐顺序是：先确认 packet-in 来源和速率，再把 BFD 周期从 100 ms 调大；如果必须保持 100 ms，则配置足够大的 BFD CoPP meter，并确保 `ovn-controller` 能稳定消费至少约 1200 packet-in/s。

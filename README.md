这组数据说明该 OpenFlow 流在采样期间产生了约 **538 packet-in/s**，不是 1200 packet-in/s：

```text
时间差 = 1395.073 - 1393.860 = 1.213 s
包数差 = 603299 - 602646 = 653
速率   = 653 / 1.213 ≈ 538 pps

字节差 = 39817734 - 39774636 = 43098
平均包长 ≈ 43098 / 653 ≈ 66 bytes
```

该流明确是：

```text
udp,tp_dst=3784 actions=controller(...)
```

因此它确实会把 BFD 报文送入 `ovn-controller`。在 538 pps 下，512 队列理论上只能覆盖约：

```text
512 / 538 ≈ 0.95 秒
```

只要 `ovn-controller` 或 OpenFlow socket 在某个时间窗口内暂停约 1 秒，就可能出现 overflow。

需要注意两点：

1. 这只是一个 OpenFlow flow 的计数，不一定包含全部 BFD 流。
2. `sleep 1` 两次命令的实际时间间隔是 1.213 秒，不能直接把 653 当作 653 pps。

先把所有 BFD controller 流一起统计，不要只筛选 `rw_dst`：

```bash
ovs-ofctl -O OpenFlow13 dump-flows br-int | \
  awk '/tp_dst=3784/ && /actions=controller/'

sleep 1

ovs-ofctl -O OpenFlow13 dump-flows br-int | \
  awk '/tp_dst=3784/ && /actions=controller/'
```

重点观察：

- 是否有多个 `nw_dst`、`metadata` 或 `cookie`
- 是否只有这一条流
- 每条流的 `n_packets` 增量
- 是否存在其他大量 `actions=controller` 的非 BFD 流

如果只有这一条流，那么当前实际输入约为 538 pps，可能意味着：

- 不是 120 条会话都在该 chassis 上接收；
- bfdd 与 OVN 协商后的实际周期不是 100 ms；
- 你观察到的是单方向或单个 logical datapath；
- 其他 BFD 流被其他 OpenFlow flow 匹配；
- 部分报文已经在更早的位置丢失。

建议同时在相关接口抓包核对：

```bash
timeout 10 tcpdump -ni any 'udp dst port 3784'
```

如果 10 秒抓到约 5380 个包，说明 OVS flow counter 与线上输入基本一致；如果抓包接近 12000 个而 flow 只有约 5380 个，则需要继续检查 flow 匹配、隧道/patch 路径和丢包位置。

另外，这条 flow 的输出中只有：

```text
actions=controller(userdata=...)
```

没有明显显示 meter。不能仅凭这一点断定 CoPP 未生效，但应确认：

```bash
ovn-sbctl lflow-list <logical-router> | \
  grep -E '3784|handle_bfd_msg|controller_meter'

ovs-appctl -t ovn-controller meter-table-list
ovs-ofctl -O OpenFlow13 meter-stats br-int
```

如果没有 BFD meter，可以针对实际聚合速率配置，例如：

```bash
ovn-nbctl meter-add bfd-meter drop 1000 pktps 250
ovn-nbctl copp-add copp-bfd bfd bfd-meter
ovn-nbctl lr-copp-add copp-bfd <logical-router>
```

这里的 `1000 pktps` 只是基于当前约 538 pps 观测值的示例；生产值应根据所有 BFD 流总和再留余量。不要把 meter 设置得低于正常 BFD 速率，否则会因为主动丢 BFD 包导致会话误判 down。

还要检查 OVS 自身是否配置了过低的 controller 限速：

```bash
ovs-vsctl --columns=name,other_config,status list Bridge
ovs-vsctl --columns=target,is_connected,status list Controller
```

确认：

```text
controller-queue-size
controller-rate-limit
controller-burst-limit
```

如果 `controller-rate-limit` 已启用且低于约 538 pps，或者 burst 太小，应该调整它；如果没有配置 rate limit，则 overflow 更可能来自 `ovn-controller` 消费不及时、CPU 调度延迟、OpenFlow socket 写阻塞，或其他 packet-in 流量叠加。

最后，建议临时打开日志观察消费端是否跟不上：

```bash
ovs-appctl -t ovs-vswitchd vlog/set connmgr:dbg
ovs-appctl -t ovn-controller vlog/set pinctrl:dbg
```

将这次 flow counter 的增速、tcpdump 速率、`ovn-controller` CPU、CoPP meter drop 计数和 overflow 日志按同一时间窗口对齐，基本可以区分是：

- BFD 总速率过高；
- OVS controller rate/burst 配置过低；
- CoPP 主动丢包；
- `ovn-controller` CPU 或 OpenFlow 连接消费阻塞；
- 非 BFD packet-in 风暴。

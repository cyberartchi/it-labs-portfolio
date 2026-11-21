# Findings - WAN Impairment Measurements

## Commands

Baseline:

~~~bash
ip -4 route
tc qdisc show
ping -c 20 1.1.1.1
~~~

Profile A:

~~~bash
tc qdisc add dev eth0 root netem delay 120ms 20ms loss 2%
tc qdisc show dev eth0
ping -c 50 -i 0.2 -q 1.1.1.1
~~~

Profile B:

~~~bash
tc qdisc change dev eth0 root netem delay 450ms 40ms loss 1%
tc qdisc show dev eth0
ping -c 50 -i 0.2 -q 1.1.1.1
~~~

Cleanup:

~~~bash
tc qdisc del dev eth0 root
tc qdisc show dev eth0
ping -c 20 -q 1.1.1.1
~~~

## Results

| State | Sent | Received | Loss | Minimum RTT | Average RTT | Maximum RTT | mdev |
|---|---:|---:|---:|---:|---:|---:|---:|
| Baseline | 20 | 20 | 0% | 28.131 ms | 28.578 ms | 29.142 ms | 0.290 ms |
| Profile A | 50 | 49 | 2% | 129.036 ms | 151.114 ms | 168.552 ms | 11.885 ms |
| Profile B | 50 | 49 | 2% | 439.684 ms | 476.989 ms | 518.628 ms | 22.793 ms |
| Restored | 20 | 20 | 0% | 27.900 ms | 28.421 ms | 28.860 ms | 0.298 ms |

## Interpretation

### Delay

Profile A raised average RTT by 122.536 ms relative to the baseline. Profile B raised it by 448.411 ms. These values closely track the configured 120 ms and 450 ms egress delays.

The delay was not doubled because the qdisc affected packets leaving `eth0`; the returning echo replies entered the interface without passing through the same egress qdisc.

### Jitter

The baseline mdev was only 0.290 ms. It increased to 11.885 ms and then 22.793 ms as the configured delay variation increased. This demonstrates why average latency alone is insufficient for evaluating interactive traffic.

### Packet Loss

Profile A configured 2% random loss and produced one lost packet in 50 probes, exactly 2%.

Profile B configured 1% random loss but also produced one lost packet in 50 probes. A 50-packet test can only show loss in two-percentage-point increments, and random impairment does not guarantee that a short sample will equal the configured probability.

### Concurrent Probes

The high-latency result included `pipe 3`. The ping interval was 0.2 seconds while RTT was approximately 0.477 seconds, so several requests were outstanding simultaneously. This was expected and did not indicate a routing loop.

### Restoration

After deleting the root qdisc, `tc qdisc show dev eth0` again reported `noqueue`. The restored average RTT differed from the original baseline by only 0.157 ms and packet loss returned to 0%.

## Limitations

- ICMP latency does not measure application throughput or available bandwidth.
- The tests used one external destination and therefore describe that path at that time.
- The 20- and 50-packet samples are intentionally short and are not long-term SLA measurements.
- The qdisc was attached only to test01 egress, not to both directions of the complete WAN path.
- Random loss results vary between runs.
- No physical second WAN or live failover was tested in this module.

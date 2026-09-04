# Network Diagnostic Scenarios

## Objective

The objective of this exercise was to interpret network test results without assuming a single cause too early. Each scenario separates confirmed facts from hypotheses and identifies the next useful diagnostic test.

## Diagnostic Method

For each scenario, the analysis followed five steps:

1. describe the reported symptom;
2. record confirmed test results;
3. identify what the evidence proves;
4. list multiple causes consistent with the evidence;
5. select the next test that can distinguish between the hypotheses.

## Scenario 1 - IP Connectivity Works but Name Resolution Fails

### Evidence

```text
ping 1.1.1.1                         -> replies received
nslookup example.com                 -> timeout
Test-NetConnection 1.1.1.1 -Port 443 -> TcpTestSucceeded: True
```

### Assessment

IP connectivity and TCP connectivity to the tested address were working. The failure was related to DNS resolution rather than a complete loss of network access.

### Possible Causes

- an unavailable configured DNS resolver;
- an incorrect DNS server address;
- filtering of DNS traffic;
- a resolver service failure;
- a problem with the local DNS configuration or cache.

### Next Test

Inspect the DNS configuration with `ipconfig /all`, query the configured resolver directly, and compare the result with another permitted DNS resolver.

### Limitation

A DNS timeout alone does not identify whether the cause is the client configuration, resolver, firewall, or network path.

## Scenario 2 - Host Responds but One Service Does Not

### Evidence

```text
ping 10.0.0.20                         -> replies received
Test-NetConnection 10.0.0.20 -Port 22  -> False
Test-NetConnection 10.0.0.20 -Port 443 -> True
```

### Assessment

The host was reachable and at least one TCP service was available. Failure on TCP port 22 did not mean that the entire server was unavailable.

### Possible Causes

- the SSH service was not running;
- SSH was listening on a different port or interface;
- a host firewall or network ACL blocked TCP port 22;
- the service accepted connections only from selected source addresses.

### Next Test

Check the listener and service status on the destination host, review firewall rules, and repeat the test from an authorized network segment.

### Limitation

Successful ICMP replies prove host reachability for ICMP, but they do not prove that every TCP service is available.

## Scenario 3 - Repeated SYN Packets Without a Response

### Evidence

```text
Client -> server: TCP SYN
Client -> server: TCP SYN retransmission
Client -> server: TCP SYN retransmission
No SYN-ACK or RST observed
```

### Assessment

The client did not receive a TCP response to its connection attempt. The capture did not prove one specific cause.

### Possible Causes

- silent firewall filtering;
- packet loss;
- incorrect routing;
- a missing return route;
- an unavailable host;
- a capture point that did not observe the response path.

### Next Test

Verify routing and host availability, inspect firewall or ACL logs, test the service locally on the destination host, and capture traffic closer to the server if possible.

### Limitation

If a closed port actively responds, a TCP RST is normally expected. The absence of both SYN-ACK and RST does not by itself prove that a firewall caused the failure.

## Conclusion

The scenarios demonstrated that successful tests have a limited scope. DNS, ICMP, and individual TCP ports must be tested separately. A professional diagnosis should state what is known, what remains uncertain, and which test should be performed next.

[Back to module](../README.md)


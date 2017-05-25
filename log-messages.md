# Exploration for Network Log Forwarder

We are exploring how we can improve the logs for packets that are accepted or denied by 
parsing the kernel logs and joining log lines with metadata for the applications running
on the cell where the logging occurs.

Here are two examples of what log lines may look like in the future:

For ingress traffic on the overlay network:

```
{
  "timestamp": "1495732374.512016535",
  "source": "network-log-forwarder",
  "message": "ingress-allowed",
  "log_level": 1,
  "data": {
    "destination": {
      "container_id": "ffec8b0f-bdcf-42e7-7a89-f01e",
      "app_guid": "c5836386-b934-424a-8d93-5894115b3130",
      "space_guid": "8bd0678d-c5fd-4991-8a5c-a75c75440e0b",
      "organization_guid": "d95a6357-6124-48fe-80f9-c8ff6c250a6e"
    },
    "packet": {
      "src_ip": "10.255.15.7",
      "src_port": 35236,
      "dst_ip": "10.255.15.13",
      "dst_port": 8080,
      "protocol": "tcp",
      "mark": "0x2"
    }
  }
}
```

For egress traffic not on the overlay network:

```
{
  "timestamp": "1495732374.512016535",
  "source": "network-log-forwarder",
  "message": "egress-allowed",
  "log_level": 1,
  "data": {
    "source": {
      "container_id": "ffec8b0f-bdcf-42e7-7a89-f01e",
      "app_guid": "c5836386-b934-424a-8d93-5894115b3130",
      "space_guid": "8bd0678d-c5fd-4991-8a5c-a75c75440e0b",
      "organization_guid": "d95a6357-6124-48fe-80f9-c8ff6c250a6e"
    },
    "packet": {
      "src_ip": "10.255.15.7",
      "src_port": 35236,
      "dst_ip": "173.294.210.139",
      "dst_port": 80,
      "protocol": "tcp",
      "mark": "0x2"
    }
  }
}
```

## Notes

The `message` field may be one of the following:

- ingress-allowed
- ingress-denied
- egress-allowed
- egress-denied

For **ingress** traffic, the metadata contains information about the **destination** application.
If the packet comes from an application that has a policy, we will log the mark as well, which could
be used to look up the source application, but this information is not readily available on the cell
to include in the log lines.

For **egress** traffic, the metadata contains information about the **source** application.


### More Example Log Lines

Here are some examples of current log lines from the kernel log:

### C2C (Ingress on the Overlay)

Accept

```
May  3 23:34:07 localhost kernel: [87921.493829] DENY_C2C_cb40f81e-52ce-41c5- IN=s-010255015007 OUT=s-010255015013 MAC=aa:aa:0a:ff:0f:07:ee:ee:0a:ff:0f:07:08:00 SRC=10.255.15.7 DST=10.255.15.13 LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=35889 DF PROTO=TCP SPT=36004 DPT=723 WINDOW=29200 RES=0x00 SYN URGP=0 MARK=0x2
```

Deny

```
May  3 23:35:07 localhost kernel: [87981.320056] OK_0002_e9e8959f-3828-4136-8 IN=s-010255015007 OUT=s-010255015013 MAC=aa:aa:0a:ff:0f:07:ee:ee:0a:ff:0f:07:08:00 SRC=10.255.15.7 DST=10.255.15.13 LEN=52 TOS=0x00 PREC=0x00 TTL=63 ID=43997 DF PROTO=TCP SPT=60012 DPT=8080 WINDOW=237 RES=0x00 ACK URGP=0 MARK=0x2
```


## ASG (Egress not on the Overlay)

### deny

TCP
```
May  3 23:35:58 localhost kernel: [88032.025828] DENY_d538d169-f2f6-4587-77b1 IN=s-010255015007 OUT=eth0 MAC=aa:aa:0a:ff:0f:07:ee:ee:0a:ff:0f:07:08:00 SRC=10.255.15.7 DST=10.10.10.1 LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=61375 DF PROTO=TCP SPT=49466 DPT=80 WINDOW=29200 RES=0x00 SYN URGP=0 MARK=0x2
```

### accept

```
May  3 23:35:35 localhost kernel: [88008.920287] OK_d538d169-f2f6-4587-77b1-f IN=s-010255015007 OUT=eth0 MAC=aa:aa:0a:ff:0f:07:ee:ee:0a:ff:0f:07:08:00 SRC=10.255.15.7 DST=173.194.210.139 LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=45400 DF PROTO=TCP SPT=35236 DPT=80 WINDOW=29200 RES=0x00 SYN URGP=0 MARK=0x2
```

### other protocols

UDP

```
May 25 17:17:54 localhost kernel: [173652.249504] DENY_da966cab-6a60-49c4-4f90 IN=s-010247180118 OUT=eth0 MAC=aa:aa:0a:f7:b4:76:ee:ee:0a:f7:b4:76:08:00 SRC=10.247.180.118 DST=10.0.0.1 LEN=37 TOS=0x00 PREC=0x00 TTL=63 ID=13026 DF PROTO=UDP SPT=54204 DPT=3454 LEN=17
```

ICMP
```
May 25 17:19:38 localhost kernel: [173756.041192] DENY_da966cab-6a60-49c4-4f90 IN=s-010247180118 OUT=eth0 MAC=aa:aa:0a:f7:b4:76:ee:ee:0a:f7:b4:76:08:00 SRC=10.247.180.118 DST=10.0.0.1 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=58750 DF PROTO=ICMP TYPE=8 CODE=0 ID=172 SEQ=1
```




## Future Log Line Examples

### Ingress


TCP

```
{
  "timestamp": "1495732374.512016535",
  "source": "network-log-forwarder",
  "message": "ingress-allowed",
  "log_level": 1,
  "data": {
    "destination": {
      "container_id": "ffec8b0f-bdcf-42e7-7a89-f01e",
      "app_guid": "c5836386-b934-424a-8d93-5894115b3130",
      "space_guid": "8bd0678d-c5fd-4991-8a5c-a75c75440e0b",
      "organization_guid": "d95a6357-6124-48fe-80f9-c8ff6c250a6e"
    },
    "packet": {
      "src_ip": "10.255.15.7",
      "src_port": 35236,
      "dst_ip": "10.255.15.13",
      "dst_port": 8080,
      "protocol": "tcp",
      "mark": "0x2"
    }
  }
}
```

### Egress - All Protocols

TCP

```
{
  "timestamp": "1495732374.512016535",
  "source": "network-log-forwarder",
  "message": "egress-allowed",
  "log_level": 1,
  "data": {
    "source": {
      "container_id": "ffec8b0f-bdcf-42e7-7a89-f01e",
      "app_guid": "c5836386-b934-424a-8d93-5894115b3130",
      "space_guid": "8bd0678d-c5fd-4991-8a5c-a75c75440e0b",
      "organization_guid": "d95a6357-6124-48fe-80f9-c8ff6c250a6e"
    },
    "packet": {
      "src_ip": "10.255.15.7",
      "src_port": 35236,
      "dst_ip": "173.294.210.139",
      "dst_port": 80,
      "protocol": "tcp",
      "mark": "0x2"
    }
  }
}
```

UDP

```
{
  "timestamp": "1495732374.512016535",
  "source": "network-log-forwarder",
  "message": "egress-allowed",
  "log_level": 1,
  "data": {
    "source": {
      "container_id": "ffec8b0f-bdcf-42e7-7a89-f01e",
      "app_guid": "c5836386-b934-424a-8d93-5894115b3130",
      "space_guid": "8bd0678d-c5fd-4991-8a5c-a75c75440e0b",
      "organization_guid": "d95a6357-6124-48fe-80f9-c8ff6c250a6e"
    },
    "packet": {
      "src_ip": "10.255.15.7",
      "src_port": 35236,
      "dst_ip": "173.294.210.139",
      "dst_port": 80,
      "protocol": "udp",
      "mark": "0x2"
    }
  }
}
```

ICMP

```
{
  "timestamp": "1495732374.512016535",
  "source": "network-log-forwarder",
  "message": "egress-allowed",
  "log_level": 1,
  "data": {
    "source": {
      "container_id": "ffec8b0f-bdcf-42e7-7a89-f01e",
      "app_guid": "c5836386-b934-424a-8d93-5894115b3130",
      "space_guid": "8bd0678d-c5fd-4991-8a5c-a75c75440e0b",
      "organization_guid": "d95a6357-6124-48fe-80f9-c8ff6c250a6e"
    },
    "packet": {
      "src_ip": "10.255.15.7",
      "dst_ip": "173.294.210.139",
      "protocol": "icmp",
      "type": 8,
      "code": 0,
      "mark": "0x2"
    }
  }
}
```

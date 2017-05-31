# Tolerance to etcd data loss

**Legacy** versions of [CF Networking Release](https://github.com/cloudfoundry-incubator/cf-networking-release)
used [flannel](https://github.com/coreos/flannel).  Flannel relies on [etcd](https://github.com/coreos/etcd).

We analyzed how etcd outages and data loss would affect container connectivity.  To see the results of this
analysis, [see this document](https://github.com/cloudfoundry-incubator/cf-networking-release/blob/v0.24.0/docs/etcd-data-loss-tolerance.md).

---

**Note**: Newer versions of `cf-networking-release` use [Silk](https://github.com/cloudfoundry-incubator/silk),
which does not suffer from these problems.

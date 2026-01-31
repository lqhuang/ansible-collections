# Kernel (runtime) configuration

## References

- [IP Sysctl — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
- [Netfilter Conntrack Sysfs variables — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/networking/nf_conntrack-sysctl.html)
- [Documentation for /proc/sys/kernel/ — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/admin-guide/sysctl/kernel.html)
- [Documentation for /proc/sys/fs/ — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/admin-guide/sysctl/fs.html)
- [sysctl defaults from systemd](https://github.com/systemd/systemd/blob/main/sysctl.d/50-default.conf)
- [sysctl defaults from debian](https://salsa.debian.org/kernel-team/linux-base/-/blob/debian/latest/sysctl.d/50-default.conf?ref_type=heads)
- [Linux Tuning | Fasterdata](https://fasterdata.es.net/host-tuning/linux): This page contains a quick reference guide for Linux tuning.

## Different levels

- `essential` (51)
  - Basic kernel runtime configuration that should be applied to all systems.
- `recommend` (60)
  - Recommended kernel runtime configuration for most systems.
  - Builds upon `essential` level.
- Fit for different purposes (70)
  - Builds upon `recommend` level.
  - `network`
  - `multitenant`
  - `workstation`
  - ...
- `security` (90)
  - Security hardened kernel runtime configuration.
  - Builds upon `essential` level.

## Depends on real use cases

- `vm.vfs_cache_pressure`
  - At the default value of `vfs_cache_pressure=100` the kernel will attempt to reclaim dentries and inodes at a "fair" rate with respect to pagecache and swapcache reclaim.
  - Decreasing `vfs_cache_pressure` causes the kernel to prefer to retain dentry and inode caches.
  - Increasing `vfs_cache_pressure` beyond 100 causes the kernel to prefer to reclaim dentries and inodes.
- `vm.max_map_count`
  - at least `1048576` (64GB - 128GB RAM), or higher
  - for large applications relying on mmap, such as databases, etc.
- `vm.nr_hugepages`
- `net.ipv4.conf.all.rp_filter`
  - Strict mode (1) or loose mode (2)
  - Depends on network topology and use cases
  - Loose mode is more appropriate for hosts that have multiple links with routes to the same networks (e.g. a client with a Wi-Fi and Ethernet both connected to the internet).
  - ```
    net.ipv4.conf.all.rp_filter = 1
    net.ipv4.conf.default.rp_filter = 1
    ```
- `net.ipv4.tcp_notsent_lowat`: 131072
  - Server side can consider setting `tcp_notsent_lowat`
  - This setting can improve interactive latency by optimizing send buffer handling. Note that this kernel setting is useful for HTTP/2 and may be detrimental to other applications, as it applies to all applications on the server.

## Different voices

- `net.ipv4.tcp_syncookies` = 1 / 0
  - Enable TCP SYN cookies to protect against SYN flood attacks.
  - Some sources recommend disabling syncookies for performance reasons, but the performance impact is need to be carefully evaluated against the security benefits.
  - [Improving syncookies - LWN.net](https://lwn.net/Articles/277146/)

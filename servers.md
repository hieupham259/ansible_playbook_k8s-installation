# Server Inventory

> OS: **Ubuntu Server 24.04 LTS** cho cả 3 server.

| # | Server        | Hostname      | IP                | RAM (GB) | CPU | Disk (GB) | Domain                                    |
|---|---------------|---------------|-------------------|----------|-----|-----------|-------------------------------------------|
| 1 | ubuntu-2404   | ubuntu-2404   | 192.168.100.100   | 4        | 2   | 40        |                                           |
| 2 | load-balancer | load-balancer | 192.168.100.101   | 2        | 1   | 40        |                                           |
| 3 | teleport      | teleport      | 192.168.100.103   | 2        | 1   | 40        | https://teleport-onpre.devopseduvn.live   |

## Kubernetes Cluster Inventory

| # | Node / Role   | Hostname      | IP                | RAM (GB) | vCPU | Disk SSD (GB) |
|---|---------------|---------------|-------------------|----------|------|---------------|
| 1 | Control plane | k8s-master    | 192.168.100.111   | 8        | 4    | 40            |
| 2 | Worker        | k8s-worker1   | 192.168.100.112   | 6        | 2    | 40            |
| 3 | Worker        | k8s-worker2   | 192.168.100.113   | 6        | 2    | 40            |

> Hai nhóm VM dùng các IP riêng biệt nên có thể chạy đồng thời, miễn toàn bộ địa chỉ đều nằm ngoài DHCP pool hoặc đã được router giữ chỗ theo MAC.

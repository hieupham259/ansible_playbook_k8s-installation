# Khắc phục sự cố cluster (Troubleshooting Clusters)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-cluster/>
>
> Gỡ lỗi các sự cố cluster thường gặp.

Tài liệu này nói về việc khắc phục sự cố (troubleshooting) ở cấp cluster; chúng tôi giả định
rằng bạn đã loại trừ khả năng ứng dụng của bạn là nguyên nhân gốc của vấn đề bạn đang gặp phải.
Xem [hướng dẫn khắc phục sự cố ứng dụng](297-debug-application-vi.md)
để có các mẹo gỡ lỗi ứng dụng. Bạn cũng có thể xem
[tài liệu tổng quan về khắc phục sự cố](296-debug-vi.md) để biết thêm
thông tin.

Để khắc phục sự cố kubectl, hãy tham khảo
[Khắc phục sự cố kubectl](314-troubleshoot-kubectl-vi.md).

## Liệt kê cluster của bạn (Listing your cluster)

Điều đầu tiên cần gỡ lỗi trong cluster là kiểm tra xem tất cả các node của bạn đã được đăng ký
đúng hay chưa.

Chạy lệnh sau:

```shell
kubectl get nodes
```

Và xác nhận rằng tất cả các node bạn mong đợi đều xuất hiện và tất cả đều ở trạng thái `Ready`.

Để có thông tin chi tiết về tình trạng tổng thể của cluster, bạn có thể chạy:

```shell
kubectl cluster-info dump
```

### Ví dụ: gỡ lỗi một node bị down hoặc không thể kết nối (Example: debugging a down/unreachable node)

Đôi khi trong lúc gỡ lỗi, việc xem trạng thái của một node có thể hữu ích -- ví dụ, vì bạn
nhận thấy hành vi lạ của một Pod đang chạy trên node đó, hoặc để tìm hiểu tại sao một Pod
không được lập lịch (schedule) lên node đó. Cũng giống như với Pod, bạn có thể dùng
`kubectl describe node` và `kubectl get node -o yaml` để truy xuất thông tin chi tiết về node.
Ví dụ, đây là những gì bạn sẽ thấy nếu một node bị down (mất kết nối mạng, hoặc kubelet chết
và không khởi động lại được, v.v.). Hãy chú ý các event cho thấy node ở trạng thái NotReady,
và cũng chú ý rằng các pod không còn chạy nữa (chúng bị trục xuất — evicted — sau năm phút ở
trạng thái NotReady).

```shell
kubectl get nodes
```

```none
NAME                     STATUS       ROLES     AGE     VERSION
kube-worker-1            NotReady     <none>    1h      v1.23.3
kubernetes-node-bols     Ready        <none>    1h      v1.23.3
kubernetes-node-st6x     Ready        <none>    1h      v1.23.3
kubernetes-node-unaj     Ready        <none>    1h      v1.23.3
```

```shell
kubectl describe node kube-worker-1
```

```none
Name:               kube-worker-1
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=kube-worker-1
                    kubernetes.io/os=linux
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 17 Feb 2022 16:46:30 -0500
Taints:             node.kubernetes.io/unreachable:NoExecute
                    node.kubernetes.io/unreachable:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  kube-worker-1
  AcquireTime:     <unset>
  RenewTime:       Thu, 17 Feb 2022 17:13:09 -0500
Conditions:
  Type                 Status    LastHeartbeatTime                 LastTransitionTime                Reason              Message
  ----                 ------    -----------------                 ------------------                ------              -------
  NetworkUnavailable   False     Thu, 17 Feb 2022 17:09:13 -0500   Thu, 17 Feb 2022 17:09:13 -0500   WeaveIsUp           Weave pod has set this
  MemoryPressure       Unknown   Thu, 17 Feb 2022 17:12:40 -0500   Thu, 17 Feb 2022 17:13:52 -0500   NodeStatusUnknown   Kubelet stopped posting node status.
  DiskPressure         Unknown   Thu, 17 Feb 2022 17:12:40 -0500   Thu, 17 Feb 2022 17:13:52 -0500   NodeStatusUnknown   Kubelet stopped posting node status.
  PIDPressure          Unknown   Thu, 17 Feb 2022 17:12:40 -0500   Thu, 17 Feb 2022 17:13:52 -0500   NodeStatusUnknown   Kubelet stopped posting node status.
  Ready                Unknown   Thu, 17 Feb 2022 17:12:40 -0500   Thu, 17 Feb 2022 17:13:52 -0500   NodeStatusUnknown   Kubelet stopped posting node status.
Addresses:
  InternalIP:  192.168.0.113
  Hostname:    kube-worker-1
Capacity:
  cpu:                2
  ephemeral-storage:  15372232Ki
  hugepages-2Mi:      0
  memory:             2025188Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  14167048988
  hugepages-2Mi:      0
  memory:             1922788Ki
  pods:               110
System Info:
  Machine ID:                 9384e2927f544209b5d7b67474bbf92b
  System UUID:                aa829ca9-73d7-064d-9019-df07404ad448
  Boot ID:                    5a295a03-aaca-4340-af20-1327fa5dab5c
  Kernel Version:             5.13.0-28-generic
  OS Image:                   Ubuntu 21.10
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.5.9
  Kubelet Version:            v1.23.3
  Kube-Proxy Version:         v1.23.3
Non-terminated Pods:          (4 in total)
  Namespace                   Name                                 CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                 ------------  ----------  ---------------  -------------  ---
  default                     nginx-deployment-67d4bdd6f5-cx2nz    500m (25%)    500m (25%)  128Mi (6%)       128Mi (6%)     23m
  default                     nginx-deployment-67d4bdd6f5-w6kd7    500m (25%)    500m (25%)  128Mi (6%)       128Mi (6%)     23m
  kube-system                 kube-proxy-dnxbz                     0 (0%)        0 (0%)      0 (0%)           0 (0%)         28m
  kube-system                 weave-net-gjxxp                      100m (5%)     0 (0%)      200Mi (10%)      0 (0%)         28m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                1100m (55%)  1 (50%)
  memory             456Mi (24%)  256Mi (13%)
  ephemeral-storage  0 (0%)       0 (0%)
  hugepages-2Mi      0 (0%)       0 (0%)
Events:
...
```

```shell
kubectl get node kube-worker-1 -o yaml
```

```yaml
apiVersion: v1
kind: Node
metadata:
  annotations:
    node.alpha.kubernetes.io/ttl: "0"
    volumes.kubernetes.io/controller-managed-attach-detach: "true"
  creationTimestamp: "2022-02-17T21:46:30Z"
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: kube-worker-1
    kubernetes.io/os: linux
  name: kube-worker-1
  resourceVersion: "4026"
  uid: 98efe7cb-2978-4a0b-842a-1a7bf12c05f8
spec: {}
status:
  addresses:
  - address: 192.168.0.113
    type: InternalIP
  - address: kube-worker-1
    type: Hostname
  allocatable:
    cpu: "2"
    ephemeral-storage: "14167048988"
    hugepages-2Mi: "0"
    memory: 1922788Ki
    pods: "110"
  capacity:
    cpu: "2"
    ephemeral-storage: 15372232Ki
    hugepages-2Mi: "0"
    memory: 2025188Ki
    pods: "110"
  conditions:
  - lastHeartbeatTime: "2022-02-17T22:20:32Z"
    lastTransitionTime: "2022-02-17T22:20:32Z"
    message: Weave pod has set this
    reason: WeaveIsUp
    status: "False"
    type: NetworkUnavailable
  - lastHeartbeatTime: "2022-02-17T22:20:15Z"
    lastTransitionTime: "2022-02-17T22:13:25Z"
    message: kubelet has sufficient memory available
    reason: KubeletHasSufficientMemory
    status: "False"
    type: MemoryPressure
  - lastHeartbeatTime: "2022-02-17T22:20:15Z"
    lastTransitionTime: "2022-02-17T22:13:25Z"
    message: kubelet has no disk pressure
    reason: KubeletHasNoDiskPressure
    status: "False"
    type: DiskPressure
  - lastHeartbeatTime: "2022-02-17T22:20:15Z"
    lastTransitionTime: "2022-02-17T22:13:25Z"
    message: kubelet has sufficient PID available
    reason: KubeletHasSufficientPID
    status: "False"
    type: PIDPressure
  - lastHeartbeatTime: "2022-02-17T22:20:15Z"
    lastTransitionTime: "2022-02-17T22:15:15Z"
    message: kubelet is posting ready status
    reason: KubeletReady
    status: "True"
    type: Ready
  daemonEndpoints:
    kubeletEndpoint:
      Port: 10250
  nodeInfo:
    architecture: amd64
    bootID: 22333234-7a6b-44d4-9ce1-67e31dc7e369
    containerRuntimeVersion: containerd://1.5.9
    kernelVersion: 5.13.0-28-generic
    kubeProxyVersion: v1.23.3
    kubeletVersion: v1.23.3
    machineID: 9384e2927f544209b5d7b67474bbf92b
    operatingSystem: linux
    osImage: Ubuntu 21.10
    systemUUID: aa829ca9-73d7-064d-9019-df07404ad448
```

## Xem log (Looking at logs)

Hiện tại, việc đào sâu hơn vào cluster đòi hỏi phải đăng nhập vào các máy liên quan. Dưới đây
là vị trí của các file log liên quan. Trên các hệ thống dùng systemd, bạn có thể cần dùng
`journalctl` thay vì xem trực tiếp các file log.

### Các node control plane (Control Plane nodes)

* `/var/log/kube-apiserver.log` - API Server, chịu trách nhiệm phục vụ API
* `/var/log/kube-scheduler.log` - Scheduler, chịu trách nhiệm đưa ra các quyết định lập lịch
* `/var/log/kube-controller-manager.log` - thành phần chạy hầu hết các controller có sẵn của
  Kubernetes, với ngoại lệ đáng chú ý là việc lập lịch (kube-scheduler đảm nhận việc lập lịch).

### Các node worker (Worker Nodes)

* `/var/log/kubelet.log` - log từ kubelet, chịu trách nhiệm chạy các container trên node
* `/var/log/kube-proxy.log` - log từ `kube-proxy`, chịu trách nhiệm điều hướng lưu lượng đến
  các endpoint của Service

## Các dạng lỗi của cluster (Cluster failure modes)

Đây là danh sách chưa đầy đủ về những điều có thể trục trặc, và cách điều chỉnh thiết lập
cluster của bạn để giảm thiểu các vấn đề đó.

### Các nguyên nhân góp phần (Contributing causes)

- VM bị tắt (shutdown)
- Phân mảnh mạng (network partition) bên trong cluster, hoặc giữa cluster và người dùng
- Crash trong phần mềm Kubernetes
- Mất dữ liệu hoặc lưu trữ bền vững (persistent storage) không khả dụng (ví dụ volume GCE PD
  hoặc AWS EBS)
- Lỗi của người vận hành (operator error), ví dụ cấu hình sai phần mềm Kubernetes hoặc phần
  mềm ứng dụng

### Các kịch bản cụ thể (Specific scenarios)

- VM của API server bị tắt hoặc apiserver bị crash
  - Hậu quả
    - không thể dừng, cập nhật, hoặc khởi chạy pod, service, replication controller mới
    - các pod và service hiện có sẽ tiếp tục hoạt động bình thường, trừ khi chúng phụ thuộc
      vào Kubernetes API
- Mất kho lưu trữ nền (backing storage) của API server
  - Hậu quả
    - thành phần kube-apiserver không khởi động thành công và không trở nên healthy
    - các kubelet sẽ không thể kết nối tới nó nhưng vẫn tiếp tục chạy các pod hiện có và cung
      cấp service proxy như cũ
    - cần khôi phục thủ công hoặc tạo lại trạng thái của apiserver trước khi khởi động lại
      apiserver
- VM của các dịch vụ hỗ trợ (node controller, replication controller manager, scheduler, v.v.)
  bị tắt hoặc crash
  - hiện tại các thành phần này được đặt cùng chỗ (colocated) với apiserver, nên khi chúng
    không khả dụng thì hậu quả tương tự như apiserver
  - trong tương lai, các thành phần này cũng sẽ được nhân bản (replicated) và có thể không
    được đặt cùng chỗ nữa
  - chúng không có trạng thái bền vững (persistent state) riêng
- Một node đơn lẻ (VM hoặc máy vật lý) bị tắt
  - Hậu quả
    - các pod trên Node đó ngừng chạy
- Phân mảnh mạng
  - Hậu quả
    - phân vùng A cho rằng các node trong phân vùng B đã down; phân vùng B cho rằng apiserver
      đã down. (Giả sử VM master nằm trong phân vùng A.)
- Lỗi phần mềm kubelet
  - Hậu quả
    - kubelet bị crash không thể khởi chạy pod mới trên node
    - kubelet có thể xóa các pod hoặc không
    - node bị đánh dấu là unhealthy
    - các replication controller khởi chạy pod mới ở nơi khác
- Lỗi của người vận hành cluster
  - Hậu quả
    - mất pod, service, v.v.
    - mất kho lưu trữ nền của apiserver
    - người dùng không thể đọc API
    - v.v.

### Các biện pháp giảm thiểu (Mitigations)

- Hành động: Dùng tính năng tự động khởi động lại VM của nhà cung cấp IaaS cho các VM IaaS
  - Giảm thiểu: VM của apiserver bị tắt hoặc apiserver bị crash
  - Giảm thiểu: VM của các dịch vụ hỗ trợ bị tắt hoặc crash

- Hành động: Dùng lưu trữ tin cậy của nhà cung cấp IaaS (ví dụ volume GCE PD hoặc AWS EBS)
  cho các VM chạy apiserver+etcd
  - Giảm thiểu: Mất kho lưu trữ nền của apiserver

- Hành động: Dùng cấu hình [tính sẵn sàng cao (high availability)](08-high-availability-vi.md)
  - Giảm thiểu: Node control plane bị tắt hoặc các thành phần control plane (scheduler, API
    server, controller-manager) bị crash
    - Chịu được một hoặc nhiều node hoặc thành phần bị lỗi đồng thời
  - Giảm thiểu: Mất kho lưu trữ nền của API server (tức là thư mục dữ liệu của etcd)
    - Giả định dùng cấu hình etcd HA (highly-available)

- Hành động: Chụp snapshot định kỳ các PD/EBS-volume của apiserver
  - Giảm thiểu: Mất kho lưu trữ nền của apiserver
  - Giảm thiểu: Một số trường hợp lỗi của người vận hành
  - Giảm thiểu: Một số trường hợp lỗi phần mềm Kubernetes

- Hành động: dùng replication controller và service đứng trước các pod
  - Giảm thiểu: Node bị tắt
  - Giảm thiểu: Lỗi phần mềm kubelet

- Hành động: thiết kế ứng dụng (container) chịu được việc khởi động lại bất ngờ
  - Giảm thiểu: Node bị tắt
  - Giảm thiểu: Lỗi phần mềm kubelet

## Tiếp theo (What's next)

* Tìm hiểu về các metric có sẵn trong
  [Resource Metrics Pipeline](311-resource-metrics-pipeline-vi.md)
* Khám phá thêm các công cụ để
  [giám sát mức sử dụng tài nguyên](312-resource-usage-monitoring-vi.md)
* Dùng Node Problem Detector để
  [giám sát tình trạng node](310-monitor-node-health-vi.md)
* Dùng `kubectl debug node` để [gỡ lỗi các node Kubernetes](308-kubectl-node-debug-vi.md)
* Dùng `crictl` để [gỡ lỗi các node Kubernetes](307-crictl-vi.md)
* Tìm hiểu thêm về [Kubernetes auditing](306-audit-vi.md)
* Dùng `telepresence` để [phát triển và gỡ lỗi service cục bộ](309-local-debugging-vi.md)

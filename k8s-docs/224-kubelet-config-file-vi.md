# Thiết lập tham số kubelet qua file cấu hình (Set Kubelet Parameters Via A Configuration File)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/>

Một tập con các tham số cấu hình của kubelet có thể được thiết lập thông qua một file cấu hình
trên đĩa, thay thế cho các cờ (flag) dòng lệnh.

Cung cấp tham số qua file cấu hình là cách tiếp cận được khuyến nghị, vì nó đơn giản hóa việc
triển khai node và quản lý cấu hình.

## Trước khi bạn bắt đầu (Before you begin)

Một số bước trong trang này sử dụng công cụ `jq`. Nếu bạn chưa có `jq`, bạn có thể cài đặt nó
qua nguồn phần mềm của hệ điều hành, hoặc tải về từ
[https://jqlang.github.io/jq/](https://jqlang.github.io/jq/).

Một số bước cũng cần tới `curl`, công cụ này có thể được cài đặt qua nguồn phần mềm của hệ
điều hành.

## Tạo file cấu hình (Create the config file)

Tập con cấu hình của kubelet có thể được cấu hình qua file được định nghĩa bởi struct
[`KubeletConfiguration`](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).

File cấu hình phải là một biểu diễn JSON hoặc YAML của các tham số trong struct này. Hãy đảm
bảo kubelet có quyền đọc trên file.

Dưới đây là một ví dụ về nội dung file này:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: "192.168.0.8"
port: 20250
serializeImagePulls: false
evictionHard:
    memory.available:  "100Mi"
    nodefs.available:  "10%"
    nodefs.inodesFree: "5%"
    imagefs.available: "15%"
    imagefs.inodesFree: "5%"
```

Trong ví dụ này, kubelet được cấu hình với các thiết lập sau:

1. `address`: kubelet sẽ phục vụ trên địa chỉ IP `192.168.0.8`.
1. `port`: kubelet sẽ phục vụ trên port `20250`.
1. `serializeImagePulls`: việc kéo (pull) image sẽ được thực hiện song song.
1. `evictionHard`: kubelet sẽ trục xuất (evict) các Pod khi xảy ra một trong các điều kiện sau:

   - Khi bộ nhớ khả dụng của node giảm xuống dưới 100MiB.
   - Khi dung lượng khả dụng của filesystem chính trên node nhỏ hơn 10%.
   - Khi dung lượng khả dụng của filesystem chứa image nhỏ hơn 15%.
   - Khi hơn 95% số inode của filesystem chính trên node đang được sử dụng.

> **Ghi chú:**
>
> Trong ví dụ trên, khi chỉ thay đổi giá trị mặc định của một tham số trong evictionHard, các
> giá trị mặc định của những tham số còn lại sẽ không được kế thừa và sẽ bị đặt về không (zero).
> Để cung cấp các giá trị tùy chỉnh, bạn nên cung cấp đầy đủ tất cả các giá trị ngưỡng tương
> ứng. Hoặc bạn có thể đặt MergeDefaultEvictionSettings thành true trong file cấu hình kubelet;
> khi đó nếu một tham số bị thay đổi thì các tham số khác sẽ kế thừa giá trị mặc định của chúng
> thay vì bằng 0.

`imagefs` là một filesystem tùy chọn mà các container runtime dùng để lưu trữ container image
và các lớp ghi (writable layer) của container.

## Khởi động một tiến trình kubelet được cấu hình qua file cấu hình (Start a kubelet process configured via the config file)

> **Ghi chú:**
>
> Nếu bạn dùng kubeadm để khởi tạo cluster, hãy sử dụng kubelet-config trong lúc tạo cluster
> với `kubeadm init`. Xem chi tiết tại
> [cấu hình kubelet bằng kubeadm](04-kubelet-integration-vi.md).

Khởi động kubelet với cờ `--config` được đặt bằng đường dẫn tới file cấu hình của kubelet.
Kubelet khi đó sẽ nạp cấu hình của nó từ file này.

Lưu ý rằng các cờ dòng lệnh nhắm tới cùng một giá trị với file cấu hình sẽ ghi đè giá trị đó.
Điều này giúp đảm bảo tính tương thích ngược với API dòng lệnh.

Lưu ý rằng các đường dẫn tương đối trong file cấu hình kubelet được phân giải tương đối so với
vị trí của file cấu hình kubelet, trong khi các đường dẫn tương đối trong cờ dòng lệnh được
phân giải tương đối so với thư mục làm việc hiện tại của kubelet.

Lưu ý rằng một số giá trị mặc định khác nhau giữa cờ dòng lệnh và file cấu hình kubelet. Nếu
`--config` được cung cấp và các giá trị không được chỉ định qua dòng lệnh, các giá trị mặc định
của phiên bản `KubeletConfiguration` sẽ được áp dụng. Trong ví dụ trên, phiên bản này là
`kubelet.config.k8s.io/v1beta1`.

## Thư mục drop-in cho các file cấu hình kubelet (Drop-in directory for kubelet configuration files) {#kubelet-conf-d}

Bạn có thể chỉ định một thư mục cấu hình drop-in cho kubelet. Theo mặc định, kubelet không tìm
kiếm các file cấu hình drop-in ở bất kỳ đâu - bạn phải chỉ định một đường dẫn. Ví dụ:
`--config-dir=/etc/kubernetes/kubelet.conf.d`

Với Kubernetes từ v1.28 đến v1.29, bạn chỉ có thể chỉ định `--config-dir` nếu bạn cũng đặt biến
môi trường `KUBELET_CONFIG_DROPIN_DIR_ALPHA` cho tiến trình kubelet (giá trị của biến này không
quan trọng).

> **Ghi chú:**
>
> Hậu tố của một file cấu hình drop-in hợp lệ cho kubelet **bắt buộc** phải là `.conf`. Ví dụ:
> `99-kubelet-address.conf`

Kubelet xử lý các file trong thư mục drop-in cấu hình của nó bằng cách sắp xếp **toàn bộ tên
file** theo thứ tự chữ-số (alphanumeric). Ví dụ, `00-kubelet.conf` được xử lý trước, sau đó bị
ghi đè bởi file có tên `01-kubelet.conf`.

Các file này có thể chứa cấu hình không đầy đủ (partial) nhưng không được sai định dạng và phải
bao gồm metadata về kiểu, cụ thể là `apiVersion` và `kind`. Việc kiểm tra hợp lệ (validation)
chỉ được thực hiện trên cấu trúc cấu hình cuối cùng được lưu nội bộ trong kubelet. Điều này
mang lại sự linh hoạt trong việc quản lý và hợp nhất cấu hình kubelet từ nhiều nguồn khác nhau,
đồng thời ngăn chặn các cấu hình không mong muốn. Tuy nhiên, cần lưu ý rằng hành vi sẽ khác
nhau tùy theo kiểu dữ liệu của các trường cấu hình.

Các kiểu dữ liệu khác nhau trong cấu trúc cấu hình kubelet được hợp nhất (merge) theo cách khác
nhau. Xem [tài liệu tham khảo](https://kubernetes.io/docs/reference/node/kubelet-config-directory-merging/)
để biết thêm thông tin.

### Thứ tự hợp nhất cấu hình kubelet (Kubelet configuration merging order)

Khi khởi động, kubelet hợp nhất cấu hình từ:

* Các feature gate được chỉ định qua dòng lệnh (độ ưu tiên thấp nhất).
* Cấu hình kubelet.
* Các file cấu hình drop-in, theo thứ tự sắp xếp.
* Các đối số dòng lệnh, không tính feature gate (độ ưu tiên cao nhất).

> **Ghi chú:**
>
> Cơ chế thư mục drop-in cấu hình của kubelet tương tự nhưng khác với cách công cụ `kubeadm`
> cho phép bạn vá (patch) cấu hình. Công cụ `kubeadm` dùng một
> [chiến lược vá](03-control-plane-flags-vi.md#patches)
> riêng cho cấu hình của nó, trong khi chiến lược vá duy nhất cho các file drop-in cấu hình
> kubelet là `replace`. Kubelet xác định thứ tự hợp nhất dựa trên việc sắp xếp các **hậu tố**
> theo thứ tự chữ-số, và thay thế mọi trường có mặt trong file có độ ưu tiên cao hơn.

## Xem cấu hình kubelet (Viewing the kubelet configuration)

Vì với tính năng này cấu hình giờ đây có thể được phân tán trên nhiều file, nếu ai đó muốn xem
cấu hình cuối cùng đang thực sự có hiệu lực, họ có thể làm theo các bước sau để xem cấu hình
kubelet:

1. Khởi động một proxy server bằng
   [`kubectl proxy`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_proxy/)
   trong terminal của bạn.

   ```bash
   kubectl proxy
   ```

   Lệnh này cho đầu ra như sau:

   ```none
   Starting to serve on 127.0.0.1:8001
   ```

1. Mở một cửa sổ terminal khác và dùng `curl` để lấy cấu hình kubelet.
   Thay `<node-name>` bằng tên thật của node của bạn:

   ```bash
   curl -X GET http://127.0.0.1:8001/api/v1/nodes/<node-name>/proxy/configz | jq .
   ```

   ```json
   {
     "kubeletconfig": {
       "enableServer": true,
       "staticPodPath": "/var/run/kubernetes/static-pods",
       "syncFrequency": "1m0s",
       "fileCheckFrequency": "20s",
       "httpCheckFrequency": "20s",
       "address": "192.168.1.16",
       "port": 10250,
       "readOnlyPort": 10255,
       "tlsCertFile": "/var/lib/kubelet/pki/kubelet.crt",
       "tlsPrivateKeyFile": "/var/lib/kubelet/pki/kubelet.key",
       "rotateCertificates": true,
       "authentication": {
         "x509": {
           "clientCAFile": "/var/run/kubernetes/client-ca.crt"
         },
         "webhook": {
           "enabled": true,
           "cacheTTL": "2m0s"
         },
         "anonymous": {
           "enabled": true
         }
       },
       "authorization": {
         "mode": "AlwaysAllow",
         "webhook": {
           "cacheAuthorizedTTL": "5m0s",
           "cacheUnauthorizedTTL": "30s"
         }
       },
       "registryPullQPS": 5,
       "registryBurst": 10,
       "eventRecordQPS": 50,
       "eventBurst": 100,
       "enableDebuggingHandlers": true,
       "healthzPort": 10248,
       "healthzBindAddress": "127.0.0.1",
       "oomScoreAdj": -999,
       "clusterDomain": "cluster.local",
       "clusterDNS": [
         "10.0.0.10"
       ],
       "streamingConnectionIdleTimeout": "4h0m0s",
       "nodeStatusUpdateFrequency": "10s",
       "nodeStatusReportFrequency": "5m0s",
       "nodeLeaseDurationSeconds": 40,
       "imageMinimumGCAge": "2m0s",
       "imageMaximumGCAge": "0s",
       "imageGCHighThresholdPercent": 85,
       "imageGCLowThresholdPercent": 80,
       "volumeStatsAggPeriod": "1m0s",
       "cgroupsPerQOS": true,
       "cgroupDriver": "systemd",
       "cpuManagerPolicy": "none",
       "cpuManagerReconcilePeriod": "10s",
       "memoryManagerPolicy": "None",
       "topologyManagerPolicy": "none",
       "topologyManagerScope": "container",
       "runtimeRequestTimeout": "2m0s",
       "hairpinMode": "promiscuous-bridge",
       "maxPods": 110,
       "podPidsLimit": -1,
       "resolvConf": "/run/systemd/resolve/resolv.conf",
       "cpuCFSQuota": true,
       "cpuCFSQuotaPeriod": "100ms",
       "nodeStatusMaxImages": 50,
       "maxOpenFiles": 1000000,
       "contentType": "application/vnd.kubernetes.protobuf",
       "kubeAPIQPS": 50,
       "kubeAPIBurst": 100,
       "serializeImagePulls": true,
       "evictionHard": {
         "imagefs.available": "15%",
         "memory.available": "100Mi",
         "nodefs.available": "10%",
         "nodefs.inodesFree": "5%",
         "imagefs.inodesFree": "5%"
       },
       "evictionPressureTransitionPeriod": "1m0s",
       "mergeDefaultEvictionSettings": false,
       "enableControllerAttachDetach": true,
       "makeIPTablesUtilChains": true,
       "iptablesMasqueradeBit": 14,
       "iptablesDropBit": 15,
       "featureGates": {
         "AllAlpha": false
       },
       "failSwapOn": false,
       "memorySwap": {},
       "containerLogMaxSize": "10Mi",
       "containerLogMaxFiles": 5,
       "configMapAndSecretChangeDetectionStrategy": "Watch",
       "enforceNodeAllocatable": [
         "pods"
       ],
       "volumePluginDir": "/usr/libexec/kubernetes/kubelet-plugins/volume/exec/",
       "logging": {
         "format": "text",
         "flushFrequency": "5s",
         "verbosity": 3,
         "options": {
           "json": {
             "infoBufferSize": "0"
           }
         }
       },
       "enableSystemLogHandler": true,
       "enableSystemLogQuery": false,
       "shutdownGracePeriod": "0s",
       "shutdownGracePeriodCriticalPods": "0s",
       "enableProfilingHandler": true,
       "enableDebugFlagsHandler": true,
       "seccompDefault": false,
       "memoryThrottlingFactor": 0.9,
       "registerNode": true,
       "localStorageCapacityIsolation": true,
       "containerRuntimeEndpoint": "unix:///var/run/crio/crio.sock"
     }
   }
   ```

## Tiếp theo (What's next)

- Tìm hiểu thêm về cấu hình kubelet qua tài liệu tham khảo
  [`KubeletConfiguration`](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).
- Tìm hiểu thêm về việc hợp nhất cấu hình kubelet trong
  [tài liệu tham khảo](https://kubernetes.io/docs/reference/node/kubelet-config-directory-merging).

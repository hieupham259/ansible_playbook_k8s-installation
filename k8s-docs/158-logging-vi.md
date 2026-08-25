# Kiến trúc ghi log (Logging Architecture)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/logging/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 11](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability), bài 4/6 · Kiểm chứng
ở Lab 11a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài dài, nhưng phần lớn độ dài đến từ các manifest ví dụ lặp đi lặp lại cùng một Pod đếm số.
Xương sống chỉ có ba tầng: log **container** trên một node, log của **thành phần hệ thống** trên
node đó, và ba kiến trúc **cấp cluster** để gom log ra khỏi node. Đọc để nắm ba tầng và ranh giới
giữa chúng.

**Phải hiểu ở lần đọc này:**

- Vòng đời log mặc định **gắn chặt vào container, pod và node**: container restart thì kubelet
  giữ lại container cũ cùng log của nó, nhưng pod bị trục xuất là log đi theo. Đây chính là lý do
  tồn tại của ghi log cấp cluster — log cần nơi lưu trữ và vòng đời riêng, và Kubernetes **không**
  cung cấp backend lưu trữ nào cho việc đó.
- **kubelet chịu trách nhiệm xoay vòng log container**, qua `containerLogMaxSize` (mặc định
  `10Mi`) và `containerLogMaxFiles` (mặc định `5`), và chỉ thị cho container runtime qua CRI ghi
  vào đúng vị trí. Hệ quả trực tiếp: `kubectl logs` **chỉ đọc được file log mới nhất**.
- Vị trí log trên node Linux: kubelet và container runtime **không** chạy trong container nên ghi
  vào journald khi có systemd; scheduler, controller manager, API server chạy trong pod nên ghi
  file `.log` trong `/var/log`, bỏ qua cơ chế log mặc định của container. Log của pod nằm ở
  `/var/log/pods`.
- Ba kiến trúc cấp cluster và đánh đổi của chúng: **agent cấp node** (chạy dạng `DaemonSet`, một
  agent mỗi node, không đụng tới ứng dụng), **sidecar** (hai kiểu, xem dưới), và **đẩy thẳng từ
  ứng dụng** (nằm ngoài phạm vi Kubernetes).
- Phân biệt hai kiểu sidecar: sidecar **truyền luồng** đọc file rồi in ra `stdout` của chính nó
  nên `kubectl logs` vẫn dùng được; sidecar **chạy agent ghi log** thì log không do kubelet kiểm
  soát nên `kubectl logs` không thấy, và tốn tài nguyên đáng kể.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Các luồng log của container* — feature gate `PodLogsQuerySplitStreams`, `?stream=Stderr` | là tính năng alpha, phải bật feature gate mới có | giai đoạn 20 cấu hình lại cluster đang chạy |
| Mục *Windows* trong *Vị trí log* (`C:\var\logs`, `C:\var\log\pods`) | cluster lab chỉ có node Linux | giai đoạn 15 (node Windows) |
| `containerLogMaxWorkers`, `containerLogMonitorInterval`, `podLogsDir` | là tinh chỉnh cho cluster có khối lượng log rất lớn | giai đoạn 20 cấu hình lại cluster đang chạy |
| `kube-log-runner` | là công cụ chuyển hướng output khi không có shell hay systemd | bài [159](159-system-logs-vi.md) |
| Bốn manifest ví dụ (`counter-pod`, hai file log, ConfigMap fluentd, sidecar agent) | đọc để thấy hình dạng là đủ; chạy chúng là việc của lab | Lab 11a |

---

Log của ứng dụng có thể giúp bạn hiểu điều gì đang diễn ra bên trong ứng dụng của mình. Log
đặc biệt hữu ích cho việc gỡ lỗi (debug) và giám sát hoạt động của cluster. Hầu hết
các ứng dụng hiện đại đều có một cơ chế ghi log nào đó. Tương tự, các container engine
cũng được thiết kế để hỗ trợ ghi log. Phương pháp ghi log dễ nhất và được áp dụng rộng rãi nhất
cho các ứng dụng chạy trong container là ghi vào luồng đầu ra chuẩn (standard output) và luồng lỗi chuẩn (standard error).

Tuy nhiên, chức năng có sẵn do container engine hoặc runtime cung cấp thường
không đủ cho một giải pháp ghi log hoàn chỉnh.

Ví dụ, bạn có thể muốn truy cập log của ứng dụng khi một container bị crash,
một pod bị trục xuất (evicted), hoặc một node bị chết.

Trong một cluster, log nên có nơi lưu trữ và vòng đời riêng, độc lập với node,
pod hay container. Khái niệm này được gọi là
[ghi log cấp cluster (cluster-level logging)](#cluster-level-logging-architectures).

Các kiến trúc ghi log cấp cluster cần một backend riêng để lưu trữ, phân tích và
truy vấn log. Kubernetes không cung cấp giải pháp lưu trữ nguyên bản (native) cho dữ liệu log. Thay vào đó,
có rất nhiều giải pháp ghi log tích hợp được với Kubernetes. Các phần dưới đây
mô tả cách xử lý và lưu trữ log trên các node.

## Log của Pod và container (Pod and container logs) {#basic-logging-in-kubernetes}

Kubernetes thu thập log từ mỗi container trong một Pod đang chạy.

Ví dụ này sử dụng manifest cho một `Pod` với một container
ghi văn bản ra luồng đầu ra chuẩn, mỗi giây một lần.

[`debug/counter-pod.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/debug/counter-pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    args: [/bin/sh, -c,
            'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
```

Để chạy pod này, dùng lệnh sau:

```shell
kubectl apply -f https://k8s.io/examples/debug/counter-pod.yaml
```

Kết quả đầu ra là:

```console
pod/counter created
```

Để lấy log, dùng lệnh `kubectl logs` như sau:

```shell
kubectl logs counter
```

Kết quả đầu ra tương tự như:

```console
0: Fri Apr  1 11:42:23 UTC 2022
1: Fri Apr  1 11:42:24 UTC 2022
2: Fri Apr  1 11:42:25 UTC 2022
```

Bạn có thể dùng `kubectl logs --previous` để lấy log từ một phiên chạy trước đó của container.
Nếu pod của bạn có nhiều container, hãy chỉ định container mà bạn muốn truy cập log bằng cách
thêm tên container vào lệnh, kèm flag `-c`, như sau:

```shell
kubectl logs counter -c count
```

### Các luồng log của container (Container log streams)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [alpha]`

Ở dạng tính năng alpha, kubelet có thể tách riêng log từ hai luồng chuẩn mà
một container tạo ra: [đầu ra chuẩn](https://en.wikipedia.org/wiki/Standard_streams#Standard_output_(stdout))
và [lỗi chuẩn](https://en.wikipedia.org/wiki/Standard_streams#Standard_error_(stderr)).
Để dùng hành vi này, bạn phải bật [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`PodLogsQuerySplitStreams`.
Khi feature gate đó được bật, Kubernetes v1.36 cho phép truy cập các
luồng log này trực tiếp qua Pod API. Bạn có thể lấy một luồng cụ thể bằng cách chỉ định tên luồng (`Stdout` hoặc `Stderr`),
qua tham số truy vấn (query string) `stream`. Bạn phải có quyền đọc subresource `log` của Pod đó.

Để minh họa tính năng này, bạn có thể tạo một Pod ghi văn bản định kỳ ra cả luồng đầu ra chuẩn lẫn luồng lỗi chuẩn.

[`debug/counter-pod-err.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/debug/counter-pod-err.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter-err
spec:
  containers:
  - name: count
    image: busybox:1.28
    args: [/bin/sh, -c,
            'i=0; while true; do echo "$i: $(date)"; echo "$i: err" >&2 ; i=$((i+1)); sleep 1; done']
```

Để chạy pod này, dùng lệnh sau:

```shell
kubectl apply -f https://k8s.io/examples/debug/counter-pod-err.yaml
```

Để chỉ lấy luồng log stderr, bạn có thể chạy:

```shell
kubectl get --raw "/api/v1/namespaces/default/pods/counter-err/log?stream=Stderr"
```

Xem [tài liệu `kubectl logs`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#logs)
để biết thêm chi tiết.

### Cách node xử lý log của container (How nodes handle container logs)

![Ghi log ở cấp node](https://kubernetes.io/images/docs/user-guide/logging/logging-node-level.png)

Container runtime xử lý và chuyển hướng mọi output mà ứng dụng chạy trong container
ghi ra các luồng `stdout` và `stderr`.
Các container runtime khác nhau hiện thực việc này theo những cách khác nhau; tuy nhiên,
phần tích hợp với kubelet được chuẩn hóa dưới tên _định dạng log CRI (CRI logging format)_.

Mặc định, nếu một container khởi động lại, kubelet giữ lại một container đã kết thúc cùng với log của nó.
Nếu một pod bị trục xuất khỏi node, mọi container tương ứng cũng bị trục xuất, cùng với log của chúng.

Kubelet cung cấp log cho các client thông qua một tính năng đặc biệt của Kubernetes API.
Cách thông thường để truy cập là chạy `kubectl logs`.

### Xoay vòng log (Log rotation)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [stable]`

Kubelet chịu trách nhiệm xoay vòng (rotate) log của container và quản lý
cấu trúc thư mục ghi log.
Kubelet gửi thông tin này tới container runtime (qua CRI),
và runtime ghi log của container vào vị trí được chỉ định.

Bạn có thể cấu hình hai [thiết lập cấu hình](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/) của kubelet là
`containerLogMaxSize` (mặc định 10Mi) và `containerLogMaxFiles` (mặc định 5),
bằng [file cấu hình kubelet](224-kubelet-config-file-vi.md).
Các thiết lập này cho phép bạn cấu hình lần lượt kích thước tối đa cho mỗi file log và số lượng
file tối đa được phép cho mỗi container.

Để thực hiện xoay vòng log hiệu quả trong những cluster có khối lượng log do
workload sinh ra lớn, kubelet cũng cung cấp một cơ chế tinh chỉnh cách log được xoay vòng,
gồm số lượt xoay vòng log có thể thực hiện đồng thời và khoảng thời gian giữa các lần log được
giám sát và xoay vòng khi cần.
Bạn có thể cấu hình hai [thiết lập cấu hình](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/) của kubelet là
`containerLogMaxWorkers` và `containerLogMonitorInterval` bằng
[file cấu hình kubelet](224-kubelet-config-file-vi.md).

Khi bạn chạy [`kubectl logs`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#logs) như trong
ví dụ ghi log cơ bản, kubelet trên node sẽ xử lý yêu cầu và
đọc trực tiếp từ file log. Kubelet trả về nội dung của file log đó.

> **Ghi chú:**
> Chỉ nội dung của file log mới nhất là truy cập được qua `kubectl logs`.
>
> Ví dụ, nếu một Pod ghi 40 MiB log và kubelet xoay vòng log
> sau mỗi 10 MiB, thì chạy `kubectl logs` trả về nhiều nhất 10MiB dữ liệu.

## Log của các thành phần hệ thống (System component logs)

Có hai loại thành phần hệ thống: loại thường chạy trong container,
và loại tham gia trực tiếp vào việc chạy các container. Ví dụ:

* Kubelet và container runtime không chạy trong container. Kubelet chạy
  các container của bạn (được gom nhóm trong các pod)
* Kubernetes scheduler, controller manager và API server chạy bên trong các pod
  (thường là static Pod).
  Thành phần etcd chạy trong control plane, và phổ biến nhất cũng ở dạng static pod.
  Nếu cluster của bạn dùng kube-proxy, bạn thường chạy nó dưới dạng `DaemonSet`.

### Vị trí log (Log locations) {#log-location-node}

Cách kubelet và container runtime ghi log phụ thuộc vào hệ điều hành
mà node sử dụng:

#### Linux

Trên các node Linux dùng systemd, kubelet và container runtime mặc định ghi vào journald.
Bạn dùng `journalctl` để đọc systemd journal; ví dụ:
`journalctl -u kubelet`.

Nếu không có systemd, kubelet và container runtime ghi vào các file `.log` trong
thư mục `/var/log`. Nếu bạn muốn log được ghi ở nơi khác, bạn có thể chạy kubelet
một cách gián tiếp qua một công cụ trợ giúp là `kube-log-runner`, và dùng công cụ đó để chuyển hướng
log của kubelet vào một thư mục do bạn chọn.

Mặc định, kubelet chỉ thị cho container runtime của bạn ghi log vào các thư mục bên trong
`/var/log/pods`.

Để biết thêm thông tin về `kube-log-runner`, đọc [System Logs](159-system-logs-vi.md#klog).

#### Windows

Mặc định, kubelet ghi log vào các file trong thư mục `C:\var\logs`
(lưu ý rằng đây không phải là `C:\var\log`).

Mặc dù `C:\var\log` là vị trí mặc định của Kubernetes cho các log này, một số
công cụ triển khai cluster thiết lập các node Windows ghi log vào `C:\var\log\kubelet` thay thế.

Nếu bạn muốn log được ghi ở nơi khác, bạn có thể chạy kubelet
một cách gián tiếp qua một công cụ trợ giúp là `kube-log-runner`, và dùng công cụ đó để chuyển hướng
log của kubelet vào một thư mục do bạn chọn.

Tuy nhiên, mặc định, kubelet chỉ thị cho container runtime của bạn ghi log bên trong
thư mục `C:\var\log\pods`.

Để biết thêm thông tin về `kube-log-runner`, đọc [System Logs](159-system-logs-vi.md#klog).

Với các thành phần cluster Kubernetes chạy trong pod, chúng ghi vào các file bên trong
thư mục `/var/log`, bỏ qua cơ chế ghi log mặc định (các thành phần này
không ghi vào systemd journal). Bạn có thể dùng các cơ chế lưu trữ của Kubernetes
để ánh xạ lưu trữ bền vững (persistent storage) vào container chạy thành phần đó.

Kubelet cho phép thay đổi thư mục log của pod từ mặc định `/var/log/pods`
sang một đường dẫn tùy chỉnh. Việc điều chỉnh này có thể được thực hiện bằng cách cấu hình
tham số `podLogsDir` trong file cấu hình của kubelet.

> **Thận trọng:**
> Điều quan trọng cần lưu ý là vị trí mặc định `/var/log/pods` đã được sử dụng trong
> một thời gian dài và một số tiến trình có thể ngầm định giả định đường dẫn này.
> Do đó, việc thay đổi tham số này phải được tiếp cận thận trọng và bạn tự chịu rủi ro.
>
> Một lưu ý khác cần nhớ là kubelet chỉ hỗ trợ khi vị trí này nằm trên cùng
> đĩa với `/var`. Ngược lại, nếu log nằm trên một filesystem tách biệt với `/var`,
> thì kubelet sẽ không theo dõi mức sử dụng của filesystem đó, có khả năng dẫn tới sự cố nếu
> nó bị đầy.

Để biết chi tiết về etcd và log của nó, xem [tài liệu etcd](https://etcd.io/docs/).
Một lần nữa, bạn có thể dùng các cơ chế lưu trữ của Kubernetes để ánh xạ lưu trữ bền vững vào
container chạy thành phần đó.

> **Ghi chú:**
> Nếu bạn triển khai các thành phần cluster Kubernetes (chẳng hạn scheduler) ghi log vào
> một volume được chia sẻ từ node cha, bạn cần cân nhắc và đảm bảo rằng những
> log đó được xoay vòng. **Kubernetes không quản lý việc xoay vòng log đó**.
>
> Hệ điều hành của bạn có thể tự động thực hiện một phần việc xoay vòng log - ví dụ,
> nếu bạn chia sẻ thư mục `/var/log` vào một static Pod của một thành phần, cơ chế xoay vòng log
> cấp node xử lý một file trong thư mục đó giống hệt như một file do bất kỳ thành phần nào
> bên ngoài Kubernetes ghi ra.
>
> Một số công cụ triển khai đã tính đến việc xoay vòng log đó và tự động hóa nó; số khác để việc này
> lại như trách nhiệm của bạn.

## Các kiến trúc ghi log cấp cluster (Cluster-level logging architectures) {#cluster-level-logging-architectures}

Mặc dù Kubernetes không cung cấp giải pháp nguyên bản cho việc ghi log cấp cluster, có
một số cách tiếp cận phổ biến bạn có thể cân nhắc. Dưới đây là vài lựa chọn:

* Dùng một agent ghi log cấp node chạy trên mọi node.
* Thêm một sidecar container chuyên trách việc ghi log vào pod của ứng dụng.
* Đẩy log trực tiếp tới một backend từ bên trong ứng dụng.

### Sử dụng agent ghi log cấp node (Using a node logging agent)

![Sử dụng agent ghi log cấp node](https://kubernetes.io/images/docs/user-guide/logging/logging-with-node-agent.png)

Bạn có thể hiện thực việc ghi log cấp cluster bằng cách đưa một _agent ghi log cấp node (node-level logging agent)_ vào mỗi node.
Agent ghi log là một công cụ chuyên trách có nhiệm vụ phơi bày (expose) log hoặc đẩy log tới một backend.
Thông thường, agent ghi log là một container có quyền truy cập vào thư mục chứa các file log từ tất cả
các container ứng dụng trên node đó.

Vì agent ghi log phải chạy trên mọi node, khuyến nghị là chạy agent
dưới dạng một `DaemonSet`.

Ghi log cấp node chỉ tạo một agent trên mỗi node và không đòi hỏi bất kỳ thay đổi nào đối với
các ứng dụng đang chạy trên node.

Các container ghi ra stdout và stderr, nhưng không theo một định dạng thống nhất nào. Một agent cấp node thu thập
những log này và chuyển tiếp chúng đi để tổng hợp (aggregation).

### Sử dụng sidecar container cùng agent ghi log (Using a sidecar container with the logging agent) {#sidecar-container-with-logging-agent}

Bạn có thể dùng sidecar container theo một trong các cách sau:

* Sidecar container truyền (stream) log của ứng dụng ra `stdout` của chính nó.
* Sidecar container chạy một agent ghi log, được cấu hình để thu nhận log
  từ một container ứng dụng.

#### Sidecar container truyền luồng log (Streaming sidecar container)

![Sidecar container với một container truyền luồng log](https://kubernetes.io/images/docs/user-guide/logging/logging-with-streaming-sidecar.png)

Bằng cách để các sidecar container ghi vào chính luồng `stdout` và `stderr`
của chúng, bạn có thể tận dụng kubelet và agent ghi log
vốn đã chạy trên mỗi node. Các sidecar container đọc log từ một file, một socket,
hoặc journald. Mỗi sidecar container in log ra luồng `stdout` hoặc `stderr` của riêng nó.

Cách tiếp cận này cho phép bạn tách riêng nhiều luồng log từ các
phần khác nhau của ứng dụng, trong đó một số phần có thể không hỗ trợ
ghi ra `stdout` hoặc `stderr`. Phần logic chuyển hướng log
là rất nhỏ, nên chi phí phát sinh không đáng kể. Ngoài ra, vì
`stdout` và `stderr` được kubelet xử lý, bạn có thể dùng các công cụ có sẵn
như `kubectl logs`.

Ví dụ, một pod chạy một container duy nhất, và container đó
ghi vào hai file log khác nhau với hai định dạng khác nhau. Đây là
manifest cho Pod đó:

[`admin/logging/two-files-counter-pod.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/admin/logging/two-files-counter-pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```

Không khuyến nghị ghi các bản ghi log có định dạng khác nhau vào cùng một luồng
log, ngay cả khi bạn xoay xở chuyển hướng được cả hai thành phần vào luồng `stdout` của
container. Thay vào đó, bạn có thể tạo hai sidecar container. Mỗi sidecar
container có thể tail một file log cụ thể từ một volume dùng chung rồi chuyển hướng
log đó vào luồng `stdout` của riêng nó.

Đây là manifest cho một pod có hai sidecar container:

[`admin/logging/two-files-counter-pod-streaming-sidecar.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/admin/logging/two-files-counter-pod-streaming-sidecar.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-1
    image: busybox:1.28
    args: [/bin/sh, -c, 'tail -n+1 -F /var/log/1.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-2
    image: busybox:1.28
    args: [/bin/sh, -c, 'tail -n+1 -F /var/log/2.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```

Bây giờ khi chạy pod này, bạn có thể truy cập từng luồng log một cách riêng biệt bằng
các lệnh sau:

```shell
kubectl logs counter count-log-1
```

Kết quả đầu ra tương tự như:

```console
0: Fri Apr  1 11:42:26 UTC 2022
1: Fri Apr  1 11:42:27 UTC 2022
2: Fri Apr  1 11:42:28 UTC 2022
...
```

```shell
kubectl logs counter count-log-2
```

Kết quả đầu ra tương tự như:

```console
Fri Apr  1 11:42:29 UTC 2022 INFO 0
Fri Apr  1 11:42:30 UTC 2022 INFO 0
Fri Apr  1 11:42:31 UTC 2022 INFO 0
...
```

Nếu bạn đã cài một agent cấp node trong cluster, agent đó sẽ tự động thu nhận những luồng
log này mà không cần cấu hình gì thêm. Nếu muốn, bạn có thể cấu hình
agent để phân tích cú pháp (parse) các dòng log tùy theo container nguồn.

Ngay cả với những Pod chỉ sử dụng ít CPU và bộ nhớ (cỡ vài millicore
cho cpu và cỡ vài megabyte cho bộ nhớ), việc ghi log vào một file rồi
truyền chúng ra `stdout` có thể tăng gấp đôi dung lượng lưu trữ bạn cần trên node.
Nếu bạn có một ứng dụng ghi vào một file duy nhất, khuyến nghị là đặt
`/dev/stdout` làm đích ghi thay vì hiện thực cách tiếp cận sidecar
container truyền luồng.

Sidecar container cũng có thể được dùng để xoay vòng các file log mà bản thân
ứng dụng không thể tự xoay vòng. Một ví dụ cho cách tiếp cận này là một container nhỏ chạy
`logrotate` định kỳ.
Tuy nhiên, cách đơn giản hơn là dùng trực tiếp `stdout` và `stderr`, và
để các chính sách xoay vòng và lưu giữ (retention) lại cho kubelet.

#### Sidecar container với agent ghi log (Sidecar container with a logging agent)

![Sidecar container với một agent ghi log](https://kubernetes.io/images/docs/user-guide/logging/logging-with-sidecar-agent.png)

Nếu agent ghi log cấp node không đủ linh hoạt cho tình huống của bạn, bạn
có thể tạo một sidecar container với một agent ghi log riêng mà bạn đã
cấu hình chuyên biệt để chạy cùng ứng dụng của mình.

> **Ghi chú:**
> Dùng agent ghi log trong sidecar container có thể dẫn tới
> mức tiêu thụ tài nguyên đáng kể. Hơn nữa, bạn sẽ không truy cập được
> những log đó bằng `kubectl logs` vì chúng không do
> kubelet kiểm soát.

Đây là hai manifest ví dụ mà bạn có thể dùng để hiện thực một sidecar container với agent ghi log.
Manifest thứ nhất chứa một [`ConfigMap`](275-configure-pod-configmap-vi.md)
để cấu hình fluentd.

[`admin/logging/fluentd-sidecar-config.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/admin/logging/fluentd-sidecar-config.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluentd.conf: |
    <source>
      type tail
      format none
      path /var/log/1.log
      pos_file /var/log/1.log.pos
      tag count.format1
    </source>

    <source>
      type tail
      format none
      path /var/log/2.log
      pos_file /var/log/2.log.pos
      tag count.format2
    </source>

    <match **>
      type google_cloud
    </match>
```

> **Ghi chú:**
> Trong các cấu hình mẫu, bạn có thể thay fluentd bằng bất kỳ agent ghi log nào, đọc
> từ bất kỳ nguồn nào bên trong một container ứng dụng.

Manifest thứ hai mô tả một pod có sidecar container chạy fluentd.
Pod này mount một volume để fluentd có thể lấy dữ liệu cấu hình của nó.

[`admin/logging/two-files-counter-pod-agent-sidecar.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/admin/logging/two-files-counter-pod-agent-sidecar.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-agent
    image: registry.k8s.io/fluentd-gcp:1.30
    env:
    - name: FLUENTD_ARGS
      value: -c /etc/fluentd-config/fluentd.conf
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: config-volume
      mountPath: /etc/fluentd-config
  volumes:
  - name: varlog
    emptyDir: {}
  - name: config-volume
    configMap:
      name: fluentd-config
```

### Xuất log trực tiếp từ ứng dụng (Exposing logs directly from the application)

![Xuất log trực tiếp từ ứng dụng](https://kubernetes.io/images/docs/user-guide/logging/logging-from-application.png)

Việc ghi log cấp cluster theo cách phơi bày hoặc đẩy log trực tiếp từ mỗi ứng dụng nằm ngoài phạm vi
của Kubernetes.

## Tiếp theo (What's next)

* Đọc về [log hệ thống của Kubernetes](159-system-logs-vi.md)
* Tìm hiểu về [Traces cho các thành phần hệ thống Kubernetes](161-system-traces-vi.md)
* Tìm hiểu cách [tùy chỉnh thông điệp kết thúc (termination message)](303-determine-reason-pod-failure-vi.md#customizing-the-termination-message)
  mà Kubernetes ghi lại khi một Pod thất bại

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 11:

1. Một Pod trên `k8s-worker2` đã ghi 40 MiB log kể từ lúc khởi động, kubelet giữ nguyên cấu hình
   mặc định. `kubectl logs` trả về nhiều nhất bao nhiêu dữ liệu, và vì sao?
2. **Câu bẫy.** Hai kiểu sidecar ghi log — kiểu truyền luồng và kiểu chạy agent — kiểu nào vẫn
   xem được bằng `kubectl logs`, kiểu nào không? Điều gì quyết định sự khác biệt đó?
3. `k8s-worker1` chết hẳn và không bật lại được. Log của các Pod từng chạy trên đó còn lấy được
   không nếu cluster chỉ dựa vào cơ chế mặc định? Cần thêm gì để còn?
4. Trên node Ubuntu 24.04 có systemd, bạn tìm log của kubelet ở đâu và log của kube-scheduler ở
   đâu? Vì sao hai thành phần của cùng một cluster lại nằm hai chỗ khác nhau?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nhiều nhất **10 MiB** — đúng bằng `containerLogMaxSize` mặc định. Bài nêu thẳng ví dụ này:
   chỉ **nội dung của file log mới nhất** là truy cập được qua `kubectl logs`, nên khi kubelet đã
   xoay vòng sau mỗi 10 MiB thì phần cũ hơn tuy còn trên đĩa (tối đa `containerLogMaxFiles` = 5
   file) nhưng `kubectl logs` không trả về.
2. Sidecar **truyền luồng** thì `kubectl logs` **vẫn dùng được**; sidecar **chạy agent ghi log**
   thì **không**. Yếu tố quyết định là log có đi qua `stdout`/`stderr` của container hay không:
   sidecar truyền luồng tail file rồi in ra `stdout` của chính nó, nên rơi vào đường đi thông
   thường do kubelet xử lý. Sidecar chạy agent thì đọc file rồi đẩy thẳng ra backend, log **không
   do kubelet kiểm soát** nên `kubectl logs` không thấy — bài ghi rõ điều này trong phần ghi chú,
   kèm cảnh báo tiêu thụ tài nguyên đáng kể.
3. **Không.** Log container sống cùng node: khi pod bị trục xuất thì container và log của chúng
   đi theo, và mất node thì mất luôn file log. Muốn giữ được, phải có **ghi log cấp cluster** —
   log phải có nơi lưu trữ và vòng đời **độc lập với node, pod và container**, tức một backend
   riêng cộng với cơ chế đẩy log ra khỏi node, phổ biến nhất là **agent cấp node chạy dạng
   `DaemonSet`**.
4. Log kubelet nằm trong **journald** (đọc bằng `journalctl -u kubelet`); log kube-scheduler nằm
   ở **file `.log` trong `/var/log`**. Lý do là hai loại thành phần khác nhau: kubelet và
   container runtime **không chạy trong container** nên dùng cơ chế log của hệ điều hành, còn
   scheduler, controller manager và API server **chạy bên trong pod** (thường là static Pod) và
   ghi thẳng vào file trong `/var/log`, bỏ qua cơ chế ghi log mặc định của container.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

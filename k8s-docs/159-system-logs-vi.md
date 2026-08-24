# Log hệ thống (System Logs)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/system-logs/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 11](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability), bài 5/6 · Kiểm chứng
ở Lab 11a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài [158](158-logging-vi.md) nói về log của **workload**; bài này nói về log của **chính
Kubernetes**. Một phần đáng kể nội dung viết cho người phát triển thành phần Kubernetes chứ không
cho người vận hành — nhận ra ranh giới đó rồi thì bài ngắn lại rất nhiều.

**Phải hiểu ở lần đọc này:**

- **Nội dung log không thuộc phạm vi bảo đảm ổn định của Kubernetes API.** Cờ dòng lệnh thì có
  cam kết, còn từng dòng log và định dạng của chúng có thể đổi giữa các bản phát hành — nên đừng
  xây cảnh báo trên một chuỗi ký tự cụ thể.
- klog luôn ghi output **ra stderr**, bất kể định dạng. Loạt cờ `--log-file`, `--logtostderr`,
  `--log-dir`… đã bị loại bỏ dần từ v1.23 và **bị xóa ở v1.26**; việc chuyển hướng là trách nhiệm
  của bên gọi — POSIX shell hoặc systemd.
- Khi không có shell và không có systemd (container distroless, Windows service), dùng
  `kube-log-runner` làm lớp bọc; bảng trong bài ánh xạ từng cách dùng sang chuyển hướng shell
  tương đương.
- Cờ `-v` điều khiển mức chi tiết: **tăng giá trị thì ghi thêm các sự kiện ngày càng ít nghiêm
  trọng hơn**, `-v=0` chỉ ghi sự kiện nghiêm trọng.
- Vị trí log lặp lại đúng ranh giới của bài 158: kubelet và container runtime ghi journald khi có
  systemd, thành phần chạy trong container ghi file `.log` ở `/var/log`, và log trong `/var/log`
  vẫn cần được xoay vòng (`logrotate`, thường theo ngày hoặc khi vượt 100MB).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Ghi log có cấu trúc* và *Ghi log theo ngữ cảnh* (`WithValues`, `WithName`, `ContextualLogging`) | viết cho người phát triển thành phần, không phải người vận hành | không cần |
| *Định dạng log JSON* — `--logging-format=json` và các khóa `ts`, `v`, `err`, `msg` | chỉ có giá trị khi đã có pipeline phân tích log để đẩy vào | CP8 giám sát và cảnh báo |
| *Truy vấn log* — `enableSystemLogHandler`, `/proxy/logs/?query=` | phải bật tùy chọn kubelet trên node đích và cấp quyền `nodes/proxy` | CP9 xử lý sự cố |
| Bảng tùy chọn `boot`, `pattern`, `sinceTime`, `untilTime`, `tailLines` | phụ thuộc mục *Truy vấn log* ở trên | CP9 xử lý sự cố |

---

Log của các thành phần hệ thống ghi lại các sự kiện xảy ra trong cluster, rất hữu ích cho việc
gỡ lỗi (debug). Bạn có thể cấu hình mức độ chi tiết (verbosity) của log để xem nhiều hay ít
chi tiết hơn. Log có thể ở mức thô như chỉ hiển thị lỗi bên trong một thành phần, hoặc ở mức
tinh như hiển thị từng bước diễn biến của các sự kiện (như log truy cập HTTP, thay đổi trạng
thái của pod, hành động của controller, hay quyết định của scheduler).

> **Cảnh báo:**
> Khác với các cờ (flag) dòng lệnh được mô tả ở đây, bản thân *nội dung log* *không* thuộc
> phạm vi bảo đảm ổn định của Kubernetes API: từng dòng log riêng lẻ và định dạng của chúng
> có thể thay đổi giữa các bản phát hành!

## Klog

klog là thư viện ghi log của Kubernetes. [klog](https://github.com/kubernetes/klog)
sinh ra các thông điệp log cho các thành phần hệ thống của Kubernetes.

Kubernetes đang trong quá trình đơn giản hóa việc ghi log trong các thành phần của mình.
Các cờ dòng lệnh klog sau đây
[đã bị loại bỏ dần (deprecated)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/2845-deprecate-klog-specific-flags-in-k8s-components)
từ Kubernetes v1.23 và bị xóa trong Kubernetes v1.26:

- `--add-dir-header`
- `--alsologtostderr`
- `--log-backtrace-at`
- `--log-dir`
- `--log-file`
- `--log-file-max-size`
- `--logtostderr`
- `--one-output`
- `--skip-headers`
- `--skip-log-headers`
- `--stderrthreshold`

Output sẽ luôn được ghi ra stderr, bất kể định dạng output là gì. Việc chuyển hướng output
được kỳ vọng do thành phần gọi thành phần Kubernetes đảm nhiệm. Đó có thể là một POSIX
shell hoặc một công cụ như systemd.

Trong một số trường hợp, ví dụ container distroless hoặc một system service trên Windows,
các tùy chọn đó không khả dụng. Khi đó có thể dùng binary
[`kube-log-runner`](https://github.com/kubernetes/kubernetes/blob/d2a8a81639fcff8d1221b900f66d28361a170654/staging/src/k8s.io/component-base/logs/kube-log-runner/README.md)
làm lớp bọc (wrapper) quanh một thành phần Kubernetes để chuyển hướng output. Một binary
dựng sẵn được kèm trong nhiều base image của Kubernetes dưới tên truyền thống là
`/go-runner`, và dưới tên `kube-log-runner` trong các gói phát hành server và node.

Bảng sau cho thấy các cách gọi `kube-log-runner` tương ứng với chuyển hướng shell như thế nào:

| Cách dùng                                       | POSIX shell (như bash)     | `kube-log-runner <options> <cmd>`                           |
| ------------------------------------------------|----------------------------|-------------------------------------------------------------|
| Gộp stderr và stdout, ghi ra stdout             | `2>&1`                     | `kube-log-runner` (hành vi mặc định)                        |
| Chuyển hướng cả hai vào file log                | `1>>/tmp/log 2>&1`         | `kube-log-runner -log-file=/tmp/log`                        |
| Sao chép vào file log và đồng thời ra stdout    | `2>&1 \| tee -a /tmp/log`  | `kube-log-runner -log-file=/tmp/log -also-stdout`           |
| Chỉ chuyển hướng stdout vào file log            | `>/tmp/log`                | `kube-log-runner -log-file=/tmp/log -redirect-stderr=false` |

### Output của klog (Klog output)

Một ví dụ về định dạng gốc truyền thống của klog:

```
I1025 00:15:15.525108       1 httplog.go:79] GET /api/v1/namespaces/kube-system/pods/metrics-server-v0.3.1-57c75779f-9p8wg: (1.512ms) 200 [pod_nanny/v0.0.0 (linux/amd64) kubernetes/$Format 10.56.1.19:51756]
```

Chuỗi thông điệp có thể chứa ký tự xuống dòng:

```
I1025 00:15:15.525108       1 example.go:79] This is a message
which has a line break.
```

### Ghi log có cấu trúc (Structured Logging)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [beta]`

> **Cảnh báo:**
> Việc chuyển đổi sang thông điệp log có cấu trúc là một quá trình đang diễn ra. Không phải
> tất cả thông điệp log đều có cấu trúc trong phiên bản này. Khi phân tích (parse) file log,
> bạn cũng phải xử lý cả những thông điệp log không có cấu trúc.
>
> Định dạng log và cách tuần tự hóa (serialize) giá trị có thể thay đổi.

Ghi log có cấu trúc đưa vào một cấu trúc thống nhất cho các thông điệp log, cho phép trích
xuất thông tin bằng chương trình. Bạn có thể lưu trữ và xử lý log có cấu trúc với ít công
sức và chi phí hơn. Đoạn mã sinh ra thông điệp log sẽ quyết định nó dùng output klog không
cấu trúc truyền thống hay ghi log có cấu trúc.

Định dạng mặc định của thông điệp log có cấu trúc là dạng văn bản, với định dạng tương
thích ngược với klog truyền thống:

```
<klog header> "<message>" <key1>="<value1>" <key2>="<value2>" ...
```

Ví dụ:

```
I1025 00:15:15.525108       1 controller_utils.go:116] "Pod status updated" pod="kube-system/kubedns" status="ready"
```

Chuỗi được đặt trong dấu nháy kép. Các giá trị khác được định dạng bằng
[`%+v`](https://pkg.go.dev/fmt#hdr-Printing), điều này có thể khiến thông điệp log tiếp tục
sang dòng kế tiếp [tùy vào dữ liệu](https://github.com/kubernetes/kubernetes/issues/106428).

```
I1025 00:15:15.525108       1 example.go:116] "Example" data="This is text with a line break\nand \"quotation marks\"." someInt=1 someFloat=0.1 someStruct={StringField: First line,
second line.}
```

### Ghi log theo ngữ cảnh (Contextual Logging)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.30 [beta]`

Ghi log theo ngữ cảnh được xây dựng trên nền ghi log có cấu trúc. Nó chủ yếu liên quan đến
cách nhà phát triển sử dụng các lời gọi ghi log: mã nguồn dựa trên khái niệm này linh hoạt
hơn và hỗ trợ thêm các trường hợp sử dụng như được mô tả trong
[KEP Contextual Logging](https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/3077-contextual-logging).

Nếu nhà phát triển dùng thêm các hàm như `WithValues` hoặc `WithName` trong thành phần của
họ, thì các mục log sẽ chứa thông tin bổ sung được bên gọi truyền vào các hàm.

Với Kubernetes 1.36, tính năng này nằm sau
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`ContextualLogging` và được bật mặc định. Hạ tầng cho tính năng này đã được thêm vào từ
phiên bản 1.24 mà không sửa đổi các thành phần. Lệnh
[`component-base/logs/example`](https://github.com/kubernetes/kubernetes/blob/v1.24.0-beta.0/staging/src/k8s.io/component-base/logs/example/cmd/logger.go)
minh họa cách dùng các lời gọi ghi log mới và cách một thành phần hỗ trợ ghi log theo ngữ
cảnh hoạt động.

```console
$ cd $GOPATH/src/k8s.io/kubernetes/staging/src/k8s.io/component-base/logs/example/cmd/
$ go run . --help
...
      --feature-gates mapStringBool  A set of key=value pairs that describe feature gates for alpha/experimental features. Options are:
                                     AllAlpha=true|false (ALPHA - default=false)
                                     AllBeta=true|false (BETA - default=false)
                                     ContextualLogging=true|false (BETA - default=true)
$ go run . --feature-gates ContextualLogging=true
...
I0222 15:13:31.645988  197901 example.go:54] "runtime" logger="example.myname" foo="bar" duration="1m0s"
I0222 15:13:31.646007  197901 example.go:55] "another runtime" logger="example" foo="bar" duration="1h0m0s" duration="1m0s"
```

Khóa `logger` và `foo="bar"` được thêm bởi bên gọi của hàm ghi thông điệp `runtime` và giá
trị `duration="1m0s"`, mà không cần sửa đổi hàm đó.

Khi ghi log theo ngữ cảnh bị tắt, `WithValues` và `WithName` không làm gì cả và các lời gọi
log đi qua logger klog toàn cục. Do đó, thông tin bổ sung này không còn xuất hiện trong
output của log nữa:

```console
$ go run . --feature-gates ContextualLogging=false
...
I0222 15:14:40.497333  198174 example.go:54] "runtime" duration="1m0s"
I0222 15:14:40.497346  198174 example.go:55] "another runtime" duration="1h0m0s" duration="1m0s"
```

### Định dạng log JSON (JSON log format)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.19 [alpha]`

> **Cảnh báo:**
> Output JSON không hỗ trợ nhiều cờ klog tiêu chuẩn. Để xem danh sách các cờ klog không được
> hỗ trợ, xem [Tài liệu tham khảo công cụ dòng lệnh](https://kubernetes.io/docs/reference/command-line-tools-reference/).
>
> Không phải mọi log đều được bảo đảm ghi ở định dạng JSON (ví dụ trong quá trình khởi động
> tiến trình). Nếu bạn định phân tích log, hãy bảo đảm bạn cũng xử lý được cả những dòng log
> không phải JSON.
>
> Tên trường và cách tuần tự hóa JSON có thể thay đổi.

Cờ `--logging-format=json` đổi định dạng log từ định dạng gốc của klog sang định dạng JSON.
Ví dụ về định dạng log JSON (đã được in đẹp):

```json
{
   "ts": 1580306777.04728,
   "v": 4,
   "msg": "Pod status updated",
   "pod":{
      "name": "nginx-1",
      "namespace": "default"
   },
   "status": "ready"
}
```

Các khóa có ý nghĩa đặc biệt:

* `ts` - dấu thời gian (timestamp) dạng Unix time (bắt buộc, float)
* `v` - mức chi tiết (verbosity) (chỉ có ở thông điệp info, không có ở thông điệp lỗi, int)
* `err` - chuỗi lỗi (tùy chọn, string)
* `msg` - thông điệp (bắt buộc, string)

Danh sách các thành phần hiện hỗ trợ định dạng JSON:

* kube-controller-manager
* kube-apiserver
* kube-scheduler
* kubelet

### Mức chi tiết của log (Log verbosity level)

Cờ `-v` điều khiển mức chi tiết của log. Tăng giá trị sẽ tăng số lượng sự kiện được ghi log.
Giảm giá trị sẽ giảm số lượng sự kiện được ghi log. Tăng mức chi tiết sẽ ghi lại các sự kiện
ngày càng ít nghiêm trọng hơn. Mức chi tiết bằng 0 chỉ ghi các sự kiện nghiêm trọng (critical).

### Vị trí log (Log location)

Có hai loại thành phần hệ thống: loại chạy trong container và loại không chạy trong
container. Ví dụ:

* Kubernetes scheduler và kube-proxy chạy trong container.
* kubelet và container runtime không chạy trong container.

Trên các máy có systemd, kubelet và container runtime ghi log vào journald. Nếu không có
systemd, chúng ghi vào các file `.log` trong thư mục `/var/log`.
Các thành phần hệ thống bên trong container luôn ghi vào các file `.log` trong thư mục
`/var/log`, bỏ qua cơ chế ghi log mặc định.
Tương tự như log của container, bạn nên xoay vòng (rotate) log của các thành phần hệ thống
trong thư mục `/var/log`.
Trong các cluster Kubernetes được tạo bởi script `kube-up.sh`, việc xoay vòng log được cấu
hình bởi công cụ `logrotate`. Công cụ `logrotate` xoay vòng log hằng ngày, hoặc khi kích
thước log vượt quá 100MB.

## Truy vấn log (Log query)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Tính năng Log Query (truy vấn log) có thể giúp gỡ lỗi trên cả các node Linux lẫn Windows.
Được giới thiệu trong Kubernetes v1.27, tính năng này cho phép xem log của các service đang
chạy trên node. Để dùng tính năng này, hãy bảo đảm hai tùy chọn cấu hình kubelet
`enableSystemLogHandler` và `enableSystemLogQuery` đều được đặt là _true_ trên node đích.

Trong Kubernetes v1.36, tính năng này đã lên mức ổn định (stable) và
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`NodeLogQuery` giờ đây bị khóa ở giá trị _true_, do đó feature gate được bật mặc định, và
`enableSystemLogHandler` trở thành tùy chọn duy nhất cần thiết để bật hoặc tắt tính năng
Log Query.

`enableSystemLogHandler` mặc định là _false_ và được khuyến nghị giữ ở trạng thái tắt trừ
khi bạn đang chủ động gỡ lỗi.

> **Cảnh báo:**
> Việc cấp quyền cho `nodes/proxy` (kể cả chỉ quyền **get**) cũng đồng thời cho phép truy
> cập các API kubelet rất mạnh, có thể bị lợi dụng để thực thi lệnh trong bất kỳ container
> nào đang chạy trên node, vì vậy hãy thận trọng trong cách bạn quản lý các quyền này.
> Xem [Xác thực/ủy quyền kubelet](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/#get-nodes-proxy-warning)
> để biết thêm thông tin.

Trên Linux, giả định là log của service khả dụng qua _journald_. Trên Windows, giả định là
log của service khả dụng trong application log provider. Trên cả hai hệ điều hành, log cũng
khả dụng bằng cách đọc các file trong `/var/log/`.

Miễn là bạn được phép tương tác với các đối tượng Node, bạn có thể thử tính năng này trên
tất cả các node của mình hoặc chỉ trên một tập con. Dưới đây là một ví dụ lấy log của
service kubelet từ một node:

```shell
# Lấy log kubelet từ node có tên node-1.example
kubectl get --raw "/api/v1/nodes/node-1.example/proxy/logs/?query=kubelet"
```

Bạn cũng có thể lấy các file, miễn là các file đó nằm trong thư mục mà kubelet cho phép lấy
log. Ví dụ, bạn có thể lấy một file log từ `/var/log` trên một node Linux:

```shell
kubectl get --raw "/api/v1/nodes/<insert-node-name-here>/proxy/logs/?query=/<insert-log-file-name-here>"
```

kubelet dùng phương pháp suy đoán (heuristics) để truy xuất log. Điều này hữu ích khi bạn
không biết một system service nào đó đang ghi log vào trình ghi log gốc của hệ điều hành
như journald hay vào một file log trong `/var/log/`. Phương pháp suy đoán trước tiên kiểm
tra trình ghi log gốc, và nếu không khả dụng, sẽ thử truy xuất log đầu tiên từ
`/var/log/<servicename>` hoặc `/var/log/<servicename>.log` hoặc
`/var/log/<servicename>/<servicename>.log`.

Danh sách đầy đủ các tùy chọn có thể sử dụng:

| Tùy chọn    | Mô tả                                                                                                |
|-------------|------------------------------------------------------------------------------------------------------|
| `boot`      | boot hiển thị các thông điệp từ một lần khởi động (boot) cụ thể của hệ thống                          |
| `pattern`   | pattern lọc các mục log theo biểu thức chính quy tương thích PERL được cung cấp                       |
| `query`     | query chỉ định (các) service hoặc file cần trả về log (bắt buộc)                                      |
| `sinceTime` | dấu thời gian [RFC3339](https://www.rfc-editor.org/rfc/rfc3339) bắt đầu hiển thị log (bao gồm mốc này) |
| `untilTime` | dấu thời gian [RFC3339](https://www.rfc-editor.org/rfc/rfc3339) kết thúc hiển thị log (bao gồm mốc này) |
| `tailLines` | chỉ định số dòng cuối của log cần lấy; mặc định là lấy toàn bộ log                                    |

Ví dụ về một truy vấn phức tạp hơn:

```shell
# Lấy log kubelet từ node có tên node-1.example mà có chứa từ "error"
kubectl get --raw "/api/v1/nodes/node-1.example/proxy/logs/?query=kubelet&pattern=error"
```

## Tiếp theo (What's next)

* Đọc về [Kiến trúc ghi log của Kubernetes](158-logging-vi.md)
* Đọc về [Ghi log có cấu trúc (Structured Logging)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/1602-structured-logging)
* Đọc về [Ghi log theo ngữ cảnh (Contextual Logging)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/3077-contextual-logging)
* Đọc về [việc loại bỏ dần các cờ klog](https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/2845-deprecate-klog-specific-flags-in-k8s-components)
* Đọc về [Quy ước về mức nghiêm trọng của log](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-instrumentation/logging.md)
* Đọc về [Log Query](https://kep.k8s.io/2258)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 11:

1. Trên `k8s-worker1` (Ubuntu 24.04, có systemd), bạn muốn xem kubelet đang báo gì. Đọc ở đâu, và
   vì sao tìm file `/var/log/kubelet.log` là hướng sai?
2. **Câu bẫy.** Bạn muốn kubelet ghi thẳng log vào `/tmp/kubelet.log` nên định thêm cờ
   `--log-file=/tmp/kubelet.log`. Cách này còn dùng được không? Nếu không thì có những đường nào
   khác?
3. Tăng `-v` từ `2` lên `6` thì log nhiều lên hay ít đi, và phần thêm vào là loại sự kiện nào?
4. Vì sao không nên viết một cảnh báo bằng cách khớp đúng một chuỗi ký tự trong dòng log của
   kube-controller-manager?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Đọc trong **systemd journal**, ví dụ `journalctl -u kubelet`. Bài nói rõ: trên máy có systemd,
   **kubelet và container runtime ghi log vào journald**; chỉ khi *không có* systemd thì chúng
   mới ghi vào các file `.log` trong `/var/log`. Node lab có systemd, nên không có file
   `/var/log/kubelet.log` để tìm.
2. **Không.** Cờ `--log-file` nằm trong danh sách các cờ klog đã bị loại bỏ dần từ v1.23 và **bị
   xóa trong v1.26**. Bài nêu nguyên tắc thay thế: output **luôn được ghi ra stderr**, và việc
   chuyển hướng được kỳ vọng do **thành phần gọi** đảm nhiệm — một POSIX shell hoặc systemd. Khi
   cả hai đều không có, dùng **`kube-log-runner -log-file=/tmp/log`**, đúng dòng tương ứng với
   `1>>/tmp/log 2>&1` trong bảng của bài. Chỗ dễ nhầm là thấy cờ này trong tài liệu cũ hoặc trong
   trí nhớ mà không để ý nó đã bị xóa.
3. **Nhiều lên.** Cờ `-v` điều khiển mức chi tiết: tăng giá trị thì tăng số lượng sự kiện được
   ghi, và phần thêm vào là **các sự kiện ngày càng ít nghiêm trọng hơn** — mức 0 chỉ ghi sự kiện
   nghiêm trọng, mức cao hơn ghi tới từng bước diễn biến như log truy cập HTTP, thay đổi trạng
   thái pod, hành động của controller hay quyết định của scheduler.
4. Vì **bản thân nội dung log không nằm trong phạm vi bảo đảm ổn định của Kubernetes API**: bài
   cảnh báo rằng khác với các cờ dòng lệnh, từng dòng log riêng lẻ và định dạng của chúng **có
   thể thay đổi giữa các bản phát hành**. Một cảnh báo khớp chuỗi sẽ im lặng hỏng sau một lần
   nâng cấp mà không ai biết.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

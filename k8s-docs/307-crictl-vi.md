# Debug node Kubernetes bằng crictl (Debugging Kubernetes nodes with crictl)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối — nhánh `/docs/tasks/`](00-ALO-TRINH-ADMIN.md#checkpoint-tiếp-nối--nhánh-docstasks)
→ [CP9 — Xử lý sự cố](00-ALO-TRINH-ADMIN.md#cp9--xử-lý-sự-cố), bài 2/10 · thực hành trực tiếp trên
node của cluster VM [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

Bài này nối tiếp bài [00 — Container runtime](00-container-runtimes-vi.md): ở đó bạn đã dựng
containerd làm CRI runtime; bài này cho bạn công cụ dòng lệnh để nói chuyện thẳng với runtime đó
ngay trên node khi cần debug — thay cho lệnh `docker` của thời dockershim.

**Phải hiểu ở lần đọc này:**

- `crictl` là giao diện dòng lệnh cho **mọi container runtime tương thích CRI**, dùng để kiểm
  tra và debug runtime cùng các ứng dụng ngay trên một node Kubernetes; yêu cầu duy nhất là
  Linux có một CRI runtime.
- Ba cách chỉ định endpoint cho `crictl`: flag `--runtime-endpoint`/`--image-endpoint`, biến môi
  trường `CONTAINER_RUNTIME_ENDPOINT`/`IMAGE_SERVICE_ENDPOINT`, hoặc file `/etc/crictl.yaml` —
  và vì sao nên đặt tường minh: không đặt thì `crictl` dò lần lượt danh sách endpoint đã biết,
  có thể ảnh hưởng hiệu năng. Với containerd, endpoint là
  `unix:///var/run/containerd/containerd.sock`.
- Phân biệt hai tầng liệt kê: `crictl pods` liệt kê pod (lọc được bằng `--name`, `--label`),
  còn `crictl ps` liệt kê container — `ps` chỉ hiện container đang chạy, phải dùng `ps -a` mới
  thấy cả container đã `Exited`.
- Bộ lệnh thao tác trực tiếp: `crictl images` (lọc theo repository, `-q` chỉ in ID),
  `crictl exec -i -t <id> <lệnh>` để chạy lệnh trong container đang chạy, và
  `crictl logs <id>` (kèm `--tail=N`) để đọc log container.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Tên image trong output mẫu (`hyperkube-amd64` v1.10, registry `k8s-gcrio.azureedge.net`, `nvidia-device-plugin`) | output từ thời trang gốc được viết, chỉ dùng minh họa định dạng cột | chạy lại từng lệnh trên `k8s-worker2` của cluster lab để thấy output thật của containerd |
| Mục "Cài đặt crictl" | node dựng theo quy trình [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) thường đã có sẵn `crictl` (gói `cri-tools` đi kèm kho gói của kubeadm) — kiểm tra bằng `crictl --version` | chỉ quay lại mục này khi node thiếu binary |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.11 [stable]`

`crictl` là một giao diện dòng lệnh cho các container runtime tương thích CRI. Bạn có thể dùng
nó để kiểm tra và debug các container runtime cũng như các ứng dụng trên một node Kubernetes.
`crictl` và mã nguồn của nó được lưu trữ trong repository
[cri-tools](https://github.com/kubernetes-sigs/cri-tools).

## Trước khi bạn bắt đầu (Before you begin)

`crictl` yêu cầu hệ điều hành Linux có một CRI runtime.

## Cài đặt crictl (Installing crictl)

Bạn có thể tải về `crictl` dưới dạng archive nén từ
[trang release](https://github.com/kubernetes-sigs/cri-tools/releases) của cri-tools, cho nhiều
kiến trúc khác nhau. Hãy tải phiên bản tương ứng với phiên bản Kubernetes của bạn. Giải nén và
di chuyển nó tới một vị trí nằm trong system path, chẳng hạn `/usr/local/bin/`.

## Cách dùng chung (General usage)

Lệnh `crictl` có một số subcommand và runtime flag. Dùng `crictl help` hoặc
`crictl <subcommand> help` để biết thêm chi tiết.

Bạn có thể đặt endpoint cho `crictl` bằng một trong các cách sau:

* Đặt các flag `--runtime-endpoint` và `--image-endpoint`.
* Đặt các biến môi trường `CONTAINER_RUNTIME_ENDPOINT` và `IMAGE_SERVICE_ENDPOINT`.
* Đặt endpoint trong file cấu hình `/etc/crictl.yaml`. Để chỉ định một file khác, dùng flag
  `--config=PATH_TO_FILE` khi bạn chạy `crictl`.

> **Ghi chú:**
> Nếu bạn không đặt endpoint, `crictl` sẽ thử kết nối tới một danh sách các endpoint đã biết,
> điều này có thể gây ảnh hưởng tới hiệu năng.

Bạn cũng có thể chỉ định giá trị timeout khi kết nối tới server và bật hoặc tắt chế độ debug,
bằng cách chỉ định giá trị `timeout` hoặc `debug` trong file cấu hình, hoặc dùng các flag dòng
lệnh `--timeout` và `--debug`.

Để xem hoặc chỉnh sửa cấu hình hiện tại, hãy xem hoặc chỉnh sửa nội dung của
`/etc/crictl.yaml`. Ví dụ, cấu hình khi dùng container runtime `containerd` sẽ tương tự như
sau:

```
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: true
```

Để tìm hiểu thêm về `crictl`, tham khảo
[tài liệu `crictl`](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md).

## Ví dụ các lệnh crictl (Example crictl commands)

Các ví dụ sau đây minh họa một số lệnh `crictl` và output mẫu.

### Liệt kê pod (List pods)

Liệt kê tất cả các pod:

```shell
crictl pods
```

Output tương tự như sau:

```
POD ID              CREATED              STATE               NAME                         NAMESPACE           ATTEMPT
926f1b5a1d33a       About a minute ago   Ready               sh-84d7dcf559-4r2gq          default             0
4dccb216c4adb       About a minute ago   Ready               nginx-65899c769f-wv2gp       default             0
a86316e96fa89       17 hours ago         Ready               kube-proxy-gblk4             kube-system         0
919630b8f81f1       17 hours ago         Ready               nvidia-device-plugin-zgbbv   kube-system         0
```

Liệt kê pod theo tên:

```shell
crictl pods --name nginx-65899c769f-wv2gp
```

Output tương tự như sau:

```
POD ID              CREATED             STATE               NAME                     NAMESPACE           ATTEMPT
4dccb216c4adb       2 minutes ago       Ready               nginx-65899c769f-wv2gp   default             0
```

Liệt kê pod theo label:

```shell
crictl pods --label run=nginx
```

Output tương tự như sau:

```
POD ID              CREATED             STATE               NAME                     NAMESPACE           ATTEMPT
4dccb216c4adb       2 minutes ago       Ready               nginx-65899c769f-wv2gp   default             0
```

### Liệt kê image (List images)

Liệt kê tất cả các image:

```shell
crictl images
```

Output tương tự như sau:

```
IMAGE                                     TAG                 IMAGE ID            SIZE
busybox                                   latest              8c811b4aec35f       1.15MB
k8s-gcrio.azureedge.net/hyperkube-amd64   v1.10.3             e179bbfe5d238       665MB
k8s-gcrio.azureedge.net/pause-amd64       3.1                 da86e6ba6ca19       742kB
nginx                                     latest              cd5239a0906a6       109MB
```

Liệt kê image theo repository:

```shell
crictl images nginx
```

Output tương tự như sau:

```
IMAGE               TAG                 IMAGE ID            SIZE
nginx               latest              cd5239a0906a6       109MB
```

Chỉ liệt kê ID của image:

```shell
crictl images -q
```

Output tương tự như sau:

```
sha256:8c811b4aec35f259572d0f79207bc0678df4c736eeec50bc9fec37ed936a472a
sha256:e179bbfe5d238de6069f3b03fccbecc3fb4f2019af741bfff1233c4d7b2970c5
sha256:da86e6ba6ca197bf6bc5e9d900febd906b133eaa4750e6bed647b0fbe50ed43e
sha256:cd5239a0906a6ccf0562354852fae04bc5b52d72a2aff9a871ddb6bd57553569
```

### Liệt kê container (List containers)

Liệt kê tất cả các container:

```shell
crictl ps -a
```

Output tương tự như sau:

```
CONTAINER ID        IMAGE                                                                                                             CREATED             STATE               NAME                       ATTEMPT
1f73f2d81bf98       busybox@sha256:141c253bc4c3fd0a201d32dc1f493bcf3fff003b6df416dea4f41046e0f37d47                                   7 minutes ago       Running             sh                         1
9c5951df22c78       busybox@sha256:141c253bc4c3fd0a201d32dc1f493bcf3fff003b6df416dea4f41046e0f37d47                                   8 minutes ago       Exited              sh                         0
87d3992f84f74       nginx@sha256:d0a8828cccb73397acb0073bf34f4d7d8aa315263f1e7806bf8c55d8ac139d5f                                     8 minutes ago       Running             nginx                      0
1941fb4da154f       k8s-gcrio.azureedge.net/hyperkube-amd64@sha256:00d814b1f7763f4ab5be80c58e98140dfc69df107f253d7fdd714b30a714260a   18 hours ago        Running             kube-proxy                 0
```

Liệt kê các container đang chạy:

```shell
crictl ps
```

Output tương tự như sau:

```
CONTAINER ID        IMAGE                                                                                                             CREATED             STATE               NAME                       ATTEMPT
1f73f2d81bf98       busybox@sha256:141c253bc4c3fd0a201d32dc1f493bcf3fff003b6df416dea4f41046e0f37d47                                   6 minutes ago       Running             sh                         1
87d3992f84f74       nginx@sha256:d0a8828cccb73397acb0073bf34f4d7d8aa315263f1e7806bf8c55d8ac139d5f                                     7 minutes ago       Running             nginx                      0
1941fb4da154f       k8s-gcrio.azureedge.net/hyperkube-amd64@sha256:00d814b1f7763f4ab5be80c58e98140dfc69df107f253d7fdd714b30a714260a   17 hours ago        Running             kube-proxy                 0
```

### Thực thi một lệnh trong container đang chạy (Execute a command in a running container)

```shell
crictl exec -i -t 1f73f2d81bf98 ls
```

Output tương tự như sau:

```
bin   dev   etc   home  proc  root  sys   tmp   usr   var
```

### Lấy log của một container (Get a container's logs)

Lấy toàn bộ log của container:

```shell
crictl logs 87d3992f84f74
```

Output tương tự như sau:

```
10.240.0.96 - - [06/Jun/2018:02:45:49 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
10.240.0.96 - - [06/Jun/2018:02:45:50 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
10.240.0.96 - - [06/Jun/2018:02:45:51 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
```

Chỉ lấy `N` dòng log mới nhất:

```shell
crictl logs --tail=1 87d3992f84f74
```

Output tương tự như sau:

```
10.240.0.96 - - [06/Jun/2018:02:45:51 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
```

## Tiếp theo (What's next)

* [Tìm hiểu thêm về `crictl`](https://github.com/kubernetes-sigs/cri-tools).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở CP9:

1. kubelet trên `k8s-worker2` của cluster lab không khởi động được, nên bạn không tin vào những
   gì `kubectl get pods` báo về node đó. SSH vào node, bạn dùng những lệnh `crictl` nào để biết
   pod và container nào đang thực sự chạy, và `crictl` cần điều kiện gì trên node để kết nối
   được với containerd?
2. **Câu bẫy.** Bạn chạy `crictl ps` trên node và không thấy container của ứng dụng bị lỗi.
   Có thể kết luận container đó không tồn tại trên node không? Nếu không, lệnh nào cho câu trả
   lời đầy đủ hơn?
3. Node vẫn hoạt động dù `/etc/crictl.yaml` không tồn tại và bạn chạy `crictl` không kèm flag
   endpoint nào. Vì sao lệnh vẫn chạy được, và vì sao bài vẫn khuyên đặt endpoint tường minh?
4. Viết hai lệnh: (a) lấy đúng 1 dòng log mới nhất của container có ID `87d3992f84f74`, và
   (b) chạy lệnh `ls` bên trong container đang chạy có ID `1f73f2d81bf98`.

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Dùng **`crictl pods`** để liệt kê pod và **`crictl ps -a`** để liệt kê container (kể cả
   container đã dừng); có thể lọc bằng `--name` hoặc `--label` với `crictl pods`. Điều kiện:
   node Linux có **một CRI runtime đang chạy**, và `crictl` biết endpoint của runtime — qua
   flag `--runtime-endpoint`/`--image-endpoint`, biến môi trường, hoặc `/etc/crictl.yaml` trỏ
   tới `unix:///var/run/containerd/containerd.sock` với containerd. `crictl` làm việc trực
   tiếp với runtime trên node, đó chính là lý do nó là công cụ debug khi tầng phía trên có
   vấn đề.
2. **Không thể kết luận như vậy.** `crictl ps` chỉ liệt kê container **đang chạy**; một
   container đã crash nằm ở trạng thái `Exited` và chỉ hiện ra với **`crictl ps -a`** — như
   trong ví dụ của bài, container `sh` với ATTEMPT 0 ở trạng thái `Exited` chỉ xuất hiện khi
   có `-a`. Trực giác từ thói quen đọc output "có gì hiện nấy" khiến người ta bỏ sót đúng cái
   container cần xem log nhất.
3. Vì khi không đặt endpoint, `crictl` **thử kết nối lần lượt tới một danh sách endpoint đã
   biết** cho tới khi được — nên lệnh vẫn chạy. Nhưng chính việc dò lần lượt đó **có thể ảnh
   hưởng hiệu năng**, vì vậy bài khuyên đặt endpoint tường minh (flag, biến môi trường, hoặc
   `/etc/crictl.yaml`).
4. (a) `crictl logs --tail=1 87d3992f84f74` — flag `--tail=N` giới hạn N dòng mới nhất.
   (b) `crictl exec -i -t 1f73f2d81bf98 ls`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau của
[CP9](00-ALO-TRINH-ADMIN.md#cp9--xử-lý-sự-cố).

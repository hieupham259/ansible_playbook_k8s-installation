# Thực hiện rollback trên một DaemonSet (Perform a Rollback on a DaemonSet)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/manage-daemon/rollback-daemon-set/

Trang này hướng dẫn cách thực hiện rollback (quay lui) trên một DaemonSet.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Server Kubernetes của bạn phải ở phiên bản 1.7 hoặc mới hơn. Để kiểm tra phiên bản, hãy nhập
`kubectl version`.

Bạn nên biết trước cách [thực hiện rolling update trên một
DaemonSet](388-update-daemon-set-vi.md).

## Thực hiện rollback trên một DaemonSet (Performing a rollback on a DaemonSet)

### Bước 1: Tìm revision của DaemonSet mà bạn muốn quay lui về (Step 1: Find the DaemonSet revision you want to roll back to)

Bạn có thể bỏ qua bước này nếu bạn chỉ muốn rollback về revision gần nhất.

Liệt kê tất cả revision của một DaemonSet:

```shell
kubectl rollout history daemonset <daemonset-name>
```

Lệnh này trả về danh sách các revision của DaemonSet:

```
daemonsets "<daemonset-name>"
REVISION        CHANGE-CAUSE
1               ...
2               ...
...
```

* Nguyên nhân thay đổi (change cause) được sao chép từ annotation `kubernetes.io/change-cause`
  của DaemonSet sang các revision của nó khi tạo. Bạn có thể chỉ định `--record=true` trong
  `kubectl` để ghi lại lệnh đã thực thi vào annotation change cause.

Để xem chi tiết của một revision cụ thể:

```shell
kubectl rollout history daemonset <daemonset-name> --revision=1
```

Lệnh này trả về chi tiết của revision đó:

```
daemonsets "<daemonset-name>" with revision #1
Pod Template:
Labels:       foo=bar
Containers:
app:
 Image:        ...
 Port:         ...
 Environment:  ...
 Mounts:       ...
Volumes:      ...
```

### Bước 2: Rollback về một revision cụ thể (Step 2: Roll back to a specific revision)

```shell
# Chỉ định số revision mà bạn lấy được ở Bước 1 vào --to-revision
kubectl rollout undo daemonset <daemonset-name> --to-revision=<revision>
```

Nếu thành công, lệnh trả về:

```
daemonset "<daemonset-name>" rolled back
```

> **Ghi chú:** Nếu cờ `--to-revision` không được chỉ định, kubectl sẽ chọn revision gần nhất.

### Bước 3: Theo dõi tiến trình rollback của DaemonSet (Step 3: Watch the progress of the DaemonSet rollback)

`kubectl rollout undo daemonset` báo cho server bắt đầu rollback DaemonSet. Việc rollback thực sự
được thực hiện bất đồng bộ (asynchronously) bên trong control plane của cluster.

Để theo dõi tiến trình của việc rollback:

```shell
kubectl rollout status ds/<daemonset-name>
```

Khi rollback hoàn tất, output tương tự như sau:

```
daemonset "<daemonset-name>" successfully rolled out
```

## Hiểu về các revision của DaemonSet (Understanding DaemonSet revisions)

Ở bước `kubectl rollout history` phía trên, bạn đã lấy được danh sách các revision của DaemonSet.
Mỗi revision được lưu trong một resource có tên là ControllerRevision.

Để xem những gì được lưu trong mỗi revision, hãy tìm các resource thô (raw) của revision DaemonSet:

```shell
kubectl get controllerrevision -l <daemonset-selector-key>=<daemonset-selector-value>
```

Lệnh này trả về danh sách các ControllerRevision:

```
NAME                               CONTROLLER                     REVISION   AGE
<daemonset-name>-<revision-hash>   DaemonSet/<daemonset-name>     1          1h
<daemonset-name>-<revision-hash>   DaemonSet/<daemonset-name>     2          1h
```

Mỗi ControllerRevision lưu các annotation và template của một revision DaemonSet.

`kubectl rollout undo` lấy một ControllerRevision cụ thể và thay thế template của DaemonSet bằng
template được lưu trong ControllerRevision đó. `kubectl rollout undo` tương đương với việc cập nhật
template của DaemonSet về một revision trước đó bằng các lệnh khác, chẳng hạn `kubectl edit` hoặc
`kubectl apply`.

> **Ghi chú:** Các revision của DaemonSet chỉ tiến về phía trước (roll forward). Nghĩa là sau khi
> một lần rollback hoàn tất, số revision (trường `.revision`) của ControllerRevision được quay lui
> về sẽ tăng lên. Ví dụ, nếu trong hệ thống bạn có revision 1 và 2, và bạn rollback từ revision 2
> về revision 1, thì ControllerRevision có `.revision: 1` sẽ trở thành `.revision: 3`.

## Xử lý sự cố (Troubleshooting)

* Xem [xử lý sự cố rolling update của
  DaemonSet](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/#troubleshooting).

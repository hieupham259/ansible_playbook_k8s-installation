# Gỡ lỗi Init Container (Debug Init Containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-application/debug-init-containers/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 24 — Xử lý sự cố](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố),
bài 9/10 · Giai đoạn này của Phần II không có lab riêng: thực hành trực tiếp trên cluster lab ở mốc
`04-metrics-ready` (xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)), tự chấm bằng Checkpoint ghi ở
cuối mục giai đoạn trong lộ trình.

Bài ngắn và là một **ca đặc biệt** của giai đoạn: Pod kẹt ở init container trông giống Pod hỏng
bình thường, nhưng cả ba lệnh quen thuộc — `get pod`, `describe pod`, `logs` — đều phải đọc hoặc
gõ khác đi một chút. Đọc bài này chính là để biết ba chỗ khác đó nằm ở đâu.

**Phải hiểu ở lần đọc này:**

- Mục *Kiểm tra trạng thái của Init Container*: `kubectl get pod <pod-name>` đã đủ để biết đang kẹt
  ở đâu — cột `STATUS` hiện `Init:N/M`, nghĩa là Pod có `M` init container và `N` cái đã hoàn thành
  (ví dụ `Init:1/2`).
- Bảng ở mục *Hiểu trạng thái Pod* và ranh giới giữa năm giá trị: `Init:N/M` (đang chạy dở),
  `Init:Error` (một init container **thực thi thất bại**), `Init:CrashLoopBackOff` (một init
  container **thất bại lặp đi lặp lại**), `Pending` (Pod **chưa bắt đầu** thực thi init container),
  `PodInitializing` hoặc `Running` (đã thực thi **xong** init container).
- Mục *Xem chi tiết về Init Container*: `kubectl describe pod <pod-name>` in ra một khối
  **`Init Containers:` riêng**, tách khỏi khối `Containers:`. Mỗi init container ở đó có `State`,
  `Last State`, `Reason`, `Exit Code` và `Restart Count` — đây là chỗ đọc để biết cái nào hỏng và
  hỏng thế nào.
- Mục *Truy cập log của Init Container*: phải **truyền tên init container** —
  `kubectl logs <pod-name> -c <init-container-2>`. Kèm mẹo của bài: init container chạy shell
  script thì đặt `set -x` ở đầu script để nó in ra từng lệnh khi thực thi.
- Cách đọc trạng thái theo hướng lập trình: trường **`status.initContainerStatuses`**, lấy bằng
  `kubectl get pod <pod-name> --template '{{.status.initContainerStatuses}}'` — cùng thông tin với
  `describe`, chỉ khác định dạng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* (minikube, các playground, `kubectl version`) | cluster lab đã sẵn sàng ở mốc `04-metrics-ready` với đủ hai worker không phải control plane | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Hai điều kiện tiên quyết: khái niệm Init Container và cách cấu hình một Init Container | đã học rồi, chỉ mở lại nếu quên | bài [50](50-init-containers-vi.md) ở [giai đoạn 3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời) và bài thực hành [276](276-configure-pod-initialization-vi.md) của cùng giai đoạn |
| Cú pháp Go template của `--template` | là kỹ năng công cụ `kubectl`, chép nguyên lệnh mẫu là đủ | bài [303](303-determine-reason-pod-failure-vi.md) (bài 8/10) vừa đọc, dùng đúng cơ chế này |

---

Trang này chỉ ra cách điều tra các vấn đề liên quan đến việc thực thi Init Container. Các dòng
lệnh ví dụ dưới đây gọi Pod là `<pod-name>` và các Init Container là `<init-container-1>` và
`<init-container-2>`.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

* Bạn nên nắm được những điều cơ bản về [Init Container](./50-init-containers-vi.md).
* Bạn nên đã [Cấu hình một Init Container](./276-configure-pod-initialization-vi.md#tạo-một-pod-có-init-container-create-a-pod-that-has-an-init-container).

## Kiểm tra trạng thái của Init Container (Checking the status of Init Containers)

Hiển thị trạng thái của Pod của bạn:

```shell
kubectl get pod <pod-name>
```

Ví dụ, trạng thái `Init:1/2` cho biết một trong hai Init Container đã hoàn thành thành công:

```
NAME         READY     STATUS     RESTARTS   AGE
<pod-name>   0/1       Init:1/2   0          7s
```

Xem [Hiểu trạng thái Pod](#hiểu-trạng-thái-pod-understanding-pod-status) để có thêm ví dụ về
các giá trị trạng thái và ý nghĩa của chúng.

## Xem chi tiết về Init Container (Getting details about Init Containers)

Xem thông tin chi tiết hơn về việc thực thi Init Container:

```shell
kubectl describe pod <pod-name>
```

Ví dụ, một Pod có hai Init Container có thể hiển thị như sau:

```
Init Containers:
  <init-container-1>:
    Container ID:    ...
    ...
    State:           Terminated
      Reason:        Completed
      Exit Code:     0
      Started:       ...
      Finished:      ...
    Ready:           True
    Restart Count:   0
    ...
  <init-container-2>:
    Container ID:    ...
    ...
    State:           Waiting
      Reason:        CrashLoopBackOff
    Last State:      Terminated
      Reason:        Error
      Exit Code:     1
      Started:       ...
      Finished:      ...
    Ready:           False
    Restart Count:   3
    ...
```

Bạn cũng có thể truy cập trạng thái của các Init Container theo cách lập trình bằng cách đọc
trường `status.initContainerStatuses` trong Pod Spec:

```shell
kubectl get pod <pod-name> --template '{{.status.initContainerStatuses}}'
```

Lệnh này sẽ trả về cùng thông tin như trên, được định dạng bằng một
[Go template](https://pkg.go.dev/text/template).

## Truy cập log của Init Container (Accessing logs from Init Containers)

Truyền tên Init Container cùng với tên Pod để truy cập log của nó.

```shell
kubectl logs <pod-name> -c <init-container-2>
```

Các Init Container chạy shell script sẽ in ra các lệnh khi chúng được thực thi. Ví dụ, bạn có
thể làm điều này trong Bash bằng cách chạy `set -x` ở đầu script.

## Hiểu trạng thái Pod (Understanding Pod status) {#understanding-pod-status}

Một trạng thái Pod bắt đầu bằng `Init:` tóm tắt trạng thái thực thi của các Init Container.
Bảng dưới đây mô tả một số giá trị trạng thái ví dụ mà bạn có thể gặp khi gỡ lỗi Init Container.

Trạng thái | Ý nghĩa
------ | -------
`Init:N/M` | Pod có `M` Init Container, và cho tới lúc này `N` container đã hoàn thành.
`Init:Error` | Một Init Container đã thực thi thất bại.
`Init:CrashLoopBackOff` | Một Init Container đã thất bại lặp đi lặp lại.
`Pending` | Pod chưa bắt đầu thực thi các Init Container.
`PodInitializing` hoặc `Running` | Pod đã thực thi xong các Init Container.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 24:

1. Trên cluster lab, một Pod nằm trên `lab-k8s-worker2` hiện `Init:1/2` rồi một lúc sau chuyển
   thành `Init:CrashLoopBackOff`. Mỗi trạng thái nói lên điều gì? Bạn mở khối nào trong output của
   `kubectl describe pod` để biết init container nào hỏng và nó thoát với exit code bao nhiêu?
2. **Câu bẫy.** Pod đang ở `Init:CrashLoopBackOff`. Bạn gõ `kubectl logs <pod-name>` và không thu
   được gì hữu ích. Vì sao, và lệnh đúng là gì?
3. Phân biệt `Pending` với `Init:N/M`, và phân biệt `Init:Error` với `Init:CrashLoopBackOff`.
4. Init container của bạn chạy một shell script khá dài và không in ra gì cả, nên bạn không biết
   nó chết ở bước nào. Bài gợi ý cách nào để thấy nó đang chạy tới đâu?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `Init:1/2` nghĩa là **Pod có hai init container và một cái đã hoàn thành thành công** — nó đang
   chạy cái thứ hai. `Init:CrashLoopBackOff` nghĩa là **một init container đã thất bại lặp đi lặp
   lại**. Đọc ở khối **`Init Containers:`** của `kubectl describe pod` — khối riêng, tách khỏi khối
   `Containers:`. Ở đó mỗi init container có `State` (ví dụ `Waiting` với `Reason:
   CrashLoopBackOff`), `Last State` (`Terminated` với `Reason: Error` và `Exit Code`), `Ready` và
   `Restart Count`.
2. Vì **`kubectl logs` không tự chọn init container**. Bài nói rõ: phải **truyền tên init container
   cùng với tên Pod** thì mới truy cập được log của nó —
   **`kubectl logs <pod-name> -c <init-container-2>`**. Ở giai đoạn `Init:...` thì container ứng
   dụng còn chưa được khởi động, nên không có log ứng dụng nào để mà xem.
3. **`Pending` là Pod chưa bắt đầu thực thi các init container**, còn **`Init:N/M` là đã bắt đầu và
   đang chạy dở** — ranh giới ở chỗ init container đã khởi động hay chưa. **`Init:Error` là một
   init container đã thực thi thất bại**, còn **`Init:CrashLoopBackOff` là một init container đã
   thất bại lặp đi lặp lại** — ranh giới ở chỗ hỏng một lần hay hỏng lặp lại, đúng như `Restart
   Count` trong khối `Init Containers:` phản ánh.
4. Đặt **`set -x` ở đầu script**. Bài nói: init container chạy shell script sẽ **in ra các lệnh khi
   chúng được thực thi**, và với Bash bạn bật điều đó bằng `set -x` ở đầu script. Sau đó đọc bằng
   `kubectl logs <pod-name> -c <tên init container>` là thấy nó dừng ở lệnh nào.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

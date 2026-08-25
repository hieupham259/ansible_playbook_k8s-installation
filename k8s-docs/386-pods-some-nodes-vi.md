# Chỉ chạy Pod trên một số Node nhất định (Running Pods on Only Some Nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-daemon/pods-some-nodes/>
>
> Hướng dẫn dùng label của node và `nodeSelector` để một DaemonSet chỉ đặt Pod lên một tập node
> nhất định thay vì mọi node trong cluster.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 29 — DaemonSet, Job nâng cao và thiết bị chuyên dụng](00-ALO-TRINH-ADMIN.md#giai-đoạn-29--daemonset-job-nâng-cao-và-thiết-bị-chuyên-dụng),
bài 5/8 · Kiểm chứng trực tiếp trên cluster lab: gắn label cho `lab-k8s-worker1`, tạo DaemonSet có
`nodeSelector` khớp label đó, rồi gắn label cho `lab-k8s-worker2` và xem Pod thứ hai tự xuất hiện.

Bài rất ngắn và là bài cuối của nhóm `tasks/manage-daemon/`. Nó dùng đúng một công cụ bạn đã học kỹ
ở bài [138](138-assign-pod-node-vi.md) và thực hành ở
[Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md) phần B2 — `nodeSelector`. Điểm mới **không** nằm ở
`nodeSelector` mà ở chỗ ai phản ứng khi tập node đổi. Một lưu ý khi làm thật: manifest trong bài để
`image: example-image`, đó là chỗ giữ chỗ, thay bằng một image chạy được trước khi apply.

**Phải hiểu ở lần đọc này:**

- Bài toán ở mục *Chỉ chạy Pod trên một số Node nhất định*: DaemonSet mặc định phủ mọi node đủ điều
  kiện; muốn thu hẹp thì **gắn label lên node** rồi lọc bằng `nodeSelector`. Ví dụ của bài là node
  có SSD cục bộ, vì dịch vụ cache chỉ có nghĩa khi có lưu trữ độ trễ thấp.
- Bước 1: `kubectl label nodes example-node-1 example-node-2 ssd=true` — label này nằm trên **đối
  tượng Node**, không nằm trên Pod, và một lệnh gắn được nhiều node cùng lúc.
- Bước 2: `nodeSelector` đặt trong `.spec.template.spec`, tức nó thuộc **Pod template**, không phải
  một trường riêng của DaemonSet. Giá trị viết là `"true"` trong ngoặc kép — label là chuỗi.
- Bước 3 là ý chính của cả bài: sau khi DaemonSet đã chạy, bạn gắn `ssd=true` cho một node thứ ba
  thì **control plane — cụ thể là DaemonSet controller — tự chạy một daemon pod mới trên node đó**.
  Tập node đủ điều kiện là đầu vào **động**, không phải chốt một lần lúc tạo.
- Cách đọc bằng chứng ở output cuối bài: `kubectl get pods -o wide`, nhìn cột `NODE` để biết Pod nằm
  đâu và cột `AGE` để thấy Pod trên node thứ ba **trẻ hơn hẳn** hai Pod kia — đó chính là dấu vết
  của việc gắn label sau.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và ba playground | Lộ trình cấm minikube, kind và cluster dùng chung | Bỏ hẳn — dùng ba VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Nhãn `app: nginx` ở `metadata.labels` của DaemonSet | Nó không tham gia vào `selector`, cũng không liên quan `nodeSelector` — chỉ là nhãn dán lên chính đối tượng DaemonSet | Ba chỗ đặt label khác nhau đã phân biệt ở bài [18](18-labels-vi.md), giai đoạn 1 |
| Các cách thu hẹp tập node khác ngoài `nodeSelector` — node affinity, taint và toleration | Bài chỉ trình bày đúng một công cụ | Bài [138](138-assign-pod-node-vi.md) và [139](139-taint-and-toleration-vi.md) ở [giai đoạn 7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction), thực hành ở [Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md) phần B2–B4 |

---

Trang này minh họa cách bạn có thể chỉ chạy Pod trên một số Node nhất định trong khuôn khổ một
DaemonSet.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Chỉ chạy Pod trên một số Node nhất định (Running Pods on only some Nodes)

Hãy hình dung bạn muốn chạy một DaemonSet, nhưng bạn chỉ cần chạy các daemon pod đó trên những
node có ổ lưu trữ thể rắn (SSD) cục bộ. Chẳng hạn, Pod có thể cung cấp dịch vụ cache cho node, và
cache chỉ hữu ích khi có sẵn bộ lưu trữ cục bộ độ trễ thấp.

### Bước 1: Thêm label cho các node của bạn (Step 1: Add labels to your nodes)

Thêm label `ssd=true` vào những node có ổ SSD.

```shell
kubectl label nodes example-node-1 example-node-2 ssd=true
```

### Bước 2: Tạo manifest (Step 2: Create the manifest)

Hãy tạo một DaemonSet để cấp phát (provision) các daemon pod chỉ trên những node được gắn label
SSD.

Tiếp theo, hãy dùng `nodeSelector` để đảm bảo DaemonSet chỉ chạy Pod trên các node có label `ssd`
được đặt bằng `"true"`.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-driver
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: ssd-driver-pod
  template:
    metadata:
      labels:
        app: ssd-driver-pod
    spec:
      nodeSelector:
        ssd: "true"
      containers:
        - name: example-container
          image: example-image
```

### Bước 3: Tạo DaemonSet (Step 3: Create the DaemonSet)

Tạo DaemonSet từ manifest bằng cách dùng `kubectl create` hoặc `kubectl apply`.

Bây giờ hãy gắn label `ssd=true` cho một node khác.

```shell
kubectl label nodes example-node-3 ssd=true
```

Việc gắn label cho node sẽ tự động kích hoạt control plane (cụ thể là DaemonSet controller) chạy
một daemon pod mới trên node đó.

```shell
kubectl get pods -o wide
```

Kết quả trả về sẽ tương tự như sau:

```
NAME                              READY     STATUS    RESTARTS   AGE    IP      NODE
<daemonset-name><some-hash-01>    1/1       Running   0          13s    .....   example-node-1
<daemonset-name><some-hash-02>    1/1       Running   0          13s    .....   example-node-2
<daemonset-name><some-hash-03>    1/1       Running   0          5s     .....   example-node-3
```

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 29:

1. Bạn gắn `ssd=true` cho `lab-k8s-worker1`, apply DaemonSet `ssd-driver`, rồi vài phút sau gắn
   `ssd=true` cho `lab-k8s-worker2`. Bạn phải làm thêm thao tác gì để có Pod trên `lab-k8s-worker2`,
   và **thành phần nào** của control plane tạo ra Pod đó?
2. **Câu bẫy.** Một Deployment cũng có thể khai `nodeSelector` trong Pod template. Nếu bạn gắn thêm
   label cho một node trong khi Deployment đó đang chạy, số Pod của nó có tăng không? Vì sao với
   DaemonSet thì lại có Pod mới?
3. Trong manifest của bài có hai chỗ đều tên gần giống nhau: `spec.selector.matchLabels` và
   `spec.template.spec.nodeSelector`. Cái nào chọn **Pod**, cái nào chọn **node** — và nếu bỏ hẳn
   `nodeSelector` đi thì DaemonSet chạy ở đâu?
4. Output cuối bài cho ba Pod với `AGE` lần lượt là `13s`, `13s` và `5s`. Chênh lệch đó nói lên điều
   gì về thứ tự các thao tác đã diễn ra?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không phải làm gì thêm** — chỉ cần gắn label. Bài nói rõ ở Bước 3: việc gắn label cho node
   **tự động kích hoạt control plane, cụ thể là DaemonSet controller**, chạy một daemon pod mới trên
   node đó. Bạn không apply lại manifest, không xóa Pod, không đụng tới DaemonSet.
2. **Không tăng.** Với Deployment, số Pod do `replicas` quyết định, còn `nodeSelector` chỉ là **điều
   kiện lọc lúc lập lịch** — nó nói Pod *được phép* đặt ở đâu, chứ không sinh thêm Pod khi có node
   mới thỏa điều kiện. Với DaemonSet thì ngược lại: **tập node đủ điều kiện chính là thứ quyết định
   số Pod**, nên DaemonSet controller theo dõi tập đó và thêm một Pod ngay khi tập đó rộng ra. Chỗ
   dễ nhầm là tưởng `nodeSelector` cư xử như nhau ở mọi controller — cùng một trường, nhưng ở
   DaemonSet nó là **đầu vào động của số lượng Pod**.
3. `spec.selector.matchLabels` (`app: ssd-driver-pod`) chọn **Pod** — nó khớp với
   `template.metadata.labels` để DaemonSet nhận ra Pod nào là của mình.
   `spec.template.spec.nodeSelector` (`ssd: "true"`) chọn **node** — nó khớp với label gắn trên đối
   tượng Node. Bỏ `nodeSelector` đi thì DaemonSet quay về hành vi mặc định: **phủ mọi node đủ điều
   kiện** trong cluster, chứ không phải là không chạy ở đâu cả.
4. Rằng hai Pod `13s` sinh ra **cùng lúc** khi DaemonSet được tạo, ứng với hai node đã mang label
   `ssd=true` từ Bước 1; còn Pod `5s` sinh ra **sau đó**, đúng vào lúc node thứ ba được gắn label ở
   Bước 3. **Cột `AGE` chính là bằng chứng cho cơ chế phản ứng theo tập node** — không cần xem log
   controller cũng đọc được thứ tự sự việc.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

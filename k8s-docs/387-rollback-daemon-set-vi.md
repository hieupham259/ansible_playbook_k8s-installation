# Thực hiện rollback trên một DaemonSet (Perform a Rollback on a DaemonSet)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/manage-daemon/rollback-daemon-set/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 29 — DaemonSet, Job nâng cao và thiết bị chuyên dụng](00-ALO-TRINH-ADMIN.md#giai-đoạn-29--daemonset-job-nâng-cao-và-thiết-bị-chuyên-dụng),
bài 4/8 · Kiểm chứng trực tiếp trên cluster lab: chạy `rollout history`, `rollout undo --to-revision`
rồi `rollout status` trên chính DaemonSet bạn vừa update ở bài [388](388-update-daemon-set-vi.md), và
đọc `ControllerRevision` của nó — đây là **nửa sau của Checkpoint giai đoạn 29**.

Bài rất ngắn nhưng là bài duy nhất của lộ trình nói về `ControllerRevision`. Bài giả định bạn vừa
đọc [388](388-update-daemon-set-vi.md) và đang có sẵn một DaemonSet đã qua ít nhất một lần update —
không có revision thứ hai thì không có gì để quay lui. Bạn đã rollback **Deployment** ở
[Lab 4a](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) phần B4 bằng cùng bộ lệnh `rollout
history`/`undo`/`status`; điểm phải để ý ở đây là **nơi lưu revision khác hẳn**, và cách đánh số
revision sau khi quay lui cũng khác trực giác.

**Phải hiểu ở lần đọc này:**

- Ba bước của quy trình: `kubectl rollout history daemonset <tên>` để liệt kê revision (thêm
  `--revision=N` để xem template của đúng revision đó), `kubectl rollout undo daemonset <tên>
  --to-revision=<n>` để quay lui, `kubectl rollout status ds/<tên>` để theo dõi.
- Vì sao bước 3 không thừa: mục *Bước 3* nói rõ `rollout undo` chỉ **báo cho server bắt đầu**, việc
  rollback chạy **bất đồng bộ** trong control plane — lệnh trả về không có nghĩa là đã xong.
- Mục *Hiểu về các revision của DaemonSet*: mỗi revision được lưu trong một resource riêng tên
  `ControllerRevision`, liệt kê được bằng `kubectl get controllerrevision -l <key>=<value>` với
  label selector của DaemonSet; mỗi ControllerRevision giữ annotation và template của revision đó.
- `rollout undo` thực chất chỉ **lấy template trong một ControllerRevision và ghi đè
  `.spec.template` của DaemonSet** — bài nói thẳng nó tương đương với việc bạn tự `kubectl edit`
  hoặc `kubectl apply` về template cũ. Rollback không phải là một cơ chế riêng biệt.
- Ghi chú cuối mục đó: revision của DaemonSet **chỉ tiến về phía trước**. Rollback từ revision 2 về
  revision 1 làm ControllerRevision đang mang `.revision: 1` **trở thành `.revision: 3`**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube, ba playground, và điều kiện "server 1.7 hoặc mới hơn" | Lộ trình cấm minikube/kind; điều kiện phiên bản thì baseline lab vượt xa | Bỏ hẳn — ba VM và bảng phiên bản khóa ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Cờ `--record=true` để ghi lệnh vào annotation `kubernetes.io/change-cause` | Đây là thói quen ghi nhật ký thao tác, không phải cơ chế rollback | Ở lần đọc này chỉ cần biết cột `CHANGE-CAUSE` lấy nội dung từ đâu; annotation nói chung đã học ở bài [20](20-annotations-vi.md) |
| Mục *Xử lý sự cố* | Chỉ là một link trỏ ngược | Bài [388](388-update-daemon-set-vi.md) mục *Khắc phục sự cố*, bạn vừa đọc ở bài 3/8 |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 29:

1. Trên `lab-k8s-master`, bạn chạy `kubectl rollout undo daemonset example-daemonset --to-revision=1`,
   lệnh in ra `rolled back`, nhưng `kubectl get pods -o wide` ngay sau đó vẫn cho thấy Pod cũ trên
   `lab-k8s-worker1` và `lab-k8s-worker2`. Vì sao chưa được kết luận là rollback hỏng, và lệnh nào
   mới trả lời đúng câu hỏi "xong chưa"?
2. **Câu bẫy.** Hệ thống đang có revision 1 và 2. Bạn rollback từ 2 về 1. Sau khi rollback xong,
   `kubectl rollout history daemonset <tên>` cho những số revision nào, và ControllerRevision chứa
   template cũ giờ mang `.revision` bằng bao nhiêu?
3. `example-daemonset` ở bài [385](385-create-daemon-set-vi.md) có selector
   `app.kubernetes.io/name: example`. Viết lệnh liệt kê các revision **thô** của nó, và nói rõ mỗi
   đối tượng trong danh sách đó chứa cái gì.
4. Bài nói `kubectl rollout undo` "tương đương" với `kubectl edit` hoặc `kubectl apply`. Tương đương
   ở chỗ nào — tức `undo` thực sự ghi vào trường nào của DaemonSet?
5. Cột `CHANGE-CAUSE` trong output của `rollout history` để trống. Nội dung của cột đó vốn được lấy
   từ đâu, và nó được chép vào revision ở thời điểm nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì `kubectl rollout undo daemonset` chỉ **báo cho server bắt đầu** rollback; việc rollback được
   thực hiện **bất đồng bộ bên trong control plane**. Lệnh in ra `rolled back` nghĩa là yêu cầu đã
   được nhận, **không** nghĩa là Pod trên các node đã đổi xong. Lệnh trả lời đúng câu hỏi "xong chưa"
   là **`kubectl rollout status ds/example-daemonset`**, và nó chỉ in `successfully rolled out` khi
   rollback hoàn tất.
2. `rollout history` cho **revision 2 và 3**, và ControllerRevision chứa template cũ giờ mang
   **`.revision: 3`**. Chỗ dễ sai là kỳ vọng "quay về 1 thì số revision hiện tại là 1": revision của
   DaemonSet **chỉ tiến về phía trước**, nên quay lui về nội dung cũ vẫn sinh ra một số revision mới,
   lớn hơn. Số revision là **thứ tự thời gian của các lần thay template**, không phải nhãn dán vào
   nội dung template.
3. `kubectl get controllerrevision -l app.kubernetes.io/name=example`. Mỗi đối tượng trong danh sách
   là **một ControllerRevision — nơi lưu một revision của DaemonSet**, giữ **annotation và template**
   của revision đó; cột `CONTROLLER` chỉ về `DaemonSet/example-daemonset` và cột `REVISION` là số
   revision. Nói cách khác, `rollout history` chỉ là cách đọc đẹp của chính tập đối tượng này.
4. Tương đương ở chỗ **cả ba đều chỉ ghi lại `.spec.template` của DaemonSet**: `rollout undo` lấy
   template đang nằm trong ControllerRevision bạn chọn và **thay template hiện tại bằng template đó**.
   Không có cơ chế "quay ngược thời gian" nào cả — vì thế nó cũng kích hoạt một rolling update như
   mọi thay đổi `.spec.template` khác, và vì thế số revision mới lại tăng lên.
5. Lấy từ annotation **`kubernetes.io/change-cause`** của DaemonSet, và nó được **sao chép sang
   revision ở thời điểm revision được tạo** — nên annotation đặt sau khi revision đã sinh ra thì
   không hồi tố. Cột trống chỉ có nghĩa là lúc tạo revision, DaemonSet không mang annotation đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

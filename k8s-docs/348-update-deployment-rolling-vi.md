# Cập nhật một Deployment mà không gây gián đoạn (Update a Deployment Without Downtime)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 4 — Workload controller](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller)
→ [4a. ReplicaSet, Deployment và rollout](00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout),
bài 7/7 của dòng **Thực hành** · Kiểm chứng ở
[Lab 4a — ReplicaSet, Deployment và rollout](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md),
**phần B4** (rollout, history, undo, pause/resume), **B5** (tham số chiến lược), **B6**
(`Recreate`) và **B7** (rollout đình trệ).

Đây là **bài cuối của nhóm 4a**. Nó là bản tóm tắt thao tác của những gì bài
[63](63-deployment-vi.md) đã trình bày ở dạng khái niệm, nên đọc nhanh được — nhưng mọi lệnh
trong bài đều xuất hiện lại trong lab.

**Phải hiểu ở lần đọc này:**

- **Chỉ** thay đổi ở `.spec.template` mới kích hoạt một rolling update. Bài đưa hai đường: sửa
  manifest rồi `kubectl apply`, hoặc `kubectl set image` khi chỉ cần đổi image
  (mục *Thực hiện một rolling update*).
- `kubectl rollout status` là cách theo dõi tiến trình: nó in từng chặng số replica mới đã được
  cập nhật, rồi số replica cũ đang chờ bị chấm dứt, cho tới khi báo `successfully rolled out`
  (mục *Theo dõi tiến trình rollout*).
- `rollout pause` rồi `rollout resume`: các thay đổi thực hiện **trong lúc đang tạm dừng không
  kích hoạt rollout riêng lẻ**; Kubernetes áp dụng tất cả cùng lúc khi bạn resume
  (mục *Tạm dừng và tiếp tục một rollout*).
- `maxUnavailable` và `maxSurge` — mặc định **25%** cho cả hai — khống chế cách RollingUpdate diễn
  ra. Phần trăm tính trên số replica mong muốn, **làm tròn xuống** với `maxUnavailable` và **làm
  tròn lên** với `maxSurge`. Chiến lược còn lại là `Recreate`: bài nói thẳng cách này **gây gián
  đoạn** (mục *Cấu hình chiến lược rolling update*).
- Rollout kẹt và đường lùi: quá `.spec.progressDeadlineSeconds` (mặc định 600 giây) thì condition
  `Progressing` chuyển `False`, đọc bằng `kubectl describe`; đường lùi là `rollout history` rồi
  `rollout undo` (có `--to-revision`). Lịch sử revision **nằm trong các ReplicaSet mà Deployment
  kiểm soát**, mặc định giữ 10 bản, và `.spec.revisionHistoryLimit` bằng `0` vô hiệu hóa hoàn
  toàn khả năng rollback.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube, playground và việc apply manifest thẳng từ `k8s.io/examples` | cluster lab đã dựng sẵn, và lab tự viết manifest vào `~/lab-work/4a/` | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Ví dụ số học chi tiết của `maxUnavailable` và `maxSurge` mà bài đẩy sang tài liệu khái niệm | ở lần đọc này chỉ cần biết hai núm đó khống chế cái gì | bài [63](63-deployment-vi.md#max-surge) cùng nhóm 4a; đo bằng số liệu lấy trên cluster của mình ở [Lab 4a](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) phần B5 |
| Cột `CHANGE-CAUSE` và annotation `kubernetes.io/change-cause` | bài nói rõ annotation này **không** được đặt tự động, nên trên cluster tự dựng cột đó luôn rỗng | [Lab 4a](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) phần B4.2, khi bạn đọc `rollout history` thật |
| Link *thực hiện rolling update trên một DaemonSet* (bài [388](388-update-daemon-set-vi.md)) ở mục *Tiếp theo* | DaemonSet mới được học ở [nhóm 4b](00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling), còn rolling update cho DaemonSet là chủ đề nâng cao | [giai đoạn 29 — DaemonSet, Job nâng cao và thiết bị chuyên dụng](00-ALO-TRINH-ADMIN.md#giai-đoạn-29--daemonset-job-nâng-cao-và-thiết-bị-chuyên-dụng) |
| Link *Horizontal Pod Autoscaling* (bài [342](342-hpa-walkthrough-vi.md)) ở mục *Tiếp theo* | HPA cần metrics-server của giai đoạn 11 — đây là **nợ #1** của lộ trình | [Sổ nợ lộ trình](00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình), trả ở [Lab 11b](labs/LAB-11B-HPA-VA-VPA.md) |

---

Trang này hướng dẫn cách cập nhật một Deployment đang chạy lên phiên bản mới bằng
cập nhật cuốn chiếu (rolling update). Rolling update thay thế dần các Pod cũ bằng các Pod
mới, nhờ đó ứng dụng của bạn vẫn khả dụng trong suốt quá trình.

## Mục tiêu (Objectives)

- Kích hoạt một rolling update trên một Deployment.
- Theo dõi tiến trình rollout.
- Tạm dừng và tiếp tục rollout.
- Cấu hình các tham số của chiến lược rolling update.
- (Nếu cần) Rollback về một revision trước đó.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn cần có sẵn một Deployment. Nếu bạn chưa có, hãy tạo Deployment nginx
từ bài
[Chạy một ứng dụng Stateless bằng Deployment](345-run-stateless-application-vi.md):

```shell
kubectl apply -f https://k8s.io/examples/application/deployment.yaml
```

Xác minh rằng Deployment đang chạy hai Pod:

```shell
kubectl get deployment nginx-deployment
```

Kết quả tương tự như sau:

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           10s
```

## Thực hiện một rolling update (Performing a rolling update)

Bất kỳ thay đổi nào đối với trường `.spec.template` của một Deployment đều kích hoạt một
rolling update. Kubernetes tạo các Pod mới với cấu hình đã cập nhật và chấm dứt dần
các Pod cũ.

### Cập nhật bằng `kubectl apply` (Updating with `kubectl apply`)

Bạn có thể kích hoạt một rolling update bằng cách chỉnh sửa manifest của Deployment rồi apply
thay đổi. Cách tiếp cận này phù hợp khi bạn lưu manifest trong hệ thống quản lý phiên bản
(version control).

Xuất Deployment hiện tại ra một file cục bộ:

```shell
kubectl get deployment nginx-deployment -o yaml > /tmp/nginx-deployment.yaml
```

Chỉnh sửa `/tmp/nginx-deployment.yaml` và đổi `.spec.template.spec.containers[0].image`
từ `nginx:1.14.2` thành `nginx:1.16.1`.

Trước khi apply, hãy so sánh các thay đổi cục bộ của bạn với trạng thái trên cluster:

```shell
kubectl diff -f /tmp/nginx-deployment.yaml
```

Kết quả tương tự như sau:

```
diff -u -N /tmp/LIVE/apps.v1.Deployment.default.nginx-deployment /tmp/MERGED/apps.v1.Deployment.default.nginx-deployment
--- /tmp/LIVE/apps.v1.Deployment...
+++ /tmp/MERGED/apps.v1.Deployment...
@@ -29,7 +29,7 @@
       containers:
-      - image: nginx:1.14.2
+      - image: nginx:1.16.1
         name: nginx
```

Apply manifest đã cập nhật:

```shell
kubectl apply -f /tmp/nginx-deployment.yaml
```

### Chỉ cập nhật container image (Updating only the container image)

Để cập nhật container image mà không cần chỉnh sửa file manifest, hãy dùng
`kubectl set image`:

```shell
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
```

Kết quả tương tự như sau:

```
deployment.apps/nginx-deployment image updated
```

Xác minh rằng image đã được cập nhật:

```shell
kubectl get deployment nginx-deployment -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Kết quả tương tự như sau:

```
nginx:1.16.1
```

## Theo dõi tiến trình rollout (Monitoring rollout progress)

Dùng `kubectl rollout status` để theo dõi tiến trình của một rolling update:

```shell
kubectl rollout status deployment/nginx-deployment
```

Kết quả tương tự như sau:

```
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out
```

Sau khi rollout hoàn tất, xác minh Deployment:

```shell
kubectl get deployment nginx-deployment
```

Kết quả tương tự như sau:

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           2m
```

## Tạm dừng và tiếp tục một rollout (Pausing and resuming a rollout)

Bạn có thể tạm dừng một rollout để kiểm tra một bản cập nhật đang dang dở, hoặc để gom
nhiều thay đổi vào một lần rollout duy nhất.

### Tạm dừng một rollout (Pausing a rollout)

```shell
kubectl rollout pause deployment/nginx-deployment
```

Kết quả tương tự như sau:

```
deployment.apps/nginx-deployment paused
```

### Thực hiện thêm thay đổi trong khi đang tạm dừng (Making additional changes while paused)

Trong khi rollout đang tạm dừng, bạn có thể thực hiện thêm các thay đổi. Những thay đổi
này không kích hoạt rollout mới cho tới khi bạn tiếp tục (resume):

```shell
kubectl set image deployment/nginx-deployment nginx=nginx:1.17.0
```

> **Ghi chú:**
> Bạn có thể thực hiện nhiều thay đổi trên một Deployment đang tạm dừng. Kubernetes áp dụng
> tất cả thay đổi cùng lúc khi bạn tiếp tục rollout.

### Tiếp tục một rollout (Resuming a rollout)

```shell
kubectl rollout resume deployment/nginx-deployment
```

Kết quả tương tự như sau:

```
deployment.apps/nginx-deployment resumed
```

Xác minh rằng rollout hoàn tất:

```shell
kubectl rollout status deployment/nginx-deployment
```

## Cấu hình chiến lược rolling update (Configuring rolling update strategy)

Deployment hỗ trợ hai
[loại chiến lược cập nhật](63-deployment-vi.md#strategy):

- **RollingUpdate** (mặc định): thay thế dần các Pod cũ bằng các Pod mới.
- **Recreate**: chấm dứt toàn bộ Pod hiện có trước khi tạo Pod mới. Cách này
  gây gián đoạn (downtime).

Với chiến lược RollingUpdate, các tham số sau kiểm soát cách Kubernetes thực hiện cập nhật:

| Tham số | Kiểm soát | Mặc định | Ví dụ |
|-----------|----------|---------|---------|
| `maxUnavailable` | Số Pod tối đa có thể không khả dụng trong quá trình cập nhật | 25% | `1` hoặc `25%` |
| `maxSurge` | Số Pod dư tối đa có thể được tạo thêm trong quá trình cập nhật | 25% | `1` hoặc `25%` |

> **Ghi chú:**
> `maxUnavailable` và `maxSurge` chấp nhận một số tuyệt đối hoặc một tỷ lệ phần trăm.
> Kubernetes tính phần trăm dựa trên số replica mong muốn, làm tròn xuống
> với `maxUnavailable` và làm tròn lên với `maxSurge`.

Để cấu hình các tham số này, dùng `kubectl patch`:

```shell
kubectl patch deployment nginx-deployment -p \
  '{"spec":{"strategy":{"rollingUpdate":{"maxUnavailable":"25%","maxSurge":"25%"}}}}'
```

Bạn cũng có thể đặt các trường này trong manifest của Deployment tại
`.spec.strategy.rollingUpdate`. Để xem các ví dụ chi tiết, hãy xem
[max unavailable](63-deployment-vi.md#max-unavailable)
và [max surge](63-deployment-vi.md#max-surge)
trong tài liệu khái niệm về Deployment.

### Phát hiện một rollout bị đình trệ (Detecting a stalled rollout)

Nếu một rollout không có tiến triển trong khoảng thời gian được chỉ định bởi
`.spec.progressDeadlineSeconds` (mặc định: 600 giây), Kubernetes đánh dấu condition
`Progressing` của Deployment là `False`. Bạn có thể kiểm tra condition này bằng cách
describe Deployment:

```shell
kubectl describe deployment nginx-deployment
```

Tìm condition `Progressing` trong phần `Conditions` của kết quả. Một rollout bị đình trệ
thường cho thấy các Pod mới đang không khởi động được. Phần `Events` của kết quả có thể
giúp chẩn đoán vấn đề.

## Rollback về một revision trước đó (Rolling back to a previous revision) {#rollback}

Nếu phiên bản mới gây ra sự cố, bạn có thể rollback về một revision trước đó.

### Xem lịch sử rollout (Viewing rollout history)

```shell
kubectl rollout history deployment/nginx-deployment
```

Kết quả tương tự như sau:

```
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

> **Ghi chú:**
> Cột `CHANGE-CAUSE` hiển thị giá trị của annotation `kubernetes.io/change-cause`
> tại thời điểm của từng revision. Annotation này **không** được đặt tự động,
> nhưng nếu bạn đang dùng một giải pháp tự động hóa để quản lý Deployment, công cụ
> bạn dùng có thể ghi một đoạn văn bản vào annotation đó.

### Rollback về revision liền trước (Rolling back to the previous revision)

```shell
kubectl rollout undo deployment/nginx-deployment
```

Kết quả tương tự như sau:

```
deployment.apps/nginx-deployment rolled back
```

### Rollback về một revision cụ thể (Rolling back to a specific revision)

```shell
kubectl rollout undo deployment/nginx-deployment --to-revision=1
```

Xác minh rằng rollback hoàn tất:

```shell
kubectl rollout status deployment/nginx-deployment
```

> **Ghi chú:**
> Lịch sử revision của một Deployment được lưu trong các ReplicaSet mà nó kiểm soát.
> Theo mặc định, Kubernetes giữ lại 10 ReplicaSet cũ. Bạn có thể thay đổi giới hạn này
> bằng cách đặt `.spec.revisionHistoryLimit` trong manifest của Deployment. Đặt nó
> thành `0` sẽ vô hiệu hóa hoàn toàn khả năng rollback.

## Dọn dẹp (Cleaning up)

Xóa Deployment:

```shell
kubectl delete deployment nginx-deployment
```

## Tiếp theo (What's next)

- Tìm hiểu thêm về [Deployment](63-deployment-vi.md).
- Tìm hiểu cách [scale một Deployment thủ công](346-scale-deployment-vi.md).
- Thực hành theo [Horizontal Pod Autoscaling](342-hpa-walkthrough-vi.md).
- Xem cách [thực hiện rolling update trên một DaemonSet](388-update-daemon-set-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ để mở lab của nhóm 4a:

1. Bạn đổi `.spec.replicas` từ 2 lên 4, rồi đổi image trong `.spec.template`. Thao tác nào kích
   hoạt rolling update, thao tác nào không? Bài lấy mốc phân biệt ở trường nào?
2. **Câu bẫy.** Bạn chạy `kubectl rollout pause`, rồi hai lệnh `kubectl set image` liên tiếp với
   hai tag khác nhau. Cluster chạy hai rollout, một rollout, hay không rollout nào? Điều gì xảy
   ra khi bạn `kubectl rollout resume`?
3. Một Deployment 4 replica trải trên `lab-k8s-worker1` và `lab-k8s-worker2`, để mặc định
   `maxUnavailable` và `maxSurge`. Hai giá trị mặc định là bao nhiêu, phần trăm đó tính trên cái
   gì, và bài quy định làm tròn theo hướng nào cho từng cái?
4. Rollout đứng im vì các Pod mới không khởi động được. Kubernetes ghi nhận chuyện đó ở đâu, sau
   mốc thời gian nào, và bạn đọc nó bằng lệnh nào?
5. Lịch sử revision của một Deployment thực chất được lưu ở đâu, mặc định giữ bao nhiêu bản, và
   đặt `.spec.revisionHistoryLimit` bằng `0` thì mất khả năng gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Đổi image **có**, đổi `replicas` **không**. Bài nói thẳng: **bất kỳ thay đổi nào đối với trường
   `.spec.template` đều kích hoạt một rolling update**. `.spec.replicas` nằm ngoài
   `.spec.template` nên chỉ là thay đổi số lượng Pod, Kubernetes không tạo Pod mới với cấu hình
   khác.
2. **Không rollout nào chạy trong lúc tạm dừng.** Đây là chỗ dễ tưởng mỗi `set image` là một
   rollout. Bài ghi rõ: những thay đổi thực hiện khi rollout đang tạm dừng **không kích hoạt
   rollout mới cho tới khi bạn resume**, và **Kubernetes áp dụng tất cả thay đổi cùng lúc** khi
   resume. Nên khi `resume`, bạn có **một** rollout duy nhất, đi thẳng tới tag cuối cùng. Đó cũng
   chính là công dụng bài nêu cho `pause`: gom nhiều thay đổi vào một lần rollout.
3. Cả hai mặc định là **25%**. Phần trăm được **tính trên số replica mong muốn**, và cách làm tròn
   khác nhau: **`maxUnavailable` làm tròn xuống**, **`maxSurge` làm tròn lên**. Hai trường này
   nhận số tuyệt đối hoặc phần trăm, đặt ở `.spec.strategy.rollingUpdate` trong manifest hoặc
   bằng `kubectl patch`.
4. Nếu rollout **không có tiến triển** trong khoảng thời gian của `.spec.progressDeadlineSeconds`
   (**mặc định 600 giây**), Kubernetes đánh dấu condition **`Progressing` của Deployment là
   `False`**. Đọc bằng `kubectl describe deployment <tên>`, tìm condition `Progressing` trong
   phần `Conditions`; phần `Events` của cùng output giúp chẩn đoán vì sao Pod mới không lên được.
5. Lưu **trong chính các ReplicaSet mà Deployment kiểm soát**. Mặc định Kubernetes **giữ lại 10
   ReplicaSet cũ**; đổi bằng `.spec.revisionHistoryLimit`. Đặt giá trị đó thành **`0` là vô hiệu
   hóa hoàn toàn khả năng rollback** — không còn ReplicaSet cũ nào để `rollout undo` quay về.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của nhóm
[4a. ReplicaSet, Deployment và rollout](00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout);
trả lời trôi cả năm câu thì mở
[Lab 4a — ReplicaSet, Deployment và rollout](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md)
và làm từ phần B0.

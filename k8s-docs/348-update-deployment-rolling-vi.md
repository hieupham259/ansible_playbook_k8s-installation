# Cập nhật một Deployment mà không gây gián đoạn (Update a Deployment Without Downtime)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/

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
- Xem cách [thực hiện rolling update trên một DaemonSet](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/).

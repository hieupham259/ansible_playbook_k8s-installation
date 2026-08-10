# Scale thủ công theo chiều ngang cho một Deployment (Horizontal Manual Scaling for a Deployment)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/scale-deployment/

Trang này hướng dẫn cách scale một Deployment theo chiều ngang một cách thủ công, bằng cách thay đổi
số replica của nó. Scale thủ công cho phép bạn kiểm soát trực tiếp số lượng Pod đang chạy khi tải
thay đổi có thể dự đoán trước hoặc khi cần quản lý chi phí.

Điều này khác với _scale theo chiều dọc_ (vertical scaling): giữ nguyên số replica, nhưng điều chỉnh
lượng tài nguyên cấp cho mỗi Pod.

## Mục tiêu (Objectives)

- Scale up một Deployment để xử lý nhiều lưu lượng hơn.
- Scale down một Deployment để tiết kiệm tài nguyên.
- Scale một Deployment về không (zero) để tạm dừng một workload.
- Hiểu khi nào nên dùng scale thủ công và khi nào nên dùng HorizontalPodAutoscaler.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn cần có sẵn một Deployment. Nếu bạn chưa có và chỉ muốn thực hành,
bạn có thể tạo Deployment nginx từ bài
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

## Scale up một Deployment (Scaling up a Deployment)

Có nhiều cách khác nhau để bạn thay đổi số replica cho một Deployment
đang tồn tại.

### Scale up bằng `kubectl scale` (Scaling up using `kubectl scale`)

Dùng `kubectl scale` để đặt số replica:

```shell
kubectl scale deployment/nginx-deployment --replicas=4
```

Kết quả tương tự như sau:

```
deployment.apps/nginx-deployment scaled
```

Xác minh rằng Deployment có bốn Pod:

```shell
kubectl get deployment nginx-deployment
```

Kết quả tương tự như sau:

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   4/4     4            4           1m
```

### Scale theo cách khai báo bằng `kubectl apply` (Declarative scaling using `kubectl apply`)

Thay vì chạy một lệnh mệnh lệnh (imperative), bạn có thể cập nhật file manifest rồi
apply nó. Cách tiếp cận này phù hợp với các quy trình quản lý cấu hình
bằng hệ thống quản lý phiên bản (version control).

Lưu cấu hình Deployment hiện tại ra một file cục bộ:

```shell
kubectl get deployment nginx-deployment -o yaml > /tmp/nginx-deployment.yaml
```

Chỉnh sửa `/tmp/nginx-deployment.yaml` và đổi `.spec.replicas` thành `4`.

Trước khi apply, hãy so sánh các thay đổi cục bộ của bạn với trạng thái trên cluster:

```shell
kubectl diff -f /tmp/nginx-deployment.yaml
```

Apply manifest đã chỉnh sửa:

```shell
kubectl apply -f /tmp/nginx-deployment.yaml
```

## Scale down một Deployment (Scaling down a Deployment)

Để giảm số lượng Pod, đặt `--replicas` thành một giá trị thấp hơn:

```shell
kubectl scale deployment/nginx-deployment --replicas=2
```

Kubernetes chấm dứt (terminate) các Pod dư thừa một cách êm thấm (gracefully), tôn trọng
thiết lập `terminationGracePeriodSeconds` của từng Pod.

Xác minh rằng Deployment có hai Pod:

```shell
kubectl get pods -l app=nginx
```

Kết quả tương tự như sau:

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-66b6c48dd5-7gl6h   1/1     Running   0          2m
nginx-deployment-66b6c48dd5-v8mkd   1/1     Running   0          2m
```

## Scale về không (Scaling to zero)

Bạn có thể scale một Deployment về không để tạm dừng workload mà không cần
xóa chính Deployment đó:

```shell
kubectl scale deployment/nginx-deployment --replicas=0
```

Xác minh rằng không có Pod nào đang chạy:

```shell
kubectl get deployment nginx-deployment
```

Kết quả tương tự như sau:

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   0/0     0            0           5m
```

> **Ghi chú:**
> Scale về không sẽ xóa toàn bộ Pod nhưng vẫn giữ lại Deployment và ReplicaSet
> của nó. Bạn có thể scale trở lại bất cứ lúc nào bằng cách đặt `--replicas`
> thành một số dương.

Các trường hợp sử dụng phổ biến của việc scale về không bao gồm:

- Tạm dừng một workload để tiết kiệm tài nguyên
- Các khoảng thời gian gỡ lỗi (debug) hoặc bảo trì
- Kiểm soát chi phí trong môi trường development hoặc staging

## Các cách khác để thay đổi số replica (Other ways to change the replica count)

Bên cạnh `kubectl scale`, bạn có thể thay đổi `.spec.replicas` bằng
`kubectl edit` hoặc `kubectl patch`.

### Scale bằng `kubectl edit` (Scale using `kubectl edit`)

```shell
kubectl edit deployment nginx-deployment
```

Thay đổi trường `.spec.replicas` trong trình soạn thảo, sau đó lưu và thoát.

### Scale bằng `kubectl patch` (Scale using `kubectl patch`)

Bạn có thể cập nhật `.spec.replicas` bằng một strategic merge patch:

```shell
kubectl patch deployment nginx-deployment -p '{"spec":{"replicas":4}}'
```

Khi viết script, hãy dùng JSON patch kèm một phép kiểm tra điều kiện tiên quyết. Lệnh sau
đặt số replica thành 4, nhưng chỉ khi số hiện tại là 2:

```shell
kubectl patch deployment nginx-deployment --type=json -p='[
  {"op": "test", "path": "/spec/replicas", "value": 2},
  {"op": "replace", "path": "/spec/replicas", "value": 4}
]'
```

Thao tác `test` khiến patch thất bại nếu giá trị hiện tại không khớp, giúp ngăn ngừa
các thay đổi ngoài ý muốn khi nhiều người hoặc nhiều script cùng sửa một Deployment.

## Khi nào dùng scale thủ công so với scale tự động (When to use manual versus automatic scaling)

| Khía cạnh | Scale thủ công | Scale tự động (HPA) |
|--------|---------------|------------------------|
| Phù hợp nhất với | Tải thay đổi có thể dự đoán, theo lịch, hoặc một lần | Nhu cầu biến động hoặc khó dự đoán |
| Cách hoạt động | Bạn đặt `.spec.replicas` trực tiếp | HPA điều chỉnh replicas dựa trên các metric quan sát được |
| Thời gian phản ứng | Ngay lập tức khi bạn chạy lệnh | Phản ứng theo metric với một độ trễ ngắn |
| Nhận biết metric | Không — bạn tự quyết định số replica | Theo dõi CPU, bộ nhớ, hoặc metric tùy chỉnh |
| Bảo trì | Cần can thiệp thủ công để điều chỉnh | Chạy tự động sau khi được cấu hình |

> **Thận trọng:**
> Nếu một HorizontalPodAutoscaler đang quản lý một Deployment, đừng đặt replicas
> thủ công. HPA liên tục điều chỉnh (reconcile) số replica và sẽ ghi đè mọi
> thay đổi thủ công.

## Dọn dẹp (Cleaning up)

Xóa Deployment:

```shell
kubectl delete deployment nginx-deployment
```

## Tiếp theo (What's next)

- Tìm hiểu thêm về [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).
- Thực hành theo [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/).
- Tìm hiểu cách [scale một StatefulSet](347-scale-stateful-set-vi.md).
- Đọc về [quản lý tài nguyên](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/).

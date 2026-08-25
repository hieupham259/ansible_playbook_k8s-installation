# Scale thủ công theo chiều ngang cho một Deployment (Horizontal Manual Scaling for a Deployment)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/scale-deployment/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 4 — Workload controller](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller)
→ [4a. ReplicaSet, Deployment và rollout](00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout),
bài 6/7 của dòng **Thực hành** · Kiểm chứng ở
[Lab 4a — ReplicaSet, Deployment và rollout](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md),
**phần B3**.

Bài ngắn và toàn bộ thao tác đều nằm trong tầm kiến thức đã có. Chỉ có bảng so sánh với
HorizontalPodAutoscaler ở gần cuối là nói tới thứ chưa học.

**Phải hiểu ở lần đọc này:**

- Scale theo chiều ngang là đổi **`.spec.replicas`**, còn scale theo chiều dọc giữ nguyên số
  replica và đổi lượng tài nguyên cấp cho mỗi Pod. Bài phân biệt hai trục này ngay ở đoạn mở đầu.
- Bốn đường đổi số replica đều ghi vào cùng một trường: `kubectl scale`, sửa manifest rồi
  `kubectl apply`, `kubectl edit`, và `kubectl patch` (mục *Scale up một Deployment* và
  *Các cách khác để thay đổi số replica*).
- Scale down **không** giết Pod ngay: Kubernetes chấm dứt các Pod dư một cách êm thấm, tôn trọng
  `terminationGracePeriodSeconds` của từng Pod (mục *Scale down một Deployment*).
- Scale về `0` xóa toàn bộ Pod nhưng **giữ lại Deployment và ReplicaSet** của nó, nên đặt
  `--replicas` về một số dương là workload chạy lại (mục *Scale về không*).
- Khi viết script, dùng JSON patch kèm phép `test` trên `/spec/replicas`: patch thất bại nếu giá
  trị hiện tại không khớp, ngăn thay đổi ngoài ý muốn khi nhiều người hoặc nhiều script cùng sửa
  một Deployment (mục *Scale bằng `kubectl patch`*).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bảng *Khi nào dùng scale thủ công so với scale tự động* và cảnh báo "HPA đang quản thì đừng đặt replicas thủ công" | HPA điều chỉnh replica theo metric quan sát được, mà nguồn metric là metrics-server của giai đoạn 11 | **nợ #1** trong [Sổ nợ lộ trình](00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình): lý thuyết ở bài [72](72-horizontal-pod-autoscale-vi.md), thực hành ở [Lab 11b](labs/LAB-11B-HPA-VA-VPA.md) |
| `kubectl diff -f` trước khi apply, trong mục *Scale theo cách khai báo bằng `kubectl apply`* | thuộc mạch quản lý object theo kiểu khai báo, không phải chuyện scale | bài [319](319-declarative-config-vi.md) cùng dòng **Thực hành** của nhóm 4a |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | cluster lab ba VM đã dựng sẵn từ trước | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Link *Horizontal Pod Autoscaling* (bài [342](342-hpa-walkthrough-vi.md)) ở mục *Tiếp theo* | cùng lý do với dòng đầu bảng | dòng **Thực hành** của [giai đoạn 11 — Observability](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability) |

---

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

- Tìm hiểu thêm về [Deployment](63-deployment-vi.md).
- Thực hành theo [Horizontal Pod Autoscaling](342-hpa-walkthrough-vi.md).
- Tìm hiểu cách [scale một StatefulSet](347-scale-stateful-set-vi.md).
- Đọc về [quản lý tài nguyên](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 4a:

1. Bài phân biệt scale theo chiều ngang với scale theo chiều dọc thế nào? Mỗi loại đụng vào phần
   nào của một Deployment?
2. Deployment của bạn đang chạy 4 replica trải trên `lab-k8s-worker1` và `lab-k8s-worker2`. Bạn
   hạ xuống 2. Kubernetes kill ngay hai Pod dư, hay làm gì khác — và thiết lập nào trên Pod
   quyết định chuyện đó?
3. **Câu bẫy.** Bạn chạy `kubectl scale deployment/nginx-deployment --replicas=0`, rồi
   `kubectl get pods` không còn Pod nào. Vậy nó có tương đương
   `kubectl delete deployment nginx-deployment` không? Muốn workload chạy lại thì làm gì?
4. Bạn viết một script tự động đổi số replica, trong khi nhiều script khác cũng sửa cùng
   Deployment đó. Bài khuyên dùng dạng patch nào, và cơ chế nào ngăn script của bạn ghi đè thay
   đổi của người khác?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Scale **ngang** đổi **số lượng Pod** — tức trường `.spec.replicas`; scale **dọc** **giữ nguyên
   số replica** và đổi **lượng tài nguyên cấp cho mỗi Pod**. Bài này chỉ nói về trục ngang, và
   nói rõ trục ngang là để xử lý nhiều lưu lượng hơn hoặc để tiết kiệm tài nguyên khi tải giảm.
2. **Không kill ngay.** Kubernetes **chấm dứt các Pod dư một cách êm thấm**, và bài nói rõ nó
   **tôn trọng `terminationGracePeriodSeconds` của từng Pod**. Vậy thiết lập quyết định nằm trên
   chính Pod chứ không nằm ở lệnh `kubectl scale`.
3. **Không tương đương.** Đây là chỗ dễ nhầm vì cả hai đều làm `kubectl get pods` trống. Bài ghi
   trong phần Ghi chú: scale về không **xóa toàn bộ Pod nhưng vẫn giữ lại Deployment và
   ReplicaSet** của nó. `kubectl delete deployment` thì xóa hẳn object. Vì object còn nguyên nên
   muốn chạy lại chỉ cần **đặt `--replicas` thành một số dương**, không phải tạo lại từ manifest.
   Đó cũng là lý do bài liệt kê các trường hợp dùng: tạm dừng workload, gỡ lỗi hoặc bảo trì, và
   kiểm soát chi phí ở môi trường development/staging.
4. Dùng **JSON patch** (`--type=json`) kèm một phép **`test`** làm tiền kiểm — ví dụ trong bài đặt
   replicas thành 4 **chỉ khi** giá trị hiện tại là 2. Thao tác `test` khiến **cả patch thất bại
   nếu giá trị hiện tại không khớp**, nên script của bạn không âm thầm đè lên thay đổi mà người
   khác vừa thực hiện.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

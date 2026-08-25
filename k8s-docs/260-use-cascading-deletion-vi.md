# Sử dụng xóa theo tầng trong Cluster (Use Cascading Deletion in a Cluster)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 4 → nhóm [4a](00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout), bài 1/7 ·
Kiểm chứng ở [Lab 4a](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) phần B8.

Đây là vế thực hành của hai bài đã đọc ở [giai đoạn 1c](00-ALO-TRINH-ADMIN.md#1c-vòng-đời-và-cơ-chế-nền-của-object):
[30](30-owners-dependents-vi.md) (owner và dependent) và [36](36-garbage-collection-vi.md) (thu gom
rác). Ở đó bạn đọc cơ chế; ở đây bạn chọn kiểu xóa và nhìn hệ quả trên một Deployment thật.

Chú ý một câu dễ bỏ qua ở mục *Trước khi bạn bắt đầu*: mỗi kiểu xóa đều làm Deployment biến mất,
nên **phải tạo lại Deployment trước mỗi kiểu**. Bỏ bước này thì hai kiểu sau không có gì để xóa.

**Phải hiểu ở lần đọc này:**

- Mục *Kiểm tra owner reference trên các pod của bạn*: Pod do Deployment sinh ra mang
  `metadata.ownerReferences` trỏ tới **ReplicaSet**, không phải Deployment, kèm `controller: true`
  và `blockOwnerDeletion: true`. Đây chính là sợi dây mà thu gom rác đi theo khi xóa theo tầng.
- Mục *Sử dụng xóa theo tầng background*: background là **mặc định**. Bài nói thẳng rằng Kubernetes
  vẫn dùng background ngay cả khi bạn chạy lệnh mà **không** có cờ `--cascade` hay đối số
  `propagationPolicy`.
- Mục *Sử dụng xóa theo tầng foreground*: `--cascade=foreground` đặt finalizer `foregroundDeletion`
  lên chính owner. Trong output còn thấy `deletionTimestamp` — object đã bị đánh dấu xóa nhưng vẫn
  còn đó, khác hẳn output của background chỉ là một `Status` với `"status": "Success"`.
- Mục *Xóa đối tượng chủ sở hữu và bỏ lại các đối tượng phụ thuộc*: `--cascade=orphan` đặt finalizer
  `orphan`, và bài kết bằng đúng một phép kiểm chứng — `kubectl get pods -l app=nginx` cho thấy các
  Pod **vẫn đang chạy** sau khi Deployment đã biến mất.
- Ba kiểu xóa là **cùng một thứ** phát biểu bằng hai giao diện: `--cascade=foreground|background|orphan`
  của `kubectl` tương ứng với `propagationPolicy` `Foreground`/`Background`/`Orphan` trong
  `DeleteOptions` gửi thẳng tới API server.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Khối `kubectl proxy --port=8080` kèm `curl -X DELETE` lặp lại ở cả ba mục | ở đây chỉ cần đối chiếu `propagationPolicy` với `--cascade`; cách nói chuyện thẳng với API server qua proxy là chủ đề riêng | bài [379](379-http-proxy-access-api-vi.md) ở [giai đoạn 28](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes) |
| Cơ chế bên trong của finalizer, và thứ tự dọn dẹp mà controller thu gom rác thực hiện | bài này chỉ cho thấy tên finalizer trong output, không giải thích chúng hoạt động ra sao | bài [29](29-finalizers-vi.md) và [36](36-garbage-collection-vi.md) đã đọc ở [giai đoạn 1c](00-ALO-TRINH-ADMIN.md#1c-vòng-đời-và-cơ-chế-nền-của-object) |
| Mục *Trước khi bạn bắt đầu* dẫn minikube và các playground công cộng | lộ trình chạy trên cluster ba VM dựng ở Lab 00, không dùng cluster tạm hay cluster dùng chung | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |

---

Trang này hướng dẫn bạn cách chỉ định kiểu
[xóa theo tầng (cascading deletion)](36-garbage-collection-vi.md#cascading-deletion)
được sử dụng trong cluster của bạn trong quá trình thu gom rác (garbage collection).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò máy chủ control plane. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc sử dụng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn cũng cần [tạo một Deployment mẫu](345-run-stateless-application-vi.md#creating-and-exploring-an-nginx-deployment)
để thử nghiệm với các kiểu xóa theo tầng khác nhau. Bạn sẽ cần tạo lại
Deployment cho mỗi kiểu xóa.

## Kiểm tra owner reference trên các pod của bạn (Check owner references on your pods)

Kiểm tra rằng trường `ownerReferences` có mặt trên các pod của bạn:

```shell
kubectl get pods -l app=nginx --output=yaml
```

Kết quả đầu ra có một trường `ownerReferences` tương tự như sau:

```yaml
apiVersion: v1
    ...
    ownerReferences:
    - apiVersion: apps/v1
      blockOwnerDeletion: true
      controller: true
      kind: ReplicaSet
      name: nginx-deployment-6b474476c4
      uid: 4fdcd81c-bd5d-41f7-97af-3a3b759af9a7
    ...
```

## Sử dụng xóa theo tầng foreground (Use foreground cascading deletion) {#use-foreground-cascading-deletion}

Theo mặc định, Kubernetes sử dụng [xóa theo tầng background](36-garbage-collection-vi.md#background-deletion)
để xóa các đối tượng phụ thuộc (dependent) của một đối tượng. Bạn có thể chuyển sang xóa theo
tầng foreground bằng `kubectl` hoặc bằng Kubernetes API, tùy theo phiên bản Kubernetes
mà cluster của bạn đang chạy. Để kiểm tra phiên bản, hãy chạy `kubectl version`.

Bạn có thể xóa các đối tượng bằng xóa theo tầng foreground thông qua `kubectl` hoặc
Kubernetes API.

**Sử dụng kubectl**

Chạy lệnh sau:

```shell
kubectl delete deployment nginx-deployment --cascade=foreground
```

**Sử dụng Kubernetes API**

1. Khởi động một phiên proxy cục bộ:

   ```shell
   kubectl proxy --port=8080
   ```

1. Dùng `curl` để kích hoạt việc xóa:

   ```shell
   curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/deployments/nginx-deployment \
       -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
       -H "Content-Type: application/json"
   ```

   Kết quả đầu ra chứa một finalizer `foregroundDeletion`
   như sau:

   ```
   "kind": "Deployment",
   "apiVersion": "apps/v1",
   "metadata": {
       "name": "nginx-deployment",
       "namespace": "default",
       "uid": "d1ce1b02-cae8-4288-8a53-30e84d8fa505",
       "resourceVersion": "1363097",
       "creationTimestamp": "2021-07-08T20:24:37Z",
       "deletionTimestamp": "2021-07-08T20:27:39Z",
       "finalizers": [
         "foregroundDeletion"
       ]
       ...
   ```

## Sử dụng xóa theo tầng background (Use background cascading deletion) {#use-background-cascading-deletion}

1. [Tạo một Deployment mẫu](345-run-stateless-application-vi.md#creating-and-exploring-an-nginx-deployment).
1. Dùng `kubectl` hoặc Kubernetes API để xóa Deployment,
   tùy theo phiên bản Kubernetes mà cluster của bạn đang chạy. Để kiểm tra phiên bản, hãy chạy
   `kubectl version`.

Bạn có thể xóa các đối tượng bằng xóa theo tầng background thông qua `kubectl`
hoặc Kubernetes API.

Kubernetes sử dụng xóa theo tầng background theo mặc định, và vẫn làm như vậy
ngay cả khi bạn chạy các lệnh dưới đây mà không có cờ `--cascade` hay đối số
`propagationPolicy`.

**Sử dụng kubectl**

Chạy lệnh sau:

```shell
kubectl delete deployment nginx-deployment --cascade=background
```

**Sử dụng Kubernetes API**

1. Khởi động một phiên proxy cục bộ:

   ```shell
   kubectl proxy --port=8080
   ```

1. Dùng `curl` để kích hoạt việc xóa:

   ```shell
   curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/deployments/nginx-deployment \
       -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Background"}' \
       -H "Content-Type: application/json"
   ```

   Kết quả đầu ra tương tự như sau:

   ```
   "kind": "Status",
   "apiVersion": "v1",
   ...
   "status": "Success",
   "details": {
       "name": "nginx-deployment",
       "group": "apps",
       "kind": "deployments",
       "uid": "cc9eefb9-2d49-4445-b1c1-d261c9396456"
   }
   ```

## Xóa đối tượng chủ sở hữu và bỏ lại các đối tượng phụ thuộc (Delete owner objects and orphan dependents) {#set-orphan-deletion-policy}

Theo mặc định, khi bạn yêu cầu Kubernetes xóa một đối tượng, controller
cũng xóa các đối tượng phụ thuộc. Bạn có thể khiến Kubernetes *bỏ lại (orphan)* các đối tượng
phụ thuộc này bằng `kubectl` hoặc Kubernetes API, tùy theo phiên bản Kubernetes mà cluster
của bạn đang chạy. Để kiểm tra phiên bản, hãy chạy `kubectl version`.

**Sử dụng kubectl**

Chạy lệnh sau:

```shell
kubectl delete deployment nginx-deployment --cascade=orphan
```

**Sử dụng Kubernetes API**

1. Khởi động một phiên proxy cục bộ:

   ```shell
   kubectl proxy --port=8080
   ```

1. Dùng `curl` để kích hoạt việc xóa:

   ```shell
   curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/deployments/nginx-deployment \
       -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
       -H "Content-Type: application/json"
   ```

   Kết quả đầu ra chứa `orphan` trong trường `finalizers`, tương tự như sau:

   ```
   "kind": "Deployment",
   "apiVersion": "apps/v1",
   "namespace": "default",
   "uid": "6f577034-42a0-479d-be21-78018c466f1f",
   "creationTimestamp": "2021-07-09T16:46:37Z",
   "deletionTimestamp": "2021-07-09T16:47:08Z",
   "deletionGracePeriodSeconds": 0,
   "finalizers": [
     "orphan"
   ],
   ...
   ```

Bạn có thể kiểm tra rằng các Pod do Deployment quản lý vẫn đang chạy:

```shell
kubectl get pods -l app=nginx
```

## Tiếp theo (What's next)

* Tìm hiểu về [chủ sở hữu và đối tượng phụ thuộc (owners and dependents)](30-owners-dependents-vi.md) trong Kubernetes.
* Tìm hiểu về [finalizer](29-finalizers-vi.md) trong Kubernetes.
* Tìm hiểu về [thu gom rác (garbage collection)](36-garbage-collection-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4:

1. `kubectl get pods -l app=nginx --output=yaml` trên các Pod của một Deployment. Trường
   `ownerReferences` trỏ tới object thuộc `kind` nào, và hai field `controller` với
   `blockOwnerDeletion` mang giá trị gì trong output của bài?
2. **Câu bẫy.** Bạn gõ `kubectl delete deployment nginx-deployment` trống trơn, không cờ nào.
   Các Pod của Deployment đó có bị xóa không, và kiểu xóa theo tầng nào đang được dùng?
3. Trên `lab-k8s-master`, bạn xóa Deployment bằng `--cascade=orphan` trong khi Pod của nó đang chạy
   trên `lab-k8s-worker1` và `lab-k8s-worker2`. Ngay sau lệnh, `kubectl get pods -l app=nginx` cho
   thấy gì, và bài dùng chính lệnh đó để chứng minh điều gì?
4. Xóa foreground và xóa background khác nhau thế nào ở phần **output** mà API server trả về, và
   sự khác nhau đó nói gì về việc object owner còn tồn tại tới lúc nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Trỏ tới **`kind: ReplicaSet`** — chứ không phải Deployment — kèm `name` và `uid` của chính
   ReplicaSet đó. Hai field còn lại trong output đều là **`true`**: `controller: true` và
   `blockOwnerDeletion: true`. Đây là sợi dây mà thu gom rác đi theo, nên xóa theo tầng bám vào
   quan hệ Pod → ReplicaSet, không phải Pod → Deployment.
2. **Có, Pod vẫn bị xóa** — và kiểu đang dùng là **background**. Bài nói rõ Kubernetes dùng xóa theo
   tầng background **theo mặc định**, và vẫn làm như vậy ngay cả khi bạn chạy lệnh mà không có cờ
   `--cascade` hay đối số `propagationPolicy`. Chỗ dễ sai là tưởng "không ghi `--cascade` thì không
   xóa theo tầng"; muốn giữ lại dependent thì phải **nói ra** bằng `--cascade=orphan`.
3. **Các Pod vẫn đang chạy.** Deployment biến mất còn Pod thì ở lại — đó đúng là điều mục
   *Xóa đối tượng chủ sở hữu và bỏ lại các đối tượng phụ thuộc* dùng lệnh
   `kubectl get pods -l app=nginx` để chứng minh: chọn `orphan` là **bỏ lại (orphan)** các đối
   tượng phụ thuộc thay vì xóa chúng theo owner.
4. Với **foreground**, output là chính object Deployment, có `deletionTimestamp` và một finalizer
   **`foregroundDeletion`** trong `metadata.finalizers` — tức **owner vẫn còn nằm trên API** sau khi
   bạn đã yêu cầu xóa. Với **background**, output là một object **`kind: Status`** với
   `"status": "Success"` và phần `details` — không còn Deployment để in ra nữa. Nói cách khác:
   foreground giữ owner lại cho tới khi phần việc xóa dependent xong, background trả về ngay.
   Kiểu `orphan` cũng để lại finalizer, nhưng tên finalizer là **`orphan`**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

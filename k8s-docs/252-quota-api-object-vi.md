# Cấu hình Quota cho các đối tượng API (Configure Quotas for API Objects)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 25 — Quản trị tài nguyên theo namespace](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace),
bài 3/7 · Phần II không có lab riêng: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md), tự chấm bằng **Checkpoint giai đoạn 25**.

Đây là bài trả lời trực tiếp một gạch của Checkpoint giai đoạn 25: *đặt quota theo số lượng
object và chứng minh nó chặn*. Bài ngắn và chạy được trọn vẹn trên cluster lab — hãy làm thật,
đừng chỉ đọc.

**Phải hiểu ở lần đọc này:**

- Quota đối tượng API giới hạn **số lượng object** thuộc một loại được tạo trong một namespace,
  và bạn khai nó trong `spec.hard` của một ResourceQuota (đoạn mở đầu và mục *Tạo một
  ResourceQuota*).
- Đọc được `status` của ResourceQuota: nó có hai khối `hard` (trần) và `used` (đã dùng), lấy
  bằng `kubectl get resourcequota <tên> --output=yaml` (mục *Tạo một ResourceQuota*).
- Đọc đúng thông báo từ chối ở mục *Thử tạo PersistentVolumeClaim thứ hai*: `... is forbidden:
  exceeded quota: object-quota-demo, requested: ..., used: ..., limited: ...` — ba con số
  `requested`/`used`/`limited` là ba thứ khác nhau, và object thứ hai **không được tạo**.
- Quota đếm **object**, không đếm mức sử dụng thật: PVC đầu tiên đứng ở trạng thái `Pending` mà
  thông báo lỗi vẫn ghi `used: persistentvolumeclaims=1` (đối chiếu mục *Tạo một
  PersistentVolumeClaim* với mục *Thử tạo PersistentVolumeClaim thứ hai*).
- Bảng chuỗi định danh ở mục *Ghi chú*: ngoài các chuỗi phẳng (`pods`, `services`, `secrets`,
  `configmaps`, `persistentvolumeclaims`, …) còn có hai chuỗi **có dấu chấm** nhắm vào **loại
  Service cụ thể**: `services.nodeports` và `services.loadbalancers`. Đặt `"0"` nghĩa là cấm
  hoàn toàn loại đó, như ví dụ trong bài.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và ba playground trực tuyến | cluster lab của bạn đã có đủ hai worker; lộ trình không dùng minikube hay cluster dùng chung | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) đã dựng đúng môi trường bài này cần |
| `storageClassName: manual` và việc PVC đứng mãi ở `Pending` | ở đây chỉ cần thấy PVC vẫn bị tính vào quota; cơ chế binding PVC ↔ PV và StorageClass là chuyện khác | bài [92 — PersistentVolume](92-persistent-volumes-vi.md) và [Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md) ở [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) — đã học |
| Nhánh *Dành cho nhà phát triển ứng dụng* trong mục *Tiếp theo* | đó là góc nhìn người viết ứng dụng (gán request/limit cho container, QoS), không phải góc nhìn quản trị namespace | các bài thực hành của [nhóm 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) và [Lab 3c](labs/LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) — đã học |

---

Trang này hướng dẫn cách cấu hình quota cho các đối tượng API, bao gồm
PersistentVolumeClaim và Service. Một quota giới hạn số lượng
đối tượng thuộc một loại cụ thể có thể được tạo trong một namespace.
Bạn chỉ định quota trong một đối tượng
[ResourceQuota](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#resourcequota-v1-core).

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

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được
cô lập khỏi phần còn lại của cluster.

```shell
kubectl create namespace quota-object-example
```

## Tạo một ResourceQuota (Create a ResourceQuota)

Dưới đây là file cấu hình cho một đối tượng ResourceQuota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota-demo
spec:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
```

Tạo ResourceQuota:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-objects.yaml --namespace=quota-object-example
```

Xem thông tin chi tiết về ResourceQuota:

```shell
kubectl get resourcequota object-quota-demo --namespace=quota-object-example --output=yaml
```

Kết quả cho thấy trong namespace quota-object-example, có thể có tối đa
một PersistentVolumeClaim, tối đa hai Service loại LoadBalancer, và không có Service
loại NodePort nào.

```yaml
status:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
  used:
    persistentvolumeclaims: "0"
    services.loadbalancers: "0"
    services.nodeports: "0"
```

## Tạo một PersistentVolumeClaim (Create a PersistentVolumeClaim)

Dưới đây là file cấu hình cho một đối tượng PersistentVolumeClaim:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-quota-demo
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

Tạo PersistentVolumeClaim:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-objects-pvc.yaml --namespace=quota-object-example
```

Xác minh rằng PersistentVolumeClaim đã được tạo:

```shell
kubectl get persistentvolumeclaims --namespace=quota-object-example
```

Kết quả cho thấy PersistentVolumeClaim tồn tại và có trạng thái Pending:

```
NAME             STATUS
pvc-quota-demo   Pending
```

## Thử tạo PersistentVolumeClaim thứ hai (Attempt to create a second PersistentVolumeClaim)

Dưới đây là file cấu hình cho PersistentVolumeClaim thứ hai:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-quota-demo-2
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
```

Thử tạo PersistentVolumeClaim thứ hai:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-objects-pvc-2.yaml --namespace=quota-object-example
```

Kết quả cho thấy PersistentVolumeClaim thứ hai không được tạo,
vì nó sẽ vượt quá quota của namespace.

```
persistentvolumeclaims "pvc-quota-demo-2" is forbidden:
exceeded quota: object-quota-demo, requested: persistentvolumeclaims=1,
used: persistentvolumeclaims=1, limited: persistentvolumeclaims=1
```

## Ghi chú (Notes)

Đây là các chuỗi được dùng để định danh những tài nguyên API có thể bị ràng buộc
bởi quota:

<table>
<tr><th>Chuỗi</th><th>Đối tượng API</th></tr>
<tr><td>"pods"</td><td>Pod</td></tr>
<tr><td>"services"</td><td>Service</td></tr>
<tr><td>"replicationcontrollers"</td><td>ReplicationController</td></tr>
<tr><td>"resourcequotas"</td><td>ResourceQuota</td></tr>
<tr><td>"secrets"</td><td>Secret</td></tr>
<tr><td>"configmaps"</td><td>ConfigMap</td></tr>
<tr><td>"persistentvolumeclaims"</td><td>PersistentVolumeClaim</td></tr>
<tr><td>"services.nodeports"</td><td>Service loại NodePort</td></tr>
<tr><td>"services.loadbalancers"</td><td>Service loại LoadBalancer</td></tr>
</table>

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace quota-object-example
```

## Tiếp theo (What's next)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình Memory Request và Limit mặc định cho một Namespace](232-memory-default-namespace-vi.md)

* [Cấu hình CPU Request và Limit mặc định cho một Namespace](230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc Memory tối thiểu và tối đa cho một Namespace](231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](229-cpu-constraint-namespace-vi.md)

* [Cấu hình Quota Memory và CPU cho một Namespace](233-quota-memory-cpu-namespace-vi.md)

* [Cấu hình Quota số lượng Pod cho một Namespace](234-quota-pod-namespace-vi.md)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên Memory cho Container và Pod](264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU cho Container và Pod](263-assign-cpu-resource-vi.md)

* [Gán tài nguyên CPU và memory ở mức Pod](265-assign-pod-level-resources-vi.md)

* [Cấu hình Quality of Service cho Pod](288-quality-service-pod-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 25:

1. `spec.hard` của `object-quota-demo` có ba dòng: `persistentvolumeclaims: "1"`,
   `services.loadbalancers: "2"`, `services.nodeports: "0"`. Namespace đó được phép tạo chính xác
   những gì? Vì sao hai dòng sau phải viết chuỗi **có dấu chấm** thay vì chỉ `services`?
2. Đọc kỹ thông báo `persistentvolumeclaims "pvc-quota-demo-2" is forbidden: exceeded quota:
   object-quota-demo, requested: persistentvolumeclaims=1, used: persistentvolumeclaims=1,
   limited: persistentvolumeclaims=1`. Mỗi con số `requested`, `used`, `limited` nói điều gì — và
   sau thông báo này, `pvc-quota-demo-2` có tồn tại trong cluster không?
3. **Câu bẫy.** `pvc-quota-demo` đang ở trạng thái `Pending`: nó chưa Bound vào PersistentVolume
   nào và chưa chiếm một byte dung lượng nào. Vậy nó có bị tính vào quota không? Chỗ nào trong
   bài chứng minh câu trả lời của bạn?
4. Trên `lab-k8s-master`, bạn muốn namespace `dev` không được tạo bất kỳ Service NodePort nào và
   giữ tối đa 5 Secret — theo bảng ở mục *Ghi chú*, `spec.hard` phải gồm hai dòng nào? Và nếu bạn
   làm lại đúng bài này trên cluster lab nhưng bỏ `storageClassName: manual` để PVC dùng
   StorageClass mặc định `local-path` (nên nó Bound ngay thay vì đứng `Pending`), bước tạo PVC
   thứ hai có còn bị chặn không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Namespace đó được phép có **tối đa một PersistentVolumeClaim, tối đa hai Service loại
   LoadBalancer, và không Service loại NodePort nào** — `"0"` là cấm hoàn toàn. Phải dùng chuỗi
   có dấu chấm vì bảng ở mục *Ghi chú* định danh **từng loại Service riêng**:
   `services.nodeports` ứng với Service loại NodePort, `services.loadbalancers` ứng với Service
   loại LoadBalancer, còn `services` là chuỗi của Service nói chung. Muốn nhắm vào một loại thì
   phải gọi đúng chuỗi của loại đó.
2. `requested=1` là **số object mà request này xin thêm**; `used=1` là **số object namespace đã
   dùng** (chính là `pvc-quota-demo` đang có); `limited=1` là **trần `hard` đang áp**. Vì
   `used + requested` vượt `limited` nên request bị từ chối. Sau thông báo đó, **`pvc-quota-demo-2`
   không tồn tại**: bài nói rõ PersistentVolumeClaim thứ hai *không được tạo*.
3. **Có, nó vẫn bị tính.** Bằng chứng nằm ngay trong thông báo lỗi của PVC thứ hai:
   `used: persistentvolumeclaims=1` — trong khi `pvc-quota-demo` vẫn đang `Pending`. Trực giác
   "chưa cấp phát gì thì chưa tiêu tốn quota" sai ở chỗ nhầm hai loại quota: quota trong bài này
   **giới hạn số lượng đối tượng thuộc một loại được tạo trong namespace**, nên nó đếm ngay khi
   object được ghi vào API, bất kể object đó đã hoạt động hay chưa.
4. Hai dòng: `services.nodeports: "0"` và `secrets: "5"`. Phần sau: **vẫn bị chặn**. Việc PVC đầu
   tiên Bound hay Pending không liên quan — quota đếm số object `persistentvolumeclaims` trong
   namespace, mà con số đó vẫn là 1, nên PVC thứ hai vẫn vượt trần `limited: 1`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

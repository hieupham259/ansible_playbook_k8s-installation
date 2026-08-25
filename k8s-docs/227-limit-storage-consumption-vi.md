# Giới hạn mức tiêu thụ lưu trữ (Limit Storage Consumption)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/limit-storage-consumption/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 26 — Vận hành lưu trữ](00-ALO-TRINH-ADMIN.md#giai-đoạn-26--vận-hành-lưu-trữ),
bài 4/4 · Phần II không có lab riêng: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md), tự chấm bằng **Checkpoint giai đoạn 26**.

Bài này nối hai mạch bạn vừa học: quota theo namespace ở
[giai đoạn 25](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace) — bài
[242](242-namespaces-tasks-vi.md) và [252](252-quota-api-object-vi.md) — với lưu trữ. Bài rất ngắn
và **chạy được gần trọn vẹn** trên cluster lab: bạn đã có StorageClass `local-path` từ
[Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md), nên chỉ cần một namespace mới, hai manifest trong
bài, và vài PVC là dựng lại được đúng các tình huống từ chối.

**Phải hiểu ở lần đọc này:**

- Ba thứ mà mục *Kịch bản* muốn giới hạn, và **cơ chế nào lo thứ nào**: số lượng PVC trong
  namespace và tổng dung lượng tích lũy do ResourceQuota lo; lượng lưu trữ mà **mỗi claim** được
  xin do LimitRange lo.
- LimitRange kiểu `type: PersistentVolumeClaim` với `max` và `min` `storage`: admission controller
  từ chối mọi PVC **vượt trên hoặc xuống dưới** khoảng đó — ví dụ trong bài, PVC xin `10Gi` bị từ
  chối vì trần là `2Gi` (mục *LimitRange để giới hạn request lưu trữ*).
- ResourceQuota với `persistentvolumeclaims: "5"` và `requests.storage: "5Gi"` đặt **hai trần cùng
  lúc**: một đếm **số lượng** claim, một đo **tổng dung lượng**. PVC mới vượt **một trong hai** là
  bị từ chối (mục *ResourceQuota để giới hạn số lượng PVC và tổng dung lượng lưu trữ tích lũy*).
- Hai trần đó **giao nhau chứ không cộng dồn**: với trần mỗi PVC `2Gi` và quota tổng `5Gi` thì ba
  PVC `2Gi` là không thể, vì thành `6Gi` cho một namespace bị chặn ở `5Gi`. Trần của từng claim
  không suy ra được trần tổng, và ngược lại (cùng mục).
- Kết luận ở mục *Tóm tắt*, đúng cặp khái niệm mà Checkpoint giai đoạn 25 bắt phân biệt: limit
  range đặt trần cho **lượng lưu trữ được yêu cầu** ở từng claim, còn resource quota chặn **mức
  tiêu thụ của cả namespace** thông qua số lượng claim và tổng dung lượng tích lũy.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và ba playground trực tuyến | cluster lab đã có sẵn ba VM; lộ trình không dùng minikube hay cluster dùng chung | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Ví dụ "volume AWS EBS có yêu cầu tối thiểu là 1Gi" | ở đây chỉ cần biết `min` tồn tại để tôn trọng ràng buộc của nhà cung cấp lưu trữ bên dưới; cluster lab dùng `local-path`, không phải volume của cloud provider | bài [92 — PersistentVolume](92-persistent-volumes-vi.md) và [Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md) ở [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) — đã học |
| Chữ *LimitRange* ở đầu bài trỏ sang trang [232](232-memory-default-namespace-vi.md) | trang đó nói về memory mặc định cho namespace, không phải LimitRange cho storage | cách viết LimitRange đầy đủ ở bài [133](133-limit-range-vi.md), [nhóm 7b](00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên) và [Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) — đã học |

---

Ví dụ này minh họa cách giới hạn lượng lưu trữ (storage) được tiêu thụ trong một namespace.

Các tài nguyên sau được sử dụng trong phần minh họa:
[ResourceQuota](134-resource-quotas-vi.md),
[LimitRange](232-memory-default-namespace-vi.md),
và [PersistentVolumeClaim](92-persistent-volumes-vi.md).

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
  tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
  không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
  bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
  các sân chơi (playground) Kubernetes sau:

  * [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  * [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  * [KodeKloud](https://kodekloud.com/public-playgrounds)

  Kubernetes server của bạn phải ở phiên bản v1.36 hoặc mới hơn. Để kiểm tra phiên bản, nhập
  lệnh `kubectl version`.

## Kịch bản: Giới hạn mức tiêu thụ lưu trữ (Scenario: Limiting Storage Consumption)

Người quản trị cluster (cluster-admin) đang vận hành một cluster phục vụ một cộng đồng người
dùng, và người quản trị muốn kiểm soát lượng lưu trữ mà một namespace đơn lẻ có thể tiêu thụ
nhằm kiểm soát chi phí.

Người quản trị muốn giới hạn:

1. Số lượng persistent volume claim trong một namespace
2. Lượng lưu trữ mà mỗi claim có thể yêu cầu
3. Tổng lượng lưu trữ tích lũy mà namespace có thể có

## LimitRange để giới hạn request lưu trữ (LimitRange to limit requests for storage) {#limitrange-to-limit-requests-for-storage}

Việc thêm một `LimitRange` vào một namespace sẽ ràng buộc kích thước request lưu trữ vào một
mức tối thiểu và tối đa. Lưu trữ được yêu cầu thông qua `PersistentVolumeClaim`. Admission
controller thực thi limit range sẽ từ chối bất kỳ PVC nào vượt trên hoặc dưới các giá trị mà
người quản trị đã đặt.

Trong ví dụ này, một PVC yêu cầu 10Gi lưu trữ sẽ bị từ chối vì nó vượt quá mức tối đa 2Gi.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: storagelimits
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 2Gi
    min:
      storage: 1Gi
```

Request lưu trữ tối thiểu được dùng khi nhà cung cấp lưu trữ bên dưới yêu cầu những mức tối
thiểu nhất định. Ví dụ, các volume AWS EBS có yêu cầu tối thiểu là 1Gi.

## ResourceQuota để giới hạn số lượng PVC và tổng dung lượng lưu trữ tích lũy (ResourceQuota to limit PVC count and cumulative storage capacity)

Người quản trị có thể giới hạn số lượng PVC trong một namespace cũng như tổng dung lượng tích
lũy của các PVC đó. Các PVC mới vượt quá một trong hai giá trị tối đa sẽ bị từ chối.

Trong ví dụ này, PVC thứ 6 trong namespace sẽ bị từ chối vì nó vượt quá số lượng tối đa là 5.
Mặt khác, một quota tối đa 5Gi khi kết hợp với giới hạn tối đa 2Gi ở trên sẽ không thể có 3
PVC mà mỗi cái là 2Gi. Vì như vậy sẽ là 6Gi được yêu cầu cho một namespace bị chặn ở mức 5Gi.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storagequota
spec:
  hard:
    persistentvolumeclaims: "5"
    requests.storage: "5Gi"
```

## Tóm tắt (Summary)

Một limit range có thể đặt mức trần cho lượng lưu trữ được yêu cầu, trong khi một resource
quota có thể chặn hiệu quả lượng lưu trữ mà một namespace tiêu thụ thông qua số lượng claim
và tổng dung lượng lưu trữ tích lũy. Điều này cho phép người quản trị cluster hoạch định ngân
sách lưu trữ của cluster mà không lo bất kỳ dự án nào vượt quá phần được cấp của họ.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 26:

1. Mục *Kịch bản* liệt kê ba thứ người quản trị muốn giới hạn. Kể lại ba thứ đó và nói rõ thứ nào
   do LimitRange lo, thứ nào do ResourceQuota lo.
2. **Câu bẫy.** Namespace đang có LimitRange `max: storage 2Gi` và ResourceQuota
   `persistentvolumeclaims: "5"`, `requests.storage: "5Gi"`. Người dùng tạo ba PVC, mỗi cái đúng
   `2Gi` — từng cái đều nằm trong trần `2Gi`, và mới chỉ là claim thứ 3 trong số 5 được phép. Cả
   ba có tạo được không?
3. Trên `lab-k8s-master`, bạn tạo một namespace mới, áp đúng hai manifest của bài vào đó (nhớ
   StorageClass mặc định của cluster lab là `local-path` từ Lab 6a), rồi tạo một PVC xin `500Mi`.
   PVC đó được chấp nhận hay bị từ chối, và **cơ chế nào** ra quyết định?
4. Theo mục *Tóm tắt*: limit range và resource quota đặt trần cho hai đối tượng khác nhau — mỗi
   cái đặt trần cho cái gì? Và dòng `persistentvolumeclaims: "5"` thuộc kiểu trần nào: đếm số
   lượng object hay đo dung lượng?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Ba thứ: **(1)** số lượng persistent volume claim trong một namespace, **(2)** lượng lưu trữ mà
   **mỗi claim** có thể yêu cầu, **(3)** tổng lượng lưu trữ tích lũy mà namespace có thể có.
   **LimitRange lo (2)** — nó ràng buộc kích thước request lưu trữ của từng PVC vào khoảng
   `min`–`max`. **ResourceQuota lo (1) và (3)** — `persistentvolumeclaims` cho số lượng,
   `requests.storage` cho tổng dung lượng tích lũy.
2. **Không — PVC thứ ba bị từ chối.** Ba claim `2Gi` là `6Gi`, trong khi namespace bị chặn ở
   `5Gi`. Chỗ dễ sai là nghĩ "mỗi cái đều hợp lệ thì cả bộ cũng hợp lệ": hai trần này **giao
   nhau**, mỗi trần được kiểm riêng, và **vượt bất kỳ trần nào cũng đủ để bị từ chối**. Đúng như
   bài nói, quota tối đa `5Gi` kết hợp với giới hạn tối đa `2Gi` thì không thể có 3 PVC mỗi cái
   `2Gi`.
3. **Bị từ chối** — và không phải vì quota. LimitRange trong bài đặt `min: storage 1Gi`, mà
   admission controller thực thi limit range từ chối mọi PVC **vượt trên hoặc xuống dưới** khoảng
   người quản trị đặt. `500Mi` nằm dưới `min`, nên chính **LimitRange** chặn nó, trong khi
   ResourceQuota chưa hề bị chạm tới (mới 0 claim, 0 dung lượng).
4. **Limit range đặt trần cho lượng lưu trữ được yêu cầu ở từng claim; resource quota chặn mức
   tiêu thụ của cả namespace** — thông qua số lượng claim và tổng dung lượng lưu trữ tích lũy.
   Dòng `persistentvolumeclaims: "5"` là trần **đếm số lượng object**, không đo dung lượng; phần
   đo dung lượng là `requests.storage`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của giai đoạn 26: trả
lời xong thì chuyển sang **Checkpoint** ở cuối
[Giai đoạn 26 — Vận hành lưu trữ](00-ALO-TRINH-ADMIN.md#giai-đoạn-26--vận-hành-lưu-trữ) và làm nó
trên cluster lab.

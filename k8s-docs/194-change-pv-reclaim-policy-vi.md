# Thay đổi Reclaim Policy của một PersistentVolume (Change the Reclaim Policy of a PersistentVolume)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy/>
>
> Trang này hướng dẫn cách thay đổi reclaim policy của một PersistentVolume trong Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối, giai đoạn 26 — Vận hành lưu trữ](00-ALO-TRINH-ADMIN.md#giai-đoạn-26--vận-hành-lưu-trữ),
bài 2/3 · nối tiếp phần reclaim policy của [bài 92](92-persistent-volumes-vi.md); thực hành
ngay trên cluster lab sau Lab 6a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Trang task rất ngắn, chỉ một lệnh `kubectl patch` — giá trị nằm ở việc hiểu **khi nào** phải
làm thao tác này: trước khi xóa một PVC mà dữ liệu bên dưới còn quý.

**Phải hiểu ở lần đọc này:**

- PersistentVolume được **cấp phát động** có reclaim policy mặc định là **`Delete`**: người
  dùng xóa PVC là volume bị xóa tự động theo.
- Với policy **`Retain`**, xóa PVC thì PV **không bị xóa** mà chuyển sang pha **Released**, và
  toàn bộ dữ liệu của nó có thể được khôi phục thủ công.
- Thao tác đổi policy là một lệnh `kubectl patch pv` sửa
  `spec.persistentVolumeReclaimPolicy`, thực hiện được **ngay trên PV đang Bound**; kiểm chứng
  bằng cột `RECLAIMPOLICY` của `kubectl get pv`.
- Trên Windows (cmd), chuỗi JSON trong `-p` phải bọc bằng **nháy kép** và escape nháy kép bên
  trong, thay vì nháy đơn như trên bash.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Policy `Recycle` được nhắc trong danh sách | đã lỗi thời (deprecated), [bài 92](92-persistent-volumes-vi.md) đã nói | không cần |
| Mục Tham khảo trỏ vào API reference của PersistentVolume/PersistentVolumeClaim | chỉ dùng khi cần tra chính xác schema của field | tra cứu khi cần |

---

Trang này hướng dẫn cách thay đổi reclaim policy của một PersistentVolume trong Kubernetes.

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

## Vì sao phải thay đổi reclaim policy của một PersistentVolume (Why change reclaim policy of a PersistentVolume)

Các PersistentVolume có thể có nhiều reclaim policy khác nhau, bao gồm "Retain", "Recycle" và
"Delete". Với các PersistentVolume được cấp phát động (dynamically provisioned), reclaim policy
mặc định là "Delete". Điều này có nghĩa là một volume được cấp phát động sẽ tự động bị xóa khi
người dùng xóa PersistentVolumeClaim tương ứng. Hành vi tự động này có thể không phù hợp nếu
volume chứa dữ liệu quý giá. Trong trường hợp đó, dùng chính sách "Retain" sẽ phù hợp hơn. Với
chính sách "Retain", nếu người dùng xóa một PersistentVolumeClaim, PersistentVolume tương ứng
sẽ không bị xóa. Thay vào đó, nó được chuyển sang pha Released, nơi toàn bộ dữ liệu của nó có
thể được khôi phục thủ công.

## Thay đổi reclaim policy của một PersistentVolume (Changing the reclaim policy of a PersistentVolume)

1. Liệt kê các PersistentVolume trong cluster của bạn:

   ```shell
   kubectl get pv
   ```

   Kết quả tương tự như sau:

   ```none
   NAME                                       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM             STORAGECLASS     REASON    AGE
   pvc-b6efd8da-b7b5-11e6-9d58-0ed433a7dd94   4Gi        RWO           Delete          Bound     default/claim1    manual                     10s
   pvc-b95650f8-b7b5-11e6-9d58-0ed433a7dd94   4Gi        RWO           Delete          Bound     default/claim2    manual                     6s
   pvc-bb3ca71d-b7b5-11e6-9d58-0ed433a7dd94   4Gi        RWO           Delete          Bound     default/claim3    manual                     3s
   ```

   Danh sách này cũng bao gồm tên của các claim đang được bind với từng volume, giúp nhận diện
   các volume được cấp phát động dễ dàng hơn.

1. Chọn một trong các PersistentVolume của bạn và thay đổi reclaim policy của nó:

   ```shell
   kubectl patch pv <your-pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
   ```

   trong đó `<your-pv-name>` là tên PersistentVolume mà bạn chọn.

   > **Ghi chú:** Trên Windows, bạn phải dùng nháy _kép_ cho bất kỳ JSONPath template nào chứa
   > khoảng trắng (không phải nháy đơn như ví dụ trên dành cho bash). Điều này kéo theo việc
   > bạn phải dùng nháy đơn hoặc nháy kép được escape quanh các giá trị literal trong template.
   > Ví dụ:
   >
   > ```cmd
   > kubectl patch pv <your-pv-name> -p "{\"spec\":{\"persistentVolumeReclaimPolicy\":\"Retain\"}}"
   > ```

1. Kiểm tra rằng PersistentVolume bạn chọn đã có đúng policy:

   ```shell
   kubectl get pv
   ```

   Kết quả tương tự như sau:

   ```none
   NAME                                       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM             STORAGECLASS     REASON    AGE
   pvc-b6efd8da-b7b5-11e6-9d58-0ed433a7dd94   4Gi        RWO           Delete          Bound     default/claim1    manual                     40s
   pvc-b95650f8-b7b5-11e6-9d58-0ed433a7dd94   4Gi        RWO           Delete          Bound     default/claim2    manual                     36s
   pvc-bb3ca71d-b7b5-11e6-9d58-0ed433a7dd94   4Gi        RWO           Retain          Bound     default/claim3    manual                     33s
   ```

   Trong kết quả trên, bạn có thể thấy volume được bind với claim `default/claim3` có reclaim
   policy là `Retain`. Nó sẽ không tự động bị xóa khi người dùng xóa claim `default/claim3`.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [PersistentVolume](92-persistent-volumes-vi.md).
* Tìm hiểu thêm về [PersistentVolumeClaim](92-persistent-volumes-vi.md#persistentvolumeclaims).

### Tham khảo (References) {#reference}

* [PersistentVolume](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/persistent-volume-v1/)
  * Hãy chú ý tới [field](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/persistent-volume-v1/#PersistentVolumeSpec)
    `.spec.persistentVolumeReclaimPolicy` của PersistentVolume.
* [PersistentVolumeClaim](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/persistent-volume-claim-v1/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở checkpoint giai đoạn 26:

1. Trên cluster lab của bạn, một PVC được cấp phát động qua StorageClass và bạn chưa từng đụng
   tới reclaim policy. Bạn xóa PVC đó — chuyện gì xảy ra với PV và dữ liệu bên dưới?
2. Sau khi đã đổi policy sang `Retain`, người dùng xóa PVC. PV chuyển sang pha nào, và điều đó
   có ý nghĩa gì với dữ liệu?
3. Muốn đổi reclaim policy của một PV đang ở trạng thái `Bound`, bạn có phải xóa và tạo lại
   PV hay PVC không?
4. Một đồng nghiệp copy nguyên lệnh `kubectl patch pv ... -p '{...}'` (nháy đơn) trong bài vào
   cửa sổ cmd trên Windows và bị lỗi. Vì sao, và sửa thế nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **PV bị xóa tự động, dữ liệu mất theo.** Với PersistentVolume được cấp phát động, reclaim
   policy mặc định là `Delete`: xóa PersistentVolumeClaim tương ứng là volume bị xóa. Đây chính
   là lý do trang này tồn tại — phải đổi sang `Retain` **trước** khi xóa PVC nếu dữ liệu quý.
2. PV chuyển sang pha **Released** và **không bị xóa**; toàn bộ dữ liệu của nó có thể được
   **khôi phục thủ công** từ volume đó.
3. **Không.** Chỉ cần một lệnh `kubectl patch pv` sửa `spec.persistentVolumeReclaimPolicy` —
   bài thực hiện thao tác này ngay trên các PV đang `Bound` và kiểm chứng bằng cột
   `RECLAIMPOLICY`. Trực giác "đổi spec của PV đang dùng thì phải tạo lại" là sai ở đây.
4. Vì trên Windows (cmd), chuỗi JSON chứa khoảng trắng phải được bọc bằng **nháy kép**, không
   phải nháy đơn như cú pháp bash; các nháy kép bên trong phải được escape. Lệnh đúng:
   `kubectl patch pv <your-pv-name> -p "{\"spec\":{\"persistentVolumeReclaimPolicy\":\"Retain\"}}"`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

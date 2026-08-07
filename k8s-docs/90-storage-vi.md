# Lưu trữ (Storage)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/>
>
> Các cách cung cấp lưu trữ dài hạn lẫn tạm thời cho các Pod trong cluster của bạn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](LO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 1/16 · Kiểm chứng ở
Lab 6a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này **không dạy cơ chế nào cả** — nó là trang mục lục của phần Storage trên kubernetes.io.
Đọc nó mất hai phút và mục đích duy nhất là để bạn thấy trước bản đồ của cả giai đoạn 6.
Đừng cố hiểu gì thêm ở đây; mọi khái niệm được nhắc tên đều có bài riêng phía sau.

**Phải hiểu ở lần đọc này:**

- Phần Storage được chia làm hai nhánh, đúng như câu mở đầu: lưu trữ **dài hạn** và lưu trữ
  **tạm thời** cho Pod. Mọi trang trong danh sách đều rơi vào một trong hai nhánh đó.
- Nhánh dài hạn có ba trang xương sống trong danh sách: *Persistent Volume*, *Storage Class*
  và *Cấp phát volume động*. Ba trang đó là bài 3, 4, 5 của giai đoạn này.
- Nhánh tạm thời có hai trang riêng biệt và **không phải một**: *Volume tạm thời* (vòng đời
  gắn với Pod) và *Lưu trữ tạm thời cục bộ* (tài nguyên đĩa của node).
- Thứ tự đọc 16 bài lấy từ [lộ trình](LO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), **không phải**
  thứ tự trong danh sách này và cũng không phải số trong tên file.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Volume Snapshot*, *Volume Snapshot Class*, *Nhân bản volume CSI*, *Volume Populator* | cần CSI driver có hỗ trợ, cluster lab chưa có | bài 10–13 của giai đoạn này, thực hành ở Lab 6b |
| *Volume Attributes Class* | phụ thuộc API `ModifyVolume` của CSI driver | bài [97](97-volume-attributes-classes-vi.md) |
| *Dung lượng lưu trữ*, *Giới hạn volume theo từng node* | là ràng buộc lập lịch, cần đã có provisioner | bài [103](103-storage-capacity-vi.md), [104](104-storage-limits-vi.md) |
| *Giám sát tình trạng volume* | tính năng alpha của CSI | bài [105](105-volume-health-monitoring-vi.md) |
| *Lưu trữ trên Windows* | cluster lab không có node Windows | giai đoạn 15, bài [106](106-windows-storage-vi.md) |

---

Đây là trang mục lục của phần khái niệm về Lưu trữ (Storage) trong tài liệu Kubernetes.
Các cách cung cấp lưu trữ dài hạn lẫn tạm thời cho các Pod trong cluster của bạn.

---

Các trang trong mục này:

* [Volume (Volumes)](https://kubernetes.io/docs/concepts/storage/volumes/)
* [Persistent Volume (Persistent Volumes)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
* [Volume dạng projected (Projected Volumes)](./93-projected-volumes-vi.md)
* [Volume tạm thời (Ephemeral Volumes)](./94-ephemeral-volumes-vi.md)
* [Storage Class (Storage Classes)](https://kubernetes.io/docs/concepts/storage/storage-classes/)
* [Volume Attributes Class (Volume Attributes Classes)](https://kubernetes.io/docs/concepts/storage/volume-attributes-classes/)
* [Cấp phát volume động (Dynamic Volume Provisioning)](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
* [Volume Snapshot (Volume Snapshots)](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
* [Volume Snapshot Class (Volume Snapshot Classes)](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/)
* [Nhân bản volume CSI (CSI Volume Cloning)](https://kubernetes.io/docs/concepts/storage/volume-pvc-datasource/)
* [Volume Populator và nguồn dữ liệu (Volume Populators and Data Sources)](https://kubernetes.io/docs/concepts/storage/volume-populators-and-data-sources/)
* [Dung lượng lưu trữ (Storage Capacity)](https://kubernetes.io/docs/concepts/storage/storage-capacity/)
* [Giới hạn volume theo từng node (Node-specific Volume Limits)](https://kubernetes.io/docs/concepts/storage/storage-limits/)
* [Lưu trữ tạm thời cục bộ (Local ephemeral storage)](https://kubernetes.io/docs/concepts/storage/ephemeral-storage/)
* [Giám sát tình trạng volume (Volume Health Monitoring)](https://kubernetes.io/docs/concepts/storage/volume-health-monitoring/)
* [Lưu trữ trên Windows (Windows Storage)](https://kubernetes.io/docs/concepts/storage/windows-storage/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Bài này không giải thích cơ chế nào. Vậy đọc xong nó bạn đã biết cách cấp một volume cho
   Pod chưa, và bạn lấy thứ tự đọc 16 bài của giai đoạn 6 từ đâu?
2. Danh sách có cả *Volume tạm thời (Ephemeral Volumes)* lẫn *Lưu trữ tạm thời cục bộ (Local
   ephemeral storage)*. Đó là hai tên gọi của cùng một thứ hay hai khái niệm khác nhau?
3. Trong hai nhánh mà câu mở đầu nêu ra, ba trang nào là xương sống của nhánh lưu trữ dài hạn?
4. Cluster lab của bạn (1 control plane `k8s-master` + 2 worker) hiện **chưa có StorageClass và
   chưa có provisioner**. Nhìn vào danh sách, phần lớn các trang sẽ chỉ kiểm chứng được sau
   khi lab nào chạy xong?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Chưa.** Đây là trang mục lục của phần khái niệm Storage, nó chỉ liệt kê các trang con chứ
   không mô tả cơ chế nào. Thứ tự đọc lấy từ
   [lộ trình](LO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), **không phải** thứ tự trong danh sách
   này và cũng không phải số trong tên file.
2. **Hai khái niệm khác nhau**, và dấu hiệu nằm ngay trong danh sách: chúng là **hai mục
   riêng biệt**, tức hai trang tài liệu riêng. *Volume tạm thời* nói về volume có vòng đời gắn
   với Pod; *Lưu trữ tạm thời cục bộ* nói về tài nguyên đĩa của node mà kubelet quản lý.
3. **Persistent Volume, Storage Class và Cấp phát volume động.** Đó là ba trang mô tả cách
   lưu trữ dài hạn được xin, được phân loại và được tạo tự động; trong lộ trình chúng là bài
   3, 4 và 5 của giai đoạn 6.
4. **Lab 6a** — lab cài provisioner và tạo StorageClass mặc định. Trước đó cluster không có
   nơi nào để cấp phát volume, nên các trang thuộc nhánh dài hạn chỉ đọc được chứ chưa làm
   được. Phần snapshot và volume nâng cao còn phải đợi Lab 6b.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

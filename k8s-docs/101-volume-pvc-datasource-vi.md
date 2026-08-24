# Nhân bản CSI Volume (CSI Volume Cloning)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/volume-pvc-datasource/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 12/16 · Kiểm chứng ở
Lab 6b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài ngắn nhất của giai đoạn 6, và gần như toàn bộ giá trị nằm ở danh sách ràng buộc trong mục
*Giới thiệu*. Cùng nhóm nợ lab với bài [99](99-volume-snapshots-vi.md), xem
[sổ nợ lab](labs/README.md#5-sổ-nợ-lab): chỉ thực hành được nếu CSI driver hỗ trợ.

**Phải hiểu ở lần đọc này:**

- Clone là **bản sao chính xác** của một volume có sẵn, tạo bằng cách trỏ `dataSource` của PVC
  mới tới một PVC đang có; thay vì tạo volume rỗng, backend tạo một bản sao — mục *Giới thiệu*
  và *Cấp phát*.
- Các ràng buộc bắt buộc: chỉ **CSI driver**, chỉ **bộ cấp phát động**, driver có thể chưa
  triển khai chức năng này; PVC nguồn phải **đã bound và không đang được sử dụng**; nguồn và
  đích phải **cùng namespace**; và hai volume phải **cùng `volumeMode`** — mục *Giới thiệu*.
- Storage class thì **không** bị ràng buộc: volume đích dùng cùng class với nguồn hay class
  khác đều được, và có thể bỏ `storageClassName` để lấy class mặc định — mục *Giới thiệu*.
- `spec.resources.requests.storage` là **bắt buộc** và phải **bằng hoặc lớn hơn** dung lượng
  volume nguồn — ghi chú trong mục *Cấp phát*.
- Sau khi tạo xong, clone là **đối tượng độc lập**: dùng, nhân bản tiếp, chụp snapshot hay xóa
  đều không cần quan tâm tới nguồn; sửa hoặc xóa nguồn cũng không ảnh hưởng clone — mục *Sử
  dụng*.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Tên tính năng `VolumePVCDataSource` | chỉ là tên nội bộ của khả năng nhân bản | không cần |
| Cách backend thực sự tạo bản sao | tùy từng CSI driver, không có hành vi chung | Lab 6b, nếu driver hỗ trợ |

---

Tài liệu này mô tả khái niệm nhân bản (clone) các CSI Volume có sẵn trong Kubernetes.
Bạn nên làm quen trước với [Volume](91-volumes-vi.md).

## Giới thiệu (Introduction)

Tính năng Nhân bản CSI Volume (CSI Volume Cloning) bổ sung
hỗ trợ chỉ định các PVC có sẵn
trong trường `dataSource` để cho biết người dùng muốn nhân bản một volume.

Một bản nhân bản (clone) được định nghĩa là một bản sao của một Kubernetes Volume có sẵn,
có thể được sử dụng như bất kỳ Volume tiêu chuẩn nào. Điểm khác biệt duy nhất là khi
cấp phát (provision), thay vì tạo một Volume rỗng "mới", thiết bị backend
tạo một bản sao chính xác của Volume được chỉ định.

Việc triển khai tính năng nhân bản, nhìn từ góc độ Kubernetes API, bổ sung
khả năng chỉ định một PVC có sẵn làm dataSource trong quá trình tạo PVC mới.
PVC nguồn phải ở trạng thái bound và khả dụng (không đang được sử dụng).

Người dùng cần lưu ý những điểm sau khi sử dụng tính năng này:

* Hỗ trợ nhân bản (`VolumePVCDataSource`) chỉ khả dụng cho các CSI driver.
* Hỗ trợ nhân bản chỉ khả dụng cho các bộ cấp phát động (dynamic provisioner).
* CSI driver có thể đã triển khai hoặc chưa triển khai chức năng nhân bản volume.
* Bạn chỉ có thể nhân bản một PVC khi nó tồn tại trong cùng namespace với PVC đích
  (nguồn và đích phải ở trong cùng một namespace).
* Nhân bản được hỗ trợ với một Storage Class khác.
  - Volume đích có thể dùng cùng storage class với nguồn hoặc một storage class khác.
  - Có thể dùng storage class mặc định và bỏ qua storageClassName trong spec.
* Nhân bản chỉ có thể thực hiện giữa hai volume dùng cùng thiết lập VolumeMode
  (nếu bạn yêu cầu một volume ở chế độ block, nguồn CŨNG PHẢI ở chế độ block).

## Cấp phát (Provisioning)

Các bản nhân bản được cấp phát giống như bất kỳ PVC nào khác, ngoại trừ việc thêm dataSource
tham chiếu tới một PVC có sẵn trong cùng namespace.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: clone-of-pvc-1
    namespace: myns
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: cloning
  resources:
    requests:
      storage: 5Gi
  dataSource:
    kind: PersistentVolumeClaim
    name: pvc-1
```

> **Ghi chú:**
> Bạn phải chỉ định một giá trị dung lượng cho `spec.resources.requests.storage`, và
> giá trị bạn chỉ định phải bằng hoặc lớn hơn dung lượng của volume nguồn.

Kết quả là một PVC mới có tên `clone-of-pvc-1` với nội dung giống hệt
nguồn `pvc-1` đã chỉ định.

## Sử dụng (Usage)

Khi PVC mới sẵn sàng, PVC nhân bản được sử dụng giống như các PVC khác.
Cũng cần lưu ý rằng tại thời điểm này, PVC mới tạo là một đối tượng độc lập.
Nó có thể được sử dụng, nhân bản, chụp snapshot hoặc xóa một cách độc lập mà không cần
quan tâm đến PVC dataSource gốc của nó. Điều này cũng có nghĩa là nguồn
không liên kết theo bất kỳ cách nào với bản nhân bản mới tạo; nguồn cũng có thể được sửa đổi
hoặc xóa mà không ảnh hưởng đến bản nhân bản mới tạo.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. PVC nguồn là 5Gi. Bạn tạo clone và xin `storage: 2Gi` cho gọn. Được không? Còn nếu bỏ hẳn
   trường `storage` thì sao?
2. Bạn muốn clone một PVC ở namespace `app-a` sang namespace `app-b`, dùng một storage class
   khác. Bài này cho phép phần nào và cấm phần nào?
3. Clone đã tạo xong. Bạn xóa PVC nguồn. Clone có mất dữ liệu không?
4. Sau Lab 6a cluster lab của bạn sẽ có một provisioner. Bạn cần kiểm tra những điều kiện nào
   trước khi kết luận nhân bản volume dùng được, và nếu không đủ thì ghi vào đâu?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không được.** Ghi chú của bài nói rõ: bạn **phải** chỉ định giá trị cho
   `spec.resources.requests.storage`, và giá trị đó phải **bằng hoặc lớn hơn** dung lượng của
   volume nguồn. Bỏ hẳn trường đó cũng không được, vì nó bắt buộc.
2. **Storage class khác thì được, namespace khác thì không.** Bài cho phép volume đích dùng
   cùng storage class với nguồn hoặc một class khác, thậm chí bỏ `storageClassName` để lấy
   class mặc định. Nhưng bạn **chỉ nhân bản được một PVC khi nó nằm trong cùng namespace với
   PVC đích**. Cùng lúc đó, hai volume còn phải **cùng `volumeMode`** — xin volume ở chế độ
   block thì nguồn cũng phải ở chế độ block.
3. **Không.** Bài nói rõ PVC mới tạo là một **đối tượng độc lập**: nguồn không liên kết theo
   bất kỳ cách nào với bản nhân bản, và nguồn có thể được sửa đổi hoặc xóa mà **không ảnh hưởng
   đến bản nhân bản**. Clone có thể được dùng, nhân bản tiếp, chụp snapshot hoặc xóa riêng.
4. Bốn điều kiện nằm ngay ở mục *Giới thiệu*: lưu trữ phải đi qua **CSI driver**; phải là **bộ
   cấp phát động** (nhân bản không dùng được với PV cấp phát tĩnh); **CSI driver cụ thể đó phải
   đã triển khai chức năng nhân bản volume** — bài nói rõ driver "có thể đã hoặc chưa"; và
   nguồn phải **đang bound, không đang được sử dụng**. Không đủ thì ghi vào
   [sổ nợ lab](labs/README.md#5-sổ-nợ-lab) — nhân bản volume nằm chung một dòng nợ với snapshot
   và được trả ở Lab 6b nếu điều kiện cho phép.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

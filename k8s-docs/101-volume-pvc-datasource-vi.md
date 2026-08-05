# Nhân bản CSI Volume (CSI Volume Cloning)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/volume-pvc-datasource/>

Tài liệu này mô tả khái niệm nhân bản (clone) các CSI Volume có sẵn trong Kubernetes.
Bạn nên làm quen trước với [Volume](https://kubernetes.io/docs/concepts/storage/volumes).

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

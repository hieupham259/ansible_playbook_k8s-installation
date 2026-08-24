# Scale một StatefulSet (Scale a StatefulSet)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/scale-stateful-set/

Tác vụ này hướng dẫn cách scale một StatefulSet. Scale một StatefulSet nghĩa là
tăng hoặc giảm số lượng replicas.

## Trước khi bạn bắt đầu (Before you begin)

- StatefulSet chỉ khả dụng từ Kubernetes phiên bản 1.5 trở lên.
  Để kiểm tra phiên bản Kubernetes của bạn, hãy chạy `kubectl version`.

- Không phải ứng dụng stateful nào cũng scale tốt. Nếu bạn không chắc có nên
  scale StatefulSet của mình hay không, hãy xem [khái niệm StatefulSet](65-statefulset-vi.md)
  hoặc [hướng dẫn StatefulSet](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/) để biết thêm thông tin.

- Bạn chỉ nên thực hiện scale khi bạn tin chắc rằng cluster ứng dụng stateful
  của mình hoàn toàn khỏe mạnh.

## Scale các StatefulSet (Scaling StatefulSets)

### Dùng kubectl để scale StatefulSet (Use kubectl to scale StatefulSets)

Trước tiên, tìm StatefulSet mà bạn muốn scale.

```shell
kubectl get statefulsets <stateful-set-name>
```

Thay đổi số replicas của StatefulSet:

```shell
kubectl scale statefulsets <stateful-set-name> --replicas=<new-replicas>
```

### Thực hiện cập nhật tại chỗ trên StatefulSet của bạn (Make in-place updates on your StatefulSets)

Ngoài ra, bạn có thể thực hiện
[cập nhật tại chỗ (in-place updates)](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#in-place-updates-of-resources)
trên các StatefulSet của mình.

Nếu StatefulSet của bạn ban đầu được tạo bằng `kubectl apply`,
hãy cập nhật `.spec.replicas` trong manifest của StatefulSet, sau đó chạy `kubectl apply`:

```shell
kubectl apply -f <stateful-set-file-updated>
```

Nếu không, hãy chỉnh sửa trường đó bằng `kubectl edit`:

```shell
kubectl edit statefulsets <stateful-set-name>
```

Hoặc dùng `kubectl patch`:

```shell
kubectl patch statefulsets <stateful-set-name> -p '{"spec":{"replicas":<new-replicas>}}'
```

## Xử lý sự cố (Troubleshooting)

### Scale down không hoạt động đúng (Scaling down does not work right)

Bạn không thể scale down một StatefulSet khi bất kỳ Pod stateful nào mà nó quản lý
đang không khỏe mạnh (unhealthy). Việc scale down chỉ diễn ra sau khi các Pod stateful
đó trở về trạng thái running và ready.

Nếu spec.replicas > 1, Kubernetes không thể xác định nguyên nhân khiến một Pod không
khỏe mạnh. Đó có thể là kết quả của một lỗi vĩnh viễn (permanent fault) hoặc một lỗi
tạm thời (transient fault). Lỗi tạm thời có thể do một lần khởi động lại cần thiết
trong quá trình nâng cấp hoặc bảo trì.

Nếu Pod không khỏe mạnh do lỗi vĩnh viễn, việc scale mà không sửa lỗi đó
có thể dẫn tới trạng thái mà số thành viên của StatefulSet
tụt xuống dưới một số lượng replicas tối thiểu cần thiết để hoạt động
đúng. Điều này có thể khiến StatefulSet của bạn trở nên không khả dụng.

Nếu Pod không khỏe mạnh do lỗi tạm thời và Pod có thể khả dụng trở lại,
lỗi tạm thời đó có thể gây trở ngại cho thao tác scale up hoặc scale down của bạn. Một số
cơ sở dữ liệu phân tán gặp vấn đề khi các node tham gia và rời đi cùng lúc. Trong những
trường hợp này, tốt hơn là suy xét các thao tác scale ở cấp độ ứng dụng, và
chỉ thực hiện scale khi bạn chắc chắn rằng cluster ứng dụng stateful của mình
hoàn toàn khỏe mạnh.

## Tiếp theo (What's next)

- Tìm hiểu thêm về [xóa một StatefulSet](340-delete-stateful-set-vi.md).

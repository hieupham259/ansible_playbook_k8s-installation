# Xóa một StatefulSet (Delete a StatefulSet)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/delete-stateful-set/

Trang này hướng dẫn bạn cách xóa một StatefulSet.

## Trước khi bạn bắt đầu (Before you begin)

- Trang này giả định rằng bạn có một ứng dụng đang chạy trên cluster, được biểu diễn bởi một StatefulSet.

## Xóa một StatefulSet (Deleting a StatefulSet)

Bạn có thể xóa một StatefulSet theo cùng cách bạn xóa các resource khác trong Kubernetes: dùng lệnh `kubectl delete`, và chỉ định StatefulSet theo file hoặc theo tên.

```shell
kubectl delete -f <file.yaml>
```

```shell
kubectl delete statefulsets <statefulset-name>
```

Bạn có thể cần xóa riêng headless service liên quan sau khi bản thân StatefulSet đã bị xóa.

```shell
kubectl delete service <service-name>
```

Khi xóa một StatefulSet thông qua `kubectl`, StatefulSet được scale xuống 0. Tất cả các Pod thuộc workload này cũng bị xóa. Nếu bạn chỉ muốn xóa StatefulSet mà không xóa các Pod, hãy dùng `--cascade=orphan`. Ví dụ:

```shell
kubectl delete -f <file.yaml> --cascade=orphan
```

Bằng cách truyền `--cascade=orphan` vào `kubectl delete`, các Pod do StatefulSet quản lý được giữ lại ngay cả sau khi bản thân đối tượng StatefulSet đã bị xóa. Nếu các Pod có label `app.kubernetes.io/name=MyApp`, sau đó bạn có thể xóa chúng như sau:

```shell
kubectl delete pods -l app.kubernetes.io/name=MyApp
```

### Persistent Volume (Persistent Volumes)

Việc xóa các Pod trong một StatefulSet sẽ không xóa các volume liên quan. Điều này nhằm đảm bảo rằng bạn có cơ hội sao chép dữ liệu ra khỏi volume trước khi xóa nó. Việc xóa PVC sau khi các Pod đã kết thúc (terminate) có thể kích hoạt việc xóa các Persistent Volume phía sau, tùy thuộc vào storage class và reclaim policy. Bạn không bao giờ nên giả định rằng mình vẫn có khả năng truy cập một volume sau khi claim đã bị xóa.

> **Ghi chú:**
> Hãy thận trọng khi xóa một PVC, vì việc này có thể dẫn tới mất dữ liệu.

### Xóa hoàn toàn một StatefulSet (Complete deletion of a StatefulSet)

Để xóa mọi thứ trong một StatefulSet, bao gồm cả các Pod liên quan, bạn có thể chạy một chuỗi lệnh tương tự như sau:

```shell
grace=$(kubectl get pods <stateful-set-pod> --template '{{.spec.terminationGracePeriodSeconds}}')
kubectl delete statefulset -l app.kubernetes.io/name=MyApp
sleep $grace
kubectl delete pvc -l app.kubernetes.io/name=MyApp

```

Trong ví dụ trên, các Pod có label `app.kubernetes.io/name=MyApp`; hãy thay bằng label của riêng bạn cho phù hợp.

### Buộc xóa các Pod của StatefulSet (Force deletion of StatefulSet pods)

Nếu bạn thấy một số Pod trong StatefulSet của mình bị kẹt ở trạng thái 'Terminating' hoặc 'Unknown' trong một khoảng thời gian dài, bạn có thể cần can thiệp thủ công để buộc xóa (force delete) các Pod khỏi apiserver. Đây là một thao tác tiềm ẩn nguy hiểm. Tham khảo [Buộc xóa các Pod của StatefulSet (Force Delete StatefulSet Pods)](https://kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/) để biết chi tiết.

## Tiếp theo (What's next)

Tìm hiểu thêm về [buộc xóa các Pod của StatefulSet](https://kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/).

# Giới hạn mức tiêu thụ lưu trữ (Limit Storage Consumption)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/limit-storage-consumption/>

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

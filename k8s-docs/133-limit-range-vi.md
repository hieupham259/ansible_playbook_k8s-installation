# Khoảng giới hạn tài nguyên (Limit Ranges)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/policy/limit-range/>

Theo mặc định, các container chạy với [tài nguyên tính toán](./110-manage-resources-containers-vi.md)
không bị giới hạn trên một cluster Kubernetes.
Bằng cách sử dụng [hạn ngạch tài nguyên (resource quota)](./134-resource-quotas-vi.md) của Kubernetes,
quản trị viên (còn gọi là _người vận hành cluster_ — cluster operator) có thể hạn chế mức tiêu thụ và việc tạo mới
tài nguyên của cluster (chẳng hạn thời gian CPU, bộ nhớ và lưu trữ bền vững — persistent storage) trong một
namespace được chỉ định.
Bên trong một namespace, một Pod có thể tiêu thụ lượng CPU và bộ nhớ tối đa mà các ResourceQuota áp dụng cho namespace đó cho phép.
Với vai trò người vận hành cluster, hoặc quản trị viên cấp namespace, bạn có thể cũng muốn đảm bảo
rằng một đối tượng đơn lẻ không thể chiếm dụng toàn bộ tài nguyên khả dụng trong một namespace.

LimitRange là một chính sách nhằm ràng buộc việc cấp phát tài nguyên (limit và request) mà bạn có thể chỉ định
cho từng loại đối tượng áp dụng được (chẳng hạn Pod hoặc PersistentVolumeClaim) trong một namespace.

Một _LimitRange_ cung cấp các ràng buộc có thể:

- Ép buộc mức sử dụng tài nguyên tính toán tối thiểu và tối đa cho mỗi Pod hoặc Container trong một namespace.
- Ép buộc mức yêu cầu lưu trữ (storage request) tối thiểu và tối đa cho mỗi
  PersistentVolumeClaim trong một namespace.
- Ép buộc tỷ lệ giữa request và limit cho một loại tài nguyên trong một namespace.
- Đặt request/limit mặc định cho tài nguyên tính toán trong một namespace và tự động
  chèn chúng vào các Container lúc chạy (runtime).

Kubernetes ràng buộc việc cấp phát tài nguyên cho các Pod trong một namespace cụ thể
mỗi khi có ít nhất một đối tượng LimitRange trong namespace đó.

Tên của một đối tượng LimitRange phải là một
[tên miền con DNS hợp lệ](./17-names-vi.md#dns-subdomain-names).

## Ràng buộc đối với limit và request tài nguyên (Constraints on resource limits and requests)

- Quản trị viên tạo một LimitRange trong một namespace.
- Người dùng tạo (hoặc cố gắng tạo) các đối tượng trong namespace đó, chẳng hạn Pod hoặc
  PersistentVolumeClaim.
- Trước hết, admission controller LimitRange áp dụng các giá trị request và limit mặc định
  cho tất cả các Pod (và các container của chúng) chưa đặt yêu cầu tài nguyên tính toán.
- Sau đó, LimitRange theo dõi mức sử dụng để đảm bảo nó không vượt quá mức tối thiểu,
  tối đa và tỷ lệ tài nguyên được định nghĩa trong bất kỳ LimitRange nào có mặt trong namespace.
- Nếu bạn cố gắng tạo hoặc cập nhật một đối tượng (Pod hoặc PersistentVolumeClaim) vi phạm
  một ràng buộc của LimitRange, yêu cầu của bạn gửi tới API server sẽ thất bại với mã trạng thái HTTP
  `403 Forbidden` kèm theo một thông báo giải thích ràng buộc đã bị vi phạm.
- Nếu bạn thêm một LimitRange trong một namespace áp dụng cho các tài nguyên liên quan đến tính toán
  như `cpu` và `memory`, bạn phải chỉ định request hoặc limit cho các giá trị đó.
  Nếu không, hệ thống có thể từ chối việc tạo Pod.
- Việc kiểm tra LimitRange chỉ diễn ra ở giai đoạn admission của Pod, không áp dụng trên các Pod đang chạy.
  Nếu bạn thêm hoặc sửa đổi một LimitRange, các Pod đã tồn tại từ trước trong namespace đó
  vẫn tiếp tục chạy không thay đổi.
- Nếu có hai hoặc nhiều đối tượng LimitRange tồn tại trong namespace, việc giá trị mặc định nào
  sẽ được áp dụng là không xác định (not deterministic).

## LimitRange và kiểm tra admission cho Pod (LimitRange and admission checks for Pods)

Một LimitRange **không** kiểm tra tính nhất quán của các giá trị mặc định mà nó áp dụng.
Điều này có nghĩa là giá trị mặc định cho _limit_ do LimitRange đặt có thể
nhỏ hơn giá trị _request_ được chỉ định cho container trong spec mà client
gửi tới API server. Nếu điều đó xảy ra, Pod cuối cùng sẽ không thể được xếp lịch (schedule).

Ví dụ, bạn định nghĩa một LimitRange với manifest dưới đây:

> **Ghi chú:** Các ví dụ sau đây hoạt động trong namespace default của cluster của bạn, vì tham số
> namespace không được định nghĩa và phạm vi của LimitRange giới hạn ở cấp namespace.
> Điều này ngụ ý rằng mọi tham chiếu hoặc thao tác trong các ví dụ này sẽ tương tác
> với các thành phần trong namespace default của cluster của bạn. Bạn có thể ghi đè
> namespace hoạt động bằng cách cấu hình namespace trong trường `metadata.namespace`.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
spec:
  limits:
  - default: # phần này định nghĩa các limit mặc định
      cpu: 500m
    defaultRequest: # phần này định nghĩa các request mặc định
      cpu: 500m
    max: # max và min định nghĩa khoảng giới hạn
      cpu: "1"
    min:
      cpu: 100m
    type: Container
```

cùng với một Pod khai báo CPU resource request là `700m`, nhưng không khai báo limit:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-conflict-with-limitrange-cpu
spec:
  containers:
  - name: demo
    image: registry.k8s.io/pause:3.8
    resources:
      requests:
        cpu: 700m
```

khi đó Pod này sẽ không được xếp lịch, thất bại với lỗi tương tự như:

```
Pod "example-conflict-with-limitrange-cpu" is invalid: spec.containers[0].resources.requests: Invalid value: "700m": must be less than or equal to cpu limit
```

Nếu bạn đặt cả `request` lẫn `limit`, thì Pod mới đó sẽ được xếp lịch thành công
ngay cả khi vẫn có cùng LimitRange đó:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-no-conflict-with-limitrange-cpu
spec:
  containers:
  - name: demo
    image: registry.k8s.io/pause:3.8
    resources:
      requests:
        cpu: 700m
      limits:
        cpu: 700m
```

## Ví dụ về ràng buộc tài nguyên (Example resource constraints)

Một số ví dụ về chính sách có thể được tạo bằng LimitRange là:

- Trong một cluster 2 node với dung lượng 8 GiB RAM và 16 core, ràng buộc các Pod trong một
  namespace request 100m CPU với limit tối đa 500m cho CPU, và request 200Mi
  cho bộ nhớ với limit tối đa 600Mi cho bộ nhớ.
- Định nghĩa limit và request CPU mặc định là 150m và request bộ nhớ mặc định là 300Mi cho
  các Container được khởi chạy mà không có request cpu và memory trong spec của chúng.

Trong trường hợp tổng limit của namespace nhỏ hơn tổng các limit
của các Pod/Container, có thể xảy ra tranh chấp (contention) tài nguyên. Trong trường hợp này,
các Container hoặc Pod sẽ không được tạo.

Cả tranh chấp lẫn các thay đổi đối với một LimitRange đều không ảnh hưởng đến các tài nguyên đã được tạo trước đó.

## Tiếp theo (What's next)

Để xem các ví dụ về việc sử dụng limit, hãy xem:

- [cách cấu hình ràng buộc CPU tối thiểu và tối đa cho mỗi namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/).
- [cách cấu hình ràng buộc bộ nhớ tối thiểu và tối đa cho mỗi namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/).
- [cách cấu hình CPU Request và Limit mặc định cho mỗi namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/).
- [cách cấu hình Request và Limit bộ nhớ mặc định cho mỗi namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/).
- [cách cấu hình mức tiêu thụ lưu trữ tối thiểu và tối đa cho mỗi namespace](https://kubernetes.io/docs/tasks/administer-cluster/limit-storage-consumption/#limitrange-to-limit-requests-for-storage).
- một [ví dụ chi tiết về cấu hình hạn ngạch cho mỗi namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/).

Tham khảo [tài liệu thiết kế LimitRanger](https://git.k8s.io/design-proposals-archive/resource-management/admission_control_limit_range.md)
để biết bối cảnh và thông tin lịch sử.

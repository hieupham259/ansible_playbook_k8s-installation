# Cơ chế admission bảo mật Pod (Pod Security Admission)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/pod-security-admission/>
>
> Tổng quan về Pod Security Admission Controller — admission controller có thể thực thi các Tiêu chuẩn bảo mật Pod (Pod Security Standards).

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

[Tiêu chuẩn bảo mật Pod (Pod Security Standards)](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
của Kubernetes định nghĩa các mức độ cô lập (isolation) khác nhau cho Pod. Các tiêu chuẩn này
cho phép bạn xác định cách bạn muốn hạn chế hành vi của các pod một cách rõ ràng và nhất quán.

Kubernetes cung cấp sẵn một admission controller tích hợp tên là _Pod Security_ để thực thi
các Tiêu chuẩn bảo mật Pod. Các hạn chế bảo mật pod được áp dụng ở cấp namespace
tại thời điểm pod được tạo.

### Thực thi Pod Security admission tích hợp sẵn (Built-in Pod Security admission enforcement)

Trang này là một phần của bộ tài liệu dành cho Kubernetes v1.36.
Nếu bạn đang chạy một phiên bản Kubernetes khác, hãy tham khảo tài liệu của phiên bản đó.

## Các mức bảo mật Pod (Pod Security levels)

Pod Security admission đặt ra các yêu cầu đối với [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
của Pod và các trường liên quan khác, theo ba mức được định nghĩa bởi
[Tiêu chuẩn bảo mật Pod](https://kubernetes.io/docs/concepts/security/pod-security-standards):
`privileged`, `baseline` và `restricted`. Hãy xem trang
[Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards)
để tìm hiểu chuyên sâu về các yêu cầu đó.

## Các label Pod Security Admission cho namespace (Pod Security Admission labels for namespaces)

Sau khi tính năng được bật hoặc webhook được cài đặt, bạn có thể cấu hình các namespace để
xác định chế độ kiểm soát admission mà bạn muốn dùng cho bảo mật pod trong từng namespace.
Kubernetes định nghĩa một tập các label mà bạn có thể đặt để chỉ định
mức Pod Security Standard định sẵn nào sẽ được dùng cho một namespace. Label mà bạn chọn
xác định hành động control plane sẽ thực hiện nếu phát hiện một vi phạm tiềm ẩn:

*Các chế độ của Pod Security Admission (Pod Security Admission modes)*

Chế độ (Mode) | Mô tả
:---------|:------------
**enforce** | Vi phạm chính sách sẽ khiến pod bị từ chối.
**audit** | Vi phạm chính sách sẽ kích hoạt việc thêm một audit annotation vào sự kiện được ghi trong [audit log](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/), nhưng pod vẫn được cho phép.
**warn** | Vi phạm chính sách sẽ kích hoạt một cảnh báo hiển thị cho người dùng, nhưng pod vẫn được cho phép.

Một namespace có thể cấu hình một, một vài hoặc tất cả các chế độ, thậm chí đặt
mức khác nhau cho từng chế độ khác nhau.

Với mỗi chế độ, có hai label xác định chính sách được sử dụng:

```yaml
# Label mức theo từng chế độ, cho biết mức chính sách nào được áp dụng cho chế độ đó.
#
# MODE phải là một trong `enforce`, `audit` hoặc `warn`.
# LEVEL phải là một trong `privileged`, `baseline` hoặc `restricted`.
pod-security.kubernetes.io/<MODE>: <LEVEL>

# Tùy chọn: label phiên bản theo từng chế độ, có thể dùng để ghim chính sách vào
# phiên bản đi kèm một phiên bản minor nhất định của Kubernetes (ví dụ v1.36).
#
# MODE phải là một trong `enforce`, `audit` hoặc `warn`.
# VERSION phải là một phiên bản minor hợp lệ của Kubernetes, hoặc `latest`.
pod-security.kubernetes.io/<MODE>-version: <VERSION>
```

Xem [Thực thi Pod Security Standards bằng label của namespace](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels)
để biết ví dụ sử dụng.

## Tài nguyên workload và Pod template (Workload resources and Pod templates) {#workload-resources-and-pod-templates}

Pod thường được tạo một cách gián tiếp, thông qua việc tạo một
[workload object](https://kubernetes.io/docs/concepts/workloads/controllers/) như Deployment
hoặc Job. Workload object định nghĩa một _Pod template_ và một controller
của workload resource sẽ tạo các Pod dựa trên template đó. Để giúp phát hiện vi phạm sớm,
cả chế độ audit và warn đều được áp dụng cho các workload resource. Tuy nhiên, chế độ enforce
**không** được áp dụng cho workload resource, mà chỉ áp dụng cho các pod object được tạo ra.

## Miễn trừ (Exemptions)

Bạn có thể định nghĩa các _miễn trừ_ (exemption) khỏi việc thực thi bảo mật pod để cho phép
tạo những pod lẽ ra đã bị cấm bởi chính sách gắn với một namespace nhất định.
Miễn trừ có thể được cấu hình tĩnh trong
[cấu hình Admission Controller](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/#configure-the-admission-controller).

Miễn trừ phải được liệt kê một cách tường minh. Các request thỏa mãn tiêu chí miễn trừ sẽ bị
Admission Controller _bỏ qua_ (mọi hành vi `enforce`, `audit` và `warn` đều được bỏ qua).
Các chiều miễn trừ bao gồm:

- **Usernames:** request từ người dùng có username đã xác thực (hoặc được mạo danh — impersonated)
  thuộc diện miễn trừ sẽ bị bỏ qua.
- **RuntimeClassNames:** pod và [workload resource](#workload-resources-and-pod-templates) chỉ định
  một runtime class name thuộc diện miễn trừ sẽ bị bỏ qua.
- **Namespaces:** pod và [workload resource](#workload-resources-and-pod-templates) nằm trong một namespace thuộc diện miễn trừ sẽ bị bỏ qua.

> **Thận trọng:**
>
> Hầu hết pod được tạo bởi một controller để đáp ứng một
> [workload resource](#workload-resources-and-pod-templates), nghĩa là việc miễn trừ một người dùng cuối
> chỉ miễn trừ họ khỏi việc thực thi khi tạo pod trực tiếp, chứ không phải khi tạo một workload resource.
> Các service account của controller (chẳng hạn `system:serviceaccount:kube-system:replicaset-controller`)
> nói chung không nên được miễn trừ, vì làm vậy sẽ ngầm miễn trừ cho bất kỳ người dùng nào có thể tạo
> workload resource tương ứng.

Các cập nhật đối với những trường pod sau đây được miễn kiểm tra chính sách, nghĩa là nếu một
request cập nhật pod chỉ thay đổi các trường này, request đó sẽ không bị từ chối ngay cả khi pod
đang vi phạm mức chính sách hiện tại:

- Mọi cập nhật metadata **ngoại trừ** các thay đổi đối với annotation seccomp hoặc AppArmor:
  - `seccomp.security.alpha.kubernetes.io/pod` (đã lỗi thời — deprecated)
  - `container.seccomp.security.alpha.kubernetes.io/*` (đã lỗi thời)
  - `container.apparmor.security.beta.kubernetes.io/*` (đã lỗi thời)
- Các cập nhật hợp lệ đối với `.spec.activeDeadlineSeconds`
- Các cập nhật hợp lệ đối với `.spec.tolerations`

## Số liệu đo (Metrics)

Dưới đây là các metric Prometheus được kube-apiserver cung cấp:

- `pod_security_errors_total`: Metric này cho biết số lượng lỗi ngăn cản quá trình đánh giá bình thường.
  Các lỗi không nghiêm trọng (non-fatal) có thể dẫn đến việc profile restricted mới nhất được dùng để thực thi.
- `pod_security_evaluations_total`: Metric này cho biết số lần đánh giá chính sách đã diễn ra,
  không tính các request bị bỏ qua hoặc được miễn trừ trong quá trình xuất số liệu.
- `pod_security_exemptions_total`: Metric này cho biết số lượng request được miễn trừ, không tính
  các request bị bỏ qua hoặc nằm ngoài phạm vi.

## Tiếp theo (What's next)

- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards)
- [Thực thi Pod Security Standards](https://kubernetes.io/docs/setup/best-practices/enforcing-pod-security-standards)
- [Thực thi Pod Security Standards bằng cách cấu hình Admission Controller tích hợp sẵn](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller)
- [Thực thi Pod Security Standards bằng label của namespace](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels)

Nếu bạn đang chạy một phiên bản Kubernetes cũ hơn và muốn nâng cấp
lên một phiên bản Kubernetes không còn PodSecurityPolicy,
hãy đọc [Di trú từ PodSecurityPolicy sang PodSecurity Admission Controller tích hợp sẵn](https://kubernetes.io/docs/tasks/configure-pod-container/migrate-from-psp).

# Chính sách (Policies)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/policy/>
>
> Quản lý bảo mật và các thực hành tốt nhất (best practices) bằng chính sách.

Chính sách (policy) trong Kubernetes là các cấu hình dùng để quản lý các cấu hình khác
hoặc các hành vi lúc chạy (runtime). Kubernetes cung cấp nhiều hình thức chính sách khác nhau,
được mô tả dưới đây:

## Áp dụng chính sách bằng các đối tượng API (Apply policies using API objects)

Một số đối tượng API đóng vai trò như chính sách. Dưới đây là một vài ví dụ:

* [NetworkPolicy](./84-network-policies-vi.md) có thể được dùng để hạn chế
  lưu lượng vào (ingress) và ra (egress) của một workload.
* [LimitRange](./133-limit-range-vi.md) quản lý các ràng buộc cấp phát tài nguyên
  trên nhiều loại đối tượng khác nhau.
* [ResourceQuota](./134-resource-quotas-vi.md) giới hạn mức tiêu thụ tài nguyên
  của một namespace.

## Áp dụng chính sách bằng admission controller (Apply policies using admission controllers)

Một admission controller chạy bên trong API server
và có thể xác thực (validate) hoặc biến đổi (mutate) các yêu cầu API. Một số admission controller
đóng vai trò áp dụng chính sách.
Ví dụ, admission controller [AlwaysPullImages](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages)
sửa đổi một Pod mới để đặt chính sách kéo image thành `Always`.

Kubernetes có sẵn một số admission controller tích hợp, có thể cấu hình được thông qua
cờ `--enable-admission-plugins` của API server.

Chi tiết về admission controller, cùng với danh sách đầy đủ các admission controller hiện có,
được ghi lại trong một mục riêng:

* [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)

## Áp dụng chính sách bằng ValidatingAdmissionPolicy (Apply policies using ValidatingAdmissionPolicy)

Validating admission policy cho phép thực thi các kiểm tra xác thực có thể cấu hình được
ngay trong API server bằng ngôn ngữ Common Expression Language (CEL). Ví dụ, một `ValidatingAdmissionPolicy`
có thể được dùng để cấm sử dụng tag image `latest`.

Một `ValidatingAdmissionPolicy` hoạt động trên một yêu cầu API và có thể được dùng để chặn (block),
ghi kiểm toán (audit) và cảnh báo (warn) người dùng về các cấu hình không tuân thủ.

Chi tiết về API `ValidatingAdmissionPolicy`, kèm theo ví dụ, được ghi lại trong một mục riêng:

* [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)

## Áp dụng chính sách bằng dynamic admission control (Apply policies using dynamic admission control)

Dynamic admission controller (hay admission webhook) chạy bên ngoài API server
dưới dạng các ứng dụng riêng biệt, đăng ký nhận các yêu cầu webhook để thực hiện
xác thực hoặc biến đổi các yêu cầu API.

Dynamic admission controller có thể được dùng để áp dụng chính sách lên các yêu cầu API
và kích hoạt các luồng công việc (workflow) khác dựa trên chính sách. Một dynamic admission controller
có thể thực hiện các kiểm tra phức tạp, bao gồm cả những kiểm tra cần truy xuất
tài nguyên khác của cluster và dữ liệu bên ngoài. Ví dụ, một bước kiểm tra xác minh image
có thể tra cứu dữ liệu từ các OCI registry để xác thực chữ ký (signature)
và chứng thực (attestation) của container image.

Chi tiết về dynamic admission control được ghi lại trong một mục riêng:

* [Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)

### Các triển khai (Implementations) {#implementations-admission-control}

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần.
> Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án bên thứ ba này.

Trong hệ sinh thái Kubernetes, các dynamic admission controller đóng vai trò như những
bộ máy chính sách (policy engine) linh hoạt đang được phát triển, chẳng hạn:

- [Kubewarden](https://github.com/kubewarden)
- [Kyverno](https://kyverno.io)
- [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper)
- [Polaris](https://polaris.docs.fairwinds.com/admission-controller/)

## Áp dụng chính sách bằng cấu hình Kubelet (Apply policies using Kubelet configurations)

Kubernetes cho phép cấu hình Kubelet trên mỗi worker node. Một số cấu hình Kubelet đóng vai trò như chính sách:

* [Giới hạn và dự trữ Process ID](https://kubernetes.io/docs/concepts/policy/pid-limiting/)
  được dùng để giới hạn và dự trữ số PID có thể cấp phát.
* [Node Resource Managers](https://kubernetes.io/docs/concepts/policy/node-resource-managers/)
  có thể quản lý tài nguyên tính toán, bộ nhớ và thiết bị cho các workload
  nhạy cảm với độ trễ (latency-critical) và có thông lượng cao (high-throughput).

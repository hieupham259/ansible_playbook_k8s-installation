# Thực thi Pod Security Standards bằng cách cấu hình Admission Controller tích hợp sẵn (Enforce Pod Security Standards by Configuring the Built-in Admission Controller)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/>

Kubernetes cung cấp một
[admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podsecurity)
tích hợp sẵn để thực thi các [Pod Security Standard](./115-pod-security-standards-vi.md).
Bạn có thể cấu hình admission controller này để đặt các giá trị mặc định trên toàn cluster và
các [miễn trừ (exemptions)](./116-pod-security-admission-vi.md#miễn-trừ-exemptions).

## Trước khi bạn bắt đầu (Before you begin)

Sau bản phát hành alpha trong Kubernetes v1.22, Pod Security Admission trở nên khả dụng theo
mặc định trong Kubernetes v1.23, ở trạng thái beta. Từ phiên bản 1.25 trở đi, Pod Security
Admission đạt mức phổ biến rộng rãi (generally available).

Để kiểm tra phiên bản, hãy nhập `kubectl version`.

Nếu bạn không chạy Kubernetes 1.36, bạn có thể chuyển sang xem trang này trong tài liệu của
phiên bản Kubernetes mà bạn đang chạy.

## Cấu hình Admission Controller (Configure the Admission Controller)

> **Ghi chú:** Cấu hình `pod-security.admission.config.k8s.io/v1` yêu cầu v1.25+.
> Với v1.23 và v1.24, dùng
> [v1beta1](https://v1-24.docs.kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/).
> Với v1.22, dùng
> [v1alpha1](https://v1-22.docs.kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/).

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1 # xem ghi chú về tương thích
    kind: PodSecurityConfiguration
    # Giá trị mặc định được áp dụng khi một nhãn (label) mode không được đặt.
    #
    # Giá trị nhãn level phải là một trong:
    # - "privileged" (mặc định)
    # - "baseline"
    # - "restricted"
    #
    # Giá trị nhãn version phải là một trong:
    # - "latest" (mặc định)
    # - phiên bản cụ thể, ví dụ "v1.36"
    defaults:
      enforce: "privileged"
      enforce-version: "latest"
      audit: "privileged"
      audit-version: "latest"
      warn: "privileged"
      warn-version: "latest"
    exemptions:
      # Mảng các username đã xác thực được miễn trừ.
      usernames: []
      # Mảng các tên runtime class được miễn trừ.
      runtimeClasses: []
      # Mảng các namespace được miễn trừ.
      namespaces: []
```

> **Ghi chú:** Manifest ở trên cần được chỉ định cho kube-apiserver thông qua flag
> `--admission-control-config-file`.

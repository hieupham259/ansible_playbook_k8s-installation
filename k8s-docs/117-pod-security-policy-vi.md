# Chính sách bảo mật Pod (Pod Security Policies)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/pod-security-policy/>

> **Cảnh báo: Tính năng đã bị gỡ bỏ (Removed feature)**
>
> PodSecurityPolicy đã [bị đánh dấu lỗi thời (deprecated)](https://kubernetes.io/blog/2021/04/08/kubernetes-1-21-release-announcement/#podsecuritypolicy-deprecation)
> trong Kubernetes v1.21, và đã bị gỡ bỏ khỏi Kubernetes ở v1.25.

Thay vì sử dụng PodSecurityPolicy, bạn có thể thực thi các hạn chế tương tự trên Pod
bằng một trong hai cách sau, hoặc cả hai:

- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- một admission plugin của bên thứ ba, do bạn tự triển khai và cấu hình

Để xem hướng dẫn di trú (migration), hãy đọc [Di trú từ PodSecurityPolicy sang PodSecurity Admission Controller tích hợp sẵn](https://kubernetes.io/docs/tasks/configure-pod-container/migrate-from-psp/).
Để biết thêm thông tin về việc gỡ bỏ API này,
xem [PodSecurityPolicy Deprecation: Past, Present, and Future](https://kubernetes.io/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/).

Nếu bạn không chạy Kubernetes v1.36, hãy kiểm tra tài liệu tương ứng với
phiên bản Kubernetes của bạn.

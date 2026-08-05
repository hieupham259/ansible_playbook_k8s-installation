# Danh sách kiểm tra bảo mật ứng dụng (Application Security Checklist)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/application-security-checklist/>
>
> Các hướng dẫn cơ bản (baseline) về việc đảm bảo bảo mật ứng dụng trên Kubernetes, dành cho các nhà phát triển ứng dụng.

Danh sách kiểm tra (checklist) này nhằm cung cấp các hướng dẫn cơ bản về việc bảo mật các ứng dụng
chạy trong Kubernetes từ góc nhìn của nhà phát triển (developer).
Danh sách này không nhằm mục đích đầy đủ và sẽ tiếp tục được phát triển theo thời gian.

Về cách đọc và sử dụng tài liệu này:

- Thứ tự các chủ đề không phản ánh thứ tự ưu tiên.
- Một số mục trong danh sách kiểm tra được giải thích chi tiết trong đoạn văn bên dưới danh sách của mỗi phần.
- Danh sách này giả định rằng `developer` là một người dùng cluster Kubernetes,
  người tương tác với các đối tượng thuộc phạm vi namespace.

> **Thận trọng:**
> Danh sách kiểm tra tự nó **không** đủ để đạt được một thế trận bảo mật (security posture) tốt.
> Một thế trận bảo mật tốt đòi hỏi sự chú ý và cải thiện liên tục, nhưng danh sách kiểm tra
> có thể là bước đầu tiên trên hành trình không có điểm dừng hướng tới sự sẵn sàng về bảo mật.
> Một số khuyến nghị trong danh sách này có thể quá chặt chẽ hoặc quá lỏng lẻo so với
> nhu cầu bảo mật cụ thể của bạn. Vì bảo mật Kubernetes không phải là "một khuôn mẫu chung cho tất cả" (one size fits all),
> mỗi nhóm mục trong danh sách kiểm tra nên được đánh giá dựa trên giá trị riêng của nó.

## Tăng cường bảo mật cơ bản (Base security hardening)

Danh sách kiểm tra sau đây cung cấp các khuyến nghị tăng cường bảo mật cơ bản
áp dụng cho hầu hết các ứng dụng triển khai lên Kubernetes.

### Thiết kế ứng dụng (Application design)

- [ ] Tuân theo các
  [nguyên tắc bảo mật](https://www.cncf.io/wp-content/uploads/2022/06/CNCF_cloud-native-security-whitepaper-May2022-v2.pdf)
  đúng đắn khi thiết kế ứng dụng.
- [ ] Ứng dụng được cấu hình với QoS class phù hợp
  thông qua resource request và limit.
  - [ ] Giới hạn (limit) bộ nhớ được đặt cho các workload với limit bằng hoặc lớn hơn request.
  - [ ] Giới hạn CPU có thể được đặt cho các workload nhạy cảm.

### Service account

- [ ] Tránh sử dụng ServiceAccount `default`. Thay vào đó, hãy tạo ServiceAccount cho
  từng workload hoặc microservice.
- [ ] `automountServiceAccountToken` nên được đặt là `false` trừ khi pod
  thực sự cần truy cập Kubernetes API để hoạt động.

### Khuyến nghị `securityContext` cấp Pod (Pod-level `securityContext` recommendations) {#security-context-pod}

- [ ] Đặt `runAsNonRoot: true`.
- [ ] Cấu hình container để thực thi với người dùng có ít đặc quyền hơn
  (ví dụ, dùng `runAsUser` và `runAsGroup`), và cấu hình quyền phù hợp
  trên các file hoặc thư mục bên trong container image.
- [ ] Tùy chọn, thêm một nhóm bổ sung với `fsGroup` để truy cập các persistent volume.
- [ ] Ứng dụng được triển khai vào một namespace có thực thi
  [Pod security standard](https://kubernetes.io/docs/concepts/security/pod-security-standards/) phù hợp.
  Nếu bạn không thể kiểm soát việc thực thi này cho (các) cluster nơi ứng dụng được
  triển khai, hãy tính đến điều đó thông qua tài liệu hướng dẫn hoặc các lớp phòng thủ theo chiều sâu (defense in depth) bổ sung.

### Khuyến nghị `securityContext` cấp container (Container-level `securityContext` recommendations) {#security-context-container}

- [ ] Vô hiệu hóa leo thang đặc quyền bằng `allowPrivilegeEscalation: false`.
- [ ] Cấu hình root filesystem ở chế độ chỉ đọc với `readOnlyRootFilesystem: true`.
- [ ] Tránh chạy container đặc quyền (đặt `privileged: false`).
- [ ] Loại bỏ tất cả capability khỏi container và chỉ thêm lại những capability cụ thể
  cần thiết cho hoạt động của container.

### Kiểm soát truy cập dựa trên vai trò (Role Based Access Control - RBAC) {#rbac}

- [ ] Các quyền như **create**, **patch**, **update** và **delete**
  chỉ nên được cấp khi cần thiết.
- [ ] Tránh tạo các quyền RBAC cho phép tạo hoặc cập nhật role, vì có thể dẫn đến
  [leo thang đặc quyền](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#privilege-escalation-prevention-and-bootstrapping).
- [ ] Rà soát các binding cho nhóm `system:unauthenticated` và gỡ bỏ chúng khi
  có thể, vì điều này trao quyền truy cập cho bất kỳ ai có thể kết nối tới API server ở tầng mạng.

Các động từ (verb) **create**, **update** và **delete** nên được cho phép một cách thận trọng.
Động từ **patch**, nếu được phép trên một Namespace, có thể
[cho phép người dùng cập nhật label trên namespace hoặc deployment](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#namespace-modification),
điều này có thể làm tăng bề mặt tấn công.

Với các workload nhạy cảm, hãy cân nhắc cung cấp một ValidatingAdmissionPolicy khuyến nghị
để hạn chế thêm các hành động ghi được phép.

### Bảo mật image (Image security)

- [ ] Sử dụng công cụ quét image để quét image trước khi triển khai container trong cluster Kubernetes.
- [ ] Sử dụng ký container (container signing) để xác minh chữ ký của container image trước khi triển khai vào cluster Kubernetes.

### Network policy (Network policies)

- [ ] Cấu hình [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
  để chỉ cho phép các luồng traffic ingress và egress như dự kiến từ các pod.

Hãy đảm bảo rằng cluster của bạn cung cấp và thực thi NetworkPolicy.
Nếu bạn viết một ứng dụng mà người dùng sẽ triển khai lên nhiều cluster khác nhau,
hãy cân nhắc liệu bạn có thể giả định rằng NetworkPolicy khả dụng và được thực thi hay không.

## Tăng cường bảo mật nâng cao (Advanced security hardening) {#advanced}

Phần này của tài liệu đề cập một số điểm tăng cường bảo mật nâng cao
có thể hữu ích tùy theo các cách thiết lập môi trường Kubernetes khác nhau.

### Bảo mật container Linux (Linux container security)

Cấu hình Security Context cho pod-container.

- [ ] [Đặt Seccomp Profile cho một container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-seccomp-profile-for-a-container).
- [ ] [Hạn chế quyền truy cập tài nguyên của container với AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/).
- [ ] [Gán nhãn SELinux cho một container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#assign-selinux-labels-to-a-container).

### Runtime class (Runtime classes)

- [ ] Cấu hình runtime class phù hợp cho các container.

> **Ghi chú:** Mục này đề cập đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần.
> Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án bên thứ ba đó.
> Đây là ghi chú miễn trừ trách nhiệm đối với nội dung của bên thứ ba.

Một số container có thể cần mức cô lập khác với mức được cung cấp bởi
runtime mặc định của cluster. `runtimeClassName` có thể được dùng trong podspec
để định nghĩa một runtime class khác.

Với các workload nhạy cảm, hãy cân nhắc dùng các công cụ mô phỏng kernel như
[gVisor](https://gvisor.dev/docs/), hoặc cô lập bằng ảo hóa với cơ chế
như [kata-containers](https://katacontainers.io/).

Trong các môi trường đòi hỏi độ tin cậy cao, hãy cân nhắc sử dụng
[máy ảo bảo mật (confidential virtual machine)](https://kubernetes.io/blog/2023/07/06/confidential-kubernetes/)
để nâng cao hơn nữa bảo mật của cluster.

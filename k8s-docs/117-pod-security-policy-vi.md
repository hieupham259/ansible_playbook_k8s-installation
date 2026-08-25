# Chính sách bảo mật Pod (Pod Security Policies)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/pod-security-policy/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 18/18 · Kiểm chứng ở [Lab 9b](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md).

Lộ trình đánh dấu bài này là **tài liệu lịch sử**, và xếp nó **sau Lab 9b** vì nó không có gì để
thực hành. PodSecurityPolicy **đã bị gỡ khỏi Kubernetes** — cluster lab chạy v1.35.6 nên hoàn
toàn không có API này. Chỉ đọc khi bạn **tiếp quản một cluster rất cũ** còn dùng PSP và cần biết
đường di trú. Cơ chế thay thế bạn đã học ở hai bài
[115](115-pod-security-standards-vi.md) và [116](116-pod-security-admission-vi.md). Trang chỉ
hơn 20 dòng, đọc trong một phút rồi khép giai đoạn 9.

**Phải hiểu ở lần đọc này:**

- PodSecurityPolicy **bị đánh dấu lỗi thời ở Kubernetes v1.21** và **bị gỡ bỏ hẳn ở v1.25** —
  đây là hai mốc khác nhau, và mốc thứ hai nghĩa là API không còn tồn tại.
- Hai cách thay thế mà bài nêu, dùng một trong hai hoặc cả hai:
  [Pod Security Admission](116-pod-security-admission-vi.md) tích hợp sẵn, hoặc **một admission
  plugin của bên thứ ba do bạn tự triển khai và cấu hình**.
- Khi gặp cluster cũ còn dùng PSP, việc cần làm là **đọc hướng dẫn di trú** sang PodSecurity
  Admission Controller, chứ không phải tìm cách dựng lại PodSecurityPolicy.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Link hướng dẫn *Di trú từ PodSecurityPolicy sang PodSecurity Admission Controller* | chỉ cần khi thật sự tiếp quản cluster cũ hơn v1.25 | không cần |
| Hai bài blog về quá trình loại bỏ PodSecurityPolicy | là bối cảnh lịch sử | không cần |

---

> **Cảnh báo: Tính năng đã bị gỡ bỏ (Removed feature)**
>
> PodSecurityPolicy đã [bị đánh dấu lỗi thời (deprecated)](https://kubernetes.io/blog/2021/04/08/kubernetes-1-21-release-announcement/#podsecuritypolicy-deprecation)
> trong Kubernetes v1.21, và đã bị gỡ bỏ khỏi Kubernetes ở v1.25.

Thay vì sử dụng PodSecurityPolicy, bạn có thể thực thi các hạn chế tương tự trên Pod
bằng một trong hai cách sau, hoặc cả hai:

- [Pod Security Admission](116-pod-security-admission-vi.md)
- một admission plugin của bên thứ ba, do bạn tự triển khai và cấu hình

Để xem hướng dẫn di trú (migration), hãy đọc [Di trú từ PodSecurityPolicy sang PodSecurity Admission Controller tích hợp sẵn](286-migrate-from-psp-vi.md).
Để biết thêm thông tin về việc gỡ bỏ API này,
xem [PodSecurityPolicy Deprecation: Past, Present, and Future](https://kubernetes.io/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/).

Nếu bạn không chạy Kubernetes v1.36, hãy kiểm tra tài liệu tương ứng với
phiên bản Kubernetes của bạn.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Bài này là tài liệu lịch sử, nên ba câu dưới đây chỉ kiểm tra bạn có đặt nó đúng chỗ hay không:

1. PodSecurityPolicy bị đánh dấu lỗi thời ở phiên bản nào, và bị **gỡ bỏ** ở phiên bản nào? Hai
   mốc đó khác nhau ra sao về hậu quả với người vận hành?
2. Cluster lab chạy Kubernetes v1.35.6. Bạn chạy một lệnh liệt kê PodSecurityPolicy trên
   `lab-k8s-master` thì nhận được gì, và vì sao?
3. Bạn tiếp quản một cluster cũ đang dùng PodSecurityPolicy. Theo bài, hai hướng thay thế là
   gì, và bạn nên đọc tài liệu nào trước khi nâng cấp?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Lỗi thời (deprecated) ở v1.21, bị gỡ bỏ khỏi Kubernetes ở v1.25.** Khác biệt về hậu quả:
   ở giai đoạn deprecated, API **vẫn còn và vẫn chạy**, bạn có thời gian chuyển đổi; sau khi bị
   gỡ bỏ thì **API không còn tồn tại**, manifest cũ không apply được nữa và mọi chính sách đang
   dựa vào nó **im lặng biến mất** khi cluster được nâng cấp qua mốc đó.
2. **Không có tài nguyên nào cả — server không nhận ra loại tài nguyên này.** Trực giác "vẫn còn
   nhưng bị cảnh báo deprecated" chỉ đúng với các bản từ v1.21 đến trước v1.25. Cluster lab chạy
   **v1.35.6**, tức là đã vượt mốc v1.25 rất xa, nên **PodSecurityPolicy không tồn tại** trong
   API của nó.
3. Hai hướng: **Pod Security Admission** tích hợp sẵn — chính là bài
   [116](116-pod-security-admission-vi.md) — hoặc **một admission plugin của bên thứ ba do bạn
   tự triển khai và cấu hình**; bài nói có thể dùng một trong hai hoặc cả hai. Trước khi nâng
   cấp, đọc **hướng dẫn di trú từ PodSecurityPolicy sang PodSecurity Admission Controller tích
   hợp sẵn** mà bài liên kết.

</details>

Đây là bài cuối của **giai đoạn 9**. Câu nào chưa trả lời được thì quay lại đúng mục tương ứng
trước khi đọc bài sau.

# TLS

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/tls/>
>
> Tìm hiểu cách bảo vệ lưu lượng (traffic) bên trong cluster của bạn bằng Transport Layer
> Security (TLS).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 18 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ), bài 1/7 · Không kiểm
chứng riêng: đây là trang mục, việc thực hành nằm ở bốn bài con.

**Đây là trang mục, không phải bài học.** Toàn bộ nội dung là danh sách bốn trang con. Đọc để
biết giai đoạn 18 gồm những gì rồi đi tiếp; đừng tick xong giai đoạn 18 chỉ vì đã đọc trang này.

**Phải hiểu ở lần đọc này:**

- Nhóm `tls/` có **bốn** việc khác nhau, đừng lẫn: cấp certificate cho **client API**
  ([397](397-certificate-issue-client-csr-vi.md)), **kubelet tự xoay** certificate của nó
  ([398](398-certificate-rotation-vi.md)), dùng **CSR API của cluster** để cấp certificate
  ([399](399-managing-tls-in-a-cluster-vi.md)), và **xoay chính CA**
  ([400](400-manual-rotation-of-ca-certificates-vi.md)).
- Ranh giới với bài [219](219-kubeadm-certs-vi.md): 219 nói về certificate **do kubeadm quản
  lý**; nhóm này nói về cơ chế certificate của **bản thân Kubernetes**, dùng được cả khi không
  có kubeadm.
- Mức nguy hiểm tăng dần theo thứ tự giai đoạn 18 đã sắp: đọc/kiểm tra → cấp mới → xoay CA. Bài
  [400](400-manual-rotation-of-ca-certificates-vi.md) đụng vào gốc tin cậy của cả cluster.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nội dung cụ thể của từng trang con | trang mục chỉ liệt kê, không giải thích | bốn bài con ngay sau, trong [Giai đoạn 18 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) |

---

Trang gốc là trang mục lục của phần *Tasks → TLS*: nội dung trang chỉ gồm danh sách các trang
con, được liệt kê dưới đây theo đúng thứ tự hiển thị trên trang web. Các trang này hướng dẫn
cách cấp phát (provision), quản lý và xoay vòng (rotate) certificate cho cluster của bạn.

## Danh sách các trang trong mục này (Pages in this section)

- [Cấp certificate cho một API client của Kubernetes bằng CertificateSigningRequest (Issue a Certificate for a Kubernetes API Client Using a CertificateSigningRequest)](397-certificate-issue-client-csr-vi.md)
- [Cấu hình xoay vòng certificate cho kubelet (Configure Certificate Rotation for the Kubelet)](398-certificate-rotation-vi.md)
- [Quản lý TLS certificate trong cluster (Manage TLS Certificates in a Cluster)](399-managing-tls-in-a-cluster-vi.md)
- [Xoay vòng CA certificate thủ công (Manual Rotation of CA Certificates)](400-manual-rotation-of-ca-certificates-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 18:

1. Nhóm `tls/` gồm bốn bài. Nêu ngắn gọn mỗi bài giải quyết việc gì khác nhau.
2. **Câu bẫy.** Bạn đã đọc bài [219](219-kubeadm-certs-vi.md) về `kubeadm certs`. Vậy nhóm `tls/`
   có phải chỉ là bản chi tiết hơn của bài đó không?
3. Trên cluster lab ba VM, certificate của `kube-apiserver` sắp hết hạn và certificate của
   kubelet trên `lab-k8s-worker2` cũng vậy. Hai thứ đó được xử lý bằng **hai con đường khác
   nhau** — nói rõ mỗi con đường thuộc bài nào trong bốn bài của nhóm, hoặc thuộc bài
   [219](219-kubeadm-certs-vi.md).

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **[397](397-certificate-issue-client-csr-vi.md)** — cấp certificate cho một **API client mới**
   (tạo danh tính người dùng). **[398](398-certificate-rotation-vi.md)** — bật cho **kubelet tự
   gia hạn** certificate của chính nó. **[399](399-managing-tls-in-a-cluster-vi.md)** — dùng
   **CertificateSigningRequest API và signer** của cluster để cấp certificate nói chung.
   **[400](400-manual-rotation-of-ca-certificates-vi.md)** — **xoay chính CA**, tức đổi gốc tin
   cậy của toàn cluster.
2. **Không.** Bài 219 nói về những certificate **kubeadm sinh ra và quản lý** — kiểm tra hạn,
   gia hạn bằng lệnh `kubeadm certs`. Nhóm `tls/` nói về **cơ chế certificate của bản thân
   Kubernetes** (CSR API, signer, kubelet rotation), tồn tại độc lập với kubeadm và dùng được
   trên cluster không do kubeadm dựng. Chỗ dễ nhầm: cả hai đều nói "certificate" và cùng nằm
   trong giai đoạn 18, nhưng một bên là công cụ của trình cài đặt, một bên là API của cluster.

3. **Hai con đường tách bạch.** Certificate của `kube-apiserver` là certificate **do kubeadm sinh
   ra và quản lý**, nên gia hạn bằng `kubeadm certs renew` — thuộc bài
   [219](219-kubeadm-certs-vi.md), không thuộc nhóm `tls/`. Certificate của kubelet trên
   `lab-k8s-worker2` thì đi đường **tự động gia hạn qua CSR API** của chính Kubernetes — thuộc
   bài [398](398-certificate-rotation-vi.md). Điểm mấu chốt: kubelet **tự xin** certificate mới
   và cluster ký cho nó, còn certificate của control plane thì **người vận hành phải ra lệnh
   gia hạn**.

</details>

Câu nào chưa trả lời được thì đọc lại trang mục trước khi sang bài
[219](219-kubeadm-certs-vi.md) và các bài còn lại của [Giai đoạn 18 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ).

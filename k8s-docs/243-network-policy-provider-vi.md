# Cài đặt một Network Policy Provider (Install a Network Policy Provider)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy),
bài 7/14 · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy).

Đây là **trang mục lục**, trang gốc không có nội dung riêng. Đọc trong vài phút, không dừng lâu.

**Phải hiểu ở lần đọc này:**

- Trang này trả lời đúng một câu hỏi: Kubernetes liệt kê những provider nào cho NetworkPolicy —
  Antrea, Calico, Cilium, kube-router, Romana và Weave Net. Nó **không** dạy cú pháp
  NetworkPolicy; phần đó nằm ở bài [84](84-network-policies-vi.md) và bài
  [201](201-declare-network-policy-vi.md) vừa đọc.
- Việc chọn provider thuộc tầng CNI, không thuộc object NetworkPolicy. Cluster lab đã có sẵn một
  CNI thực thi được NetworkPolicy từ [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md), nên
  bạn **không** cần cài thêm provider nào trong danh sách này.
- Mục Romana trong danh sách là **tài liệu lịch sử** — lộ trình đánh dấu như vậy ở bài kế tiếp
  ([248](248-romana-network-policy-vi.md)); đừng coi nó là một lựa chọn ngang hàng với năm mục còn lại.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Năm trang provider Antrea, Calico, Cilium, kube-router, Weave Net | mỗi trang là một quy trình cài CNI khác; đổi CNI sẽ phá snapshot cluster của [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) | mở đúng trang của provider bạn phải tiếp quản; lộ trình chỉ bắt đọc một trang provider là bài [248](248-romana-network-policy-vi.md) |
| Mục [Romana](248-romana-network-policy-vi.md) | là tài liệu lịch sử, không phải hướng dẫn còn dùng được | bài 8/14 ngay sau đây, đọc với đúng vai trò đó |

---

Đây là trang mục lục. Trang gốc không có nội dung riêng, chỉ liệt kê các trang hướng dẫn cài đặt
một trình cung cấp (provider) Network Policy:

* [Dùng Antrea cho NetworkPolicy (Use Antrea for NetworkPolicy)](244-antrea-network-policy-vi.md)
* [Dùng Calico cho NetworkPolicy (Use Calico for NetworkPolicy)](245-calico-network-policy-vi.md)
* [Dùng Cilium cho NetworkPolicy (Use Cilium for NetworkPolicy)](246-cilium-network-policy-vi.md)
* [Dùng Kube-router cho NetworkPolicy (Use Kube-router for NetworkPolicy)](247-kube-router-network-policy-vi.md)
* [Romana cho NetworkPolicy (Romana for NetworkPolicy)](248-romana-network-policy-vi.md)
* [Weave Net cho NetworkPolicy (Weave Net for NetworkPolicy)](249-weave-network-policy-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Trang này không có một dòng nào về cú pháp NetworkPolicy. Vậy nó trả lời câu hỏi nào mà bài
   [84](84-network-policies-vi.md) và bài [201](201-declare-network-policy-vi.md) không trả lời?
2. **Câu bẫy.** Cluster lab của bạn đã viết và kiểm chứng được NetworkPolicy từ
   [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md). Vậy để làm Checkpoint giai đoạn 21 —
   chặn toàn bộ ingress rồi mở đúng một cổng — bạn có phải cài thêm một provider trong danh sách
   này lên `lab-k8s-master` không?
3. Trong sáu mục của trang, mục nào lộ trình xếp là tài liệu lịch sử, và điều đó đổi cách bạn đọc
   nó thế nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó trả lời câu hỏi **"ai thực thi NetworkPolicy"** — tức **danh sách provider** mà Kubernetes
   liệt kê: Antrea, Calico, Cilium, kube-router, Romana, Weave Net. Hai bài kia dạy **viết** một
   object NetworkPolicy; trang này chỉ chỉ chỗ tìm hướng dẫn cài phần **thực thi** nó.
2. **Không.** Việc chọn provider thuộc tầng CNI, và CNI của cluster lab đã thực thi được
   NetworkPolicy từ Lab 5b. Cài thêm một provider trong danh sách nghĩa là **đổi CNI**, tức phá
   trạng thái cluster mà các lab sau dựa vào — không phải điều Checkpoint giai đoạn 21 yêu cầu.
3. **Romana.** Lộ trình đánh dấu nó là tài liệu lịch sử, nên đọc **chỉ để nhận diện khi tiếp
   quản cluster cũ còn dùng nó**, không đọc như một lựa chọn triển khai mới.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

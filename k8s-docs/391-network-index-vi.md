# Mạng (Networking)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/network/>
>
> Tìm hiểu cách cấu hình mạng (networking) cho cluster của bạn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy),
bài 10/14 · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy).

Đây là **trang mục lục** của nhóm `tasks/network/`, trang gốc không có nội dung kỹ thuật riêng.
Đọc trong vài phút để lấy bản đồ của bốn bài cuối giai đoạn 21.

**Phải hiểu ở lần đọc này:**

- Bốn trang con liệt kê ở mục *Danh sách các trang trong mục này* chính là **bài 11/14 đến 14/14**
  của giai đoạn 21, và thứ tự trên trang gốc trùng với thứ tự đọc của lộ trình.
- Ba việc mà đoạn mở đầu nêu ra cho cả nhóm: tùy biến việc **phân giải tên bên trong Pod**, **mở
  rộng và cấu hình lại dải IP dành cho Service**, và **kiểm chứng chế độ dual-stack IPv4/IPv6**.
- Hai trang [393](393-extend-service-ip-ranges-vi.md) và
  [394](394-reconfigure-default-service-ip-ranges-vi.md) là một **cặp** cùng nói về dải IP của
  Service nhưng làm hai việc khác nhau — *mở rộng* dải và *cấu hình lại* dải mặc định. Đọc đúng
  thứ tự đó, đừng trộn hai trang vào một khái niệm.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nội dung kỹ thuật của từng trang con | trang này chỉ là mục lục | lần lượt bài [392](392-customize-hosts-file-for-pods-vi.md), [393](393-extend-service-ip-ranges-vi.md), [394](394-reconfigure-default-service-ip-ranges-vi.md), [395](395-validate-dual-stack-vi.md) ngay sau đây |
| Cụm "kiểm chứng chế độ dual-stack IPv4/IPv6" ở đoạn mở đầu | cluster lab chạy thuần IPv4 theo [A1.2 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) | bài [395](395-validate-dual-stack-vi.md) — bài 14/14, nói rõ phần nào không đo được trên cluster lab |

---

Trang gốc là trang mục lục của phần *Tasks → Networking*: nội dung trang chỉ gồm danh sách các
trang con, được liệt kê dưới đây theo đúng thứ tự hiển thị trên trang web. Các trang này hướng
dẫn những tác vụ mạng ở mức cluster: tùy biến việc phân giải tên (name resolution) bên trong
Pod, mở rộng và cấu hình lại dải IP dành cho Service, cũng như kiểm chứng chế độ dual-stack
IPv4/IPv6.

## Danh sách các trang trong mục này (Pages in this section)

- [Thêm entry vào /etc/hosts của Pod bằng HostAliases (Adding entries to Pod /etc/hosts with HostAliases)](392-customize-hosts-file-for-pods-vi.md)
- [Mở rộng dải IP của Service (Extend Service IP Ranges)](393-extend-service-ip-ranges-vi.md)
- [Cấu hình lại ServiceCIDR mặc định của Kubernetes (Kubernetes Default ServiceCIDR Reconfiguration)](394-reconfigure-default-service-ip-ranges-vi.md)
- [Kiểm chứng dual-stack IPv4/IPv6 (Validate IPv4/IPv6 dual-stack)](395-validate-dual-stack-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Bốn trang con của mục này gom về mấy nhóm việc, và mỗi nhóm giải quyết chuyện gì ở mức cluster?
2. **Câu bẫy.** Hai trang [393](393-extend-service-ip-ranges-vi.md) và
   [394](394-reconfigure-default-service-ip-ranges-vi.md) đều nói về dải IP của Service. Chúng là
   hai cách viết của cùng một việc, hay là hai việc khác nhau? Đọc kỹ tiêu đề rồi trả lời.
3. Cluster lab chạy thuần IPv4 với Service CIDR mặc định của kubeadm. Trong bốn trang con, trang
   nào bạn sẽ **không** kiểm chứng đầy đủ được trên `lab-k8s-master` và hai worker?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Ba nhóm**, đúng như đoạn mở đầu của trang liệt kê: **tùy biến phân giải tên bên trong Pod**
   (trang [392](392-customize-hosts-file-for-pods-vi.md)); **mở rộng và cấu hình lại dải IP dành
   cho Service** (hai trang [393](393-extend-service-ip-ranges-vi.md) và
   [394](394-reconfigure-default-service-ip-ranges-vi.md)); **kiểm chứng dual-stack IPv4/IPv6**
   (trang [395](395-validate-dual-stack-vi.md)).
2. **Hai việc khác nhau.** Tiêu đề trang 393 là *mở rộng* dải IP của Service, tiêu đề trang 394 là
   *cấu hình lại* ServiceCIDR **mặc định**. Gộp chúng làm một là hiểu sai ngay từ đầu — trang 394
   còn được lộ trình ghi rõ là phải làm **sau** trang 393.
3. Trang [395](395-validate-dual-stack-vi.md) — **kiểm chứng dual-stack IPv4/IPv6**. Cluster lab
   chỉ có IPv4, nên phần đo đạc IPv6 của trang đó không có gì để đối chiếu trên ba VM.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

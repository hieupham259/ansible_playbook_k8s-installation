# Cấu hình DNS cho một cluster (Configure DNS for a Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/access-application-cluster/configure-dns-cluster/>
>
> Trang này giới thiệu addon DNS cho cluster của Kubernetes và nơi tìm hướng dẫn
> cấu hình chi tiết.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), dòng **Thực
hành**, bài 7/10 · Kiểm chứng ở [Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) phần B0.3 và
B4.1 — xác nhận addon DNS của cluster là CoreDNS do kubeadm cài, Service của nó tên `kube-dns`, và
`nameserver` trong Pod trỏ đúng ClusterIP đó.

Trang này chỉ dài ba câu và là một **trang cửa ngõ**: nó nói cluster của bạn đã có sẵn addon DNS
nào, rồi đẩy toàn bộ phần cấu hình sang bài [204](204-dns-custom-nameservers-vi.md). Lab 5a cũng
ghi rõ lab chỉ **đọc** addon mặc định, không sửa ConfigMap của CoreDNS.

**Phải hiểu ở lần đọc này:**

- Kubernetes cung cấp một **addon DNS cho cluster**, và ở hầu hết môi trường được hỗ trợ thì addon
  đó **bật mặc định** — bạn không phải cài thêm để Pod phân giải được tên Service.
- Từ phiên bản 1.11 trở đi, **CoreDNS** là lựa chọn được khuyến nghị và là thứ **kubeadm cài mặc
  định** — tức đúng cái đang chạy trên cluster lab của bạn.
- Trang này **không** chứa hướng dẫn cấu hình nào; mọi chi tiết nằm ở bài
  [204 — Tùy chỉnh DNS Service](204-dns-custom-nameservers-vi.md).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Toàn bộ phần cấu hình CoreDNS mà bài trỏ sang | sửa Corefile là đụng vào `kube-system`; ở giai đoạn 5 bạn chỉ đọc addon mặc định | bài [204 — Tùy chỉnh DNS Service](204-dns-custom-nameservers-vi.md), [giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy) |
| Ý "được cài đặt mặc định cùng với kubeadm" — addon được cài ở bước nào của quá trình dựng cluster | bạn chưa tự chạy `kubeadm init`, nên chưa thấy bước cài addon | [Lab 8a](labs/LAB-8A-DUNG-CLUSTER-BANG-KUBEADM.md) phần B7.6, [giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) |

---

Kubernetes cung cấp một addon DNS cho cluster, mà hầu hết các môi trường được hỗ trợ đều
bật mặc định. Từ Kubernetes phiên bản 1.11 trở đi, CoreDNS là lựa chọn được khuyến nghị
và được cài đặt mặc định cùng với kubeadm.

Để biết thêm thông tin về cách cấu hình CoreDNS cho một cluster Kubernetes, xem
[Tùy chỉnh DNS Service](204-dns-custom-nameservers-vi.md)
(đã có [bản dịch tiếng Việt](204-dns-custom-nameservers-vi.md)).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Cluster lab của bạn được dựng bằng kubeadm. Để một Pod trên `lab-k8s-worker1` phân giải được tên
   một Service, bạn phải cài thêm thành phần nào không? Vì sao?
2. **Câu bẫy.** Trang tên là "Cấu hình DNS cho một cluster". Vậy nó hướng dẫn bạn sửa cấu hình DNS ở
   đâu, và bạn phải sang đâu để thực sự làm việc đó?
3. Bạn tiếp quản một cluster lạ. Addon DNS của nó có chắc là CoreDNS không? Bài cho bạn căn cứ gì để
   trả lời?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không phải cài gì cả.** Bài nói Kubernetes cung cấp một addon DNS cho cluster, và "hầu hết các
   môi trường được hỗ trợ đều bật mặc định"; riêng với kubeadm thì CoreDNS "được cài đặt mặc định"
   cùng công cụ này. Cluster lab dựng bằng kubeadm nên addon đã có sẵn từ lúc cluster hình thành.
2. **Nó không hướng dẫn gì cả** — đây là điểm dễ hụt hẫng khi mở trang ra. Toàn bộ nội dung là giới
   thiệu addon DNS và một câu chỉ đường: muốn cấu hình CoreDNS thì sang bài
   [204 — Tùy chỉnh DNS Service](204-dns-custom-nameservers-vi.md). Tên trang mô tả **chủ đề**, không
   mô tả nội dung.
3. **Không chắc.** Bài chỉ khẳng định rằng *từ phiên bản 1.11 trở đi* CoreDNS là lựa chọn **được
   khuyến nghị** và là thứ kubeadm cài mặc định — hai điều kiện, chứ không phải một lời khẳng định
   rằng mọi cluster đều chạy CoreDNS. Cluster cũ hơn, hoặc cluster dựng bằng công cụ khác, có thể
   mang addon DNS khác; và bài cũng chỉ nói "hầu hết các môi trường **được hỗ trợ**" mới bật addon
   mặc định. Việc cần làm là đọc thẳng addon đang chạy trong `kube-system`, đúng như Lab 5a làm ở
   phần B0.3.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

# Sử dụng Antrea cho NetworkPolicy (Use Antrea for NetworkPolicy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/antrea-network-policy/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy)
— **trang con của bài 7/14**, mục [Cài đặt một Network Policy
Provider](243-network-policy-provider-vi.md); nó không có dòng riêng trong lộ trình. Phần II không
có lab: thực hành thẳng trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm
bằng **Checkpoint** ở cuối [mục giai đoạn
21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy).

Cluster lab **đã chạy Calico** từ [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) — đó là
lab đổi CNI và trả nợ *NetworkPolicy được thực thi thật*. Vì vậy trong năm trang provider, chỉ bài
[245](245-calico-network-policy-vi.md) khớp cluster của bạn. Trang này **không được cài** vào
cluster lab: đổi CNI sẽ phá [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot) mà các lab sau dựa
vào. Đọc để biết **có lựa chọn nào và nó khác các lựa chọn kia ở đâu**, không để làm theo.

**Phải hiểu ở lần đọc này:**

- Trang này **không chứa một lệnh nào**. Cả ba mục đều là con trỏ ra ngoài: nền tảng dự án ở
  *Introduction to Antrea*, còn quy trình triển khai nằm trong hướng dẫn *Getting Started* của
  repo Antrea. Đặc trưng của Antrea trong nhóm sáu provider là ở chỗ đó — Kubernetes chỉ ghi nhận
  nó là một **plugin CNI**, phần cài đặt giao hết cho tài liệu của dự án.
- Thứ tự mà mục *Trước khi bạn bắt đầu* và mục *Triển khai Antrea với kubeadm* đặt ra: **cluster
  dựng bằng kubeadm trước, Antrea sau**. Antrea không phải một phần của kubeadm; nó là thứ bạn
  thêm vào một cluster đã có.
- Mục *Tiếp theo* khép vòng bằng bài [201](201-declare-network-policy-vi.md) — cùng một bài kiểm
  chứng dùng chung cho mọi provider trong nhóm. Object NetworkPolicy không đổi theo provider; chỉ
  phần **thực thi** nó là đổi.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hướng dẫn *Getting Started* của repo Antrea mà mục *Triển khai Antrea với kubeadm* trỏ tới | đó là quy trình cài một CNI thật | **không chạy trên cluster lab** — cluster đã dùng Calico từ [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) và đổi CNI phá [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot). Chỉ mở khi bạn tiếp quản một cluster đã chạy Antrea |
| *Introduction to Antrea* — tài liệu nền của dự án | là tài liệu của một dự án bên ngoài, không thuộc phạm vi lộ trình | ranh giới "Kubernetes định nghĩa gì, plugin hiện thực gì" đã đọc ở bài [183](183-network-plugins-vi.md), thực hành ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) phần B2 |

---

Trang này hướng dẫn cách cài đặt và sử dụng plugin CNI Antrea trên Kubernetes.
Để tìm hiểu thông tin nền về Project Antrea, hãy đọc
[Introduction to Antrea](https://antrea.io/docs/).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes. Hãy làm theo
[hướng dẫn bắt đầu với kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
để dựng một cluster.

## Triển khai Antrea với kubeadm (Deploying Antrea with kubeadm)

Hãy làm theo hướng dẫn
[Getting Started](https://github.com/vmware-tanzu/antrea/blob/main/docs/getting-started.md)
để triển khai Antrea cho kubeadm.

## Tiếp theo (What's next)

Khi cluster của bạn đã chạy, bạn có thể làm theo bài
[Khai báo Network Policy](201-declare-network-policy-vi.md) để thử nghiệm
NetworkPolicy của Kubernetes.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Cả trang không có lấy một lệnh `kubectl` hay `apply`. Vậy nó đóng vai trò gì trong nhóm trang
   provider, và ai giữ phần quy trình cài đặt thật?
2. Tiêu đề mục cài đặt gọi thẳng tên kubeadm. Điều đó có nghĩa Antrea là một thành phần **của**
   kubeadm không, hay là thứ đến **sau** kubeadm? Căn cứ vào đâu trong bài để trả lời?
3. **Câu bẫy.** Mục *Tiếp theo* bảo: cluster chạy rồi thì làm theo bài
   [201](201-declare-network-policy-vi.md) để thử NetworkPolicy. Bạn đã làm đúng việc đó trên
   cluster lab. Vậy đọc xong trang này bạn có phải chạy hướng dẫn *Getting Started* của Antrea lên
   `lab-k8s-master` để "làm theo bài cho đủ" không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó là **một con trỏ**, không phải một hướng dẫn. Trang chỉ ghi nhận Antrea là một plugin CNI
   dùng được cho NetworkPolicy rồi **giao toàn bộ quy trình cho tài liệu của chính dự án Antrea**
   — phần nền ở *Introduction to Antrea*, phần triển khai ở hướng dẫn *Getting Started* trong repo
   Antrea. Đó cũng là đặc trưng phân biệt trang này với các trang provider khác trong nhóm: nó
   ngắn nhất và không giữ lại thao tác nào.
2. **Đến sau.** Mục *Trước khi bạn bắt đầu* đặt điều kiện là **đã có một cluster Kubernetes**, dựng
   theo hướng dẫn kubeadm; mục kế mới nói tới việc *triển khai Antrea* cho cluster đó. Tên kubeadm
   trong tiêu đề chỉ nói **loại cluster** mà hướng dẫn nhắm tới, không nói Antrea nằm trong
   kubeadm.
3. **Không.** Cài Antrea nghĩa là **đổi CNI** của cluster lab, trong khi cluster đã chạy Calico từ
   [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) và đã kiểm chứng được NetworkPolicy chặn
   thật. Trang này ở lần đọc giai đoạn 21 là **để biết lựa chọn**, không phải để thi công: đổi CNI
   phá [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot) mà các lab sau dựa vào. Chỗ dễ sai là đọc
   mục *Tiếp theo* như một mệnh lệnh — nó viết cho người **vừa cài Antrea**, không viết cho người
   đã có sẵn một provider khác.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

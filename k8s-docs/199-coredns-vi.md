# Dùng CoreDNS cho khám phá dịch vụ (Using CoreDNS for Service Discovery)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/coredns/>
>
> Trang này mô tả quy trình nâng cấp CoreDNS và cách cài đặt CoreDNS thay cho kube-dns.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy),
bài 2/14 · Phần II không có lab riêng: kiểm chứng bằng **Checkpoint của chính giai đoạn 21** trên
cluster lab dựng ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) — sửa Corefile của CoreDNS thêm một
domain chuyển tiếp rồi kiểm chứng bằng `nslookup` từ trong Pod.

Bài nối tiếp phần lý thuyết DNS ở bài [10](10-dns-pod-service-vi.md) và liên quan tới
[Giai đoạn 17 — Nâng cấp cluster](00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster) vì
`kubeadm upgrade` tự xử lý CoreDNS.

Đây là trang rất ngắn, phần lớn là chỉ đường sang tài liệu của dự án CoreDNS trên GitHub.
Với cluster lab dựng bằng kubeadm, bạn không phải làm gì thủ công: kubeadm đã cài CoreDNS
và sẽ giữ cấu hình khi nâng cấp.

**Phải hiểu ở lần đọc này:**

- CoreDNS là DNS server đảm nhiệm vai trò cluster DNS, là dự án của CNCF, và từ Kubernetes
  1.21 nó là ứng dụng DNS **duy nhất** mà kubeadm hỗ trợ — kube-dns đã bị loại bỏ (mục
  *Chuyển sang CoreDNS*).
- Khi nâng cấp một cluster cũ còn dùng kube-dns bằng kubeadm, kubeadm tự sinh cấu hình
  CoreDNS ("Corefile") từ ConfigMap của kube-dns, giữ lại stub domain và upstream name
  server (mục *Nâng cấp cluster hiện có bằng kubeadm*).
- Khi nâng cấp cluster, điều phải bảo đảm là **giữ lại Corefile hiện có**; nếu nâng cấp bằng
  `kubeadm` thì kubeadm tự lo việc này (mục *Nâng cấp CoreDNS*).
- Phiên bản CoreDNS mà kubeadm cài gắn với từng phiên bản Kubernetes — tra ở trang
  *CoreDNS version in Kubernetes* trên GitHub, không tự chọn tùy ý.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
|---|---|---|
| *Cài đặt CoreDNS* (triển khai thủ công thay kube-dns) | Cluster lab do kubeadm dựng đã có sẵn CoreDNS; triển khai thủ công chỉ cần khi không dùng kubeadm | Chỉ khi vận hành cluster không dựng bằng kubeadm — ngoài phạm vi lộ trình |
| *Tinh chỉnh CoreDNS* và các link scaling trên GitHub | Cần số liệu tải thật mới có ý nghĩa | giai đoạn 21 — bài [Autoscale the DNS Service](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy) |
| Sửa Corefile để thêm use case mới (mục *Tiếp theo*) | Thao tác sửa Corefile cụ thể được dạy trong bài Customizing DNS Service | giai đoạn 21 — bài [Customizing DNS Service](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy) |

---

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.9 hoặc mới hơn. Để kiểm tra phiên bản, hãy
chạy `kubectl version`.

## Giới thiệu về CoreDNS (About CoreDNS)

[CoreDNS](https://coredns.io) là một DNS server linh hoạt, có khả năng mở rộng
(extensible), có thể đảm nhiệm vai trò DNS của cluster Kubernetes. Giống như Kubernetes,
dự án CoreDNS được CNCF chủ trì.

Bạn có thể dùng CoreDNS thay cho kube-dns trong cluster của mình bằng cách thay thế
kube-dns trong một deployment hiện có, hoặc dùng các công cụ như kubeadm — công cụ sẽ
triển khai và nâng cấp cluster giúp bạn.

## Cài đặt CoreDNS (Installing CoreDNS)

Để triển khai thủ công hoặc thay thế kube-dns, hãy xem tài liệu tại
[website CoreDNS](https://coredns.io/manual/installation/).

## Chuyển sang CoreDNS (Migrating to CoreDNS)

### Nâng cấp cluster hiện có bằng kubeadm (Upgrading an existing cluster with kubeadm)

Ở Kubernetes phiên bản 1.21, kubeadm đã loại bỏ hỗ trợ `kube-dns` với vai trò ứng dụng
DNS. Với `kubeadm` v1.36, ứng dụng DNS duy nhất được hỗ trợ cho cluster là CoreDNS.

Bạn có thể chuyển sang CoreDNS khi dùng `kubeadm` để nâng cấp một cluster đang sử dụng
`kube-dns`. Trong trường hợp này, `kubeadm` sinh cấu hình CoreDNS ("Corefile") dựa trên
ConfigMap của `kube-dns`, giữ nguyên các cấu hình về stub domain và upstream name server.

## Nâng cấp CoreDNS (Upgrading CoreDNS)

Bạn có thể tra phiên bản CoreDNS mà kubeadm cài đặt cho từng phiên bản Kubernetes tại trang
[CoreDNS version in Kubernetes](https://github.com/coredns/deployment/blob/master/kubernetes/CoreDNS-k8s_version.md).

CoreDNS có thể được nâng cấp thủ công trong trường hợp bạn chỉ muốn nâng cấp riêng CoreDNS
hoặc muốn dùng image tùy chỉnh của mình. Có sẵn một
[hướng dẫn và quy trình từng bước](https://github.com/coredns/deployment/blob/master/kubernetes/Upgrading_CoreDNS.md)
hữu ích để bảo đảm việc nâng cấp diễn ra suôn sẻ. Hãy chắc chắn rằng cấu hình CoreDNS hiện
có ("Corefile") được giữ lại khi nâng cấp cluster của bạn.

Nếu bạn nâng cấp cluster bằng công cụ `kubeadm`, `kubeadm` có thể tự động lo việc giữ lại
cấu hình CoreDNS hiện có.

## Tinh chỉnh CoreDNS (Tuning CoreDNS)

Khi mức sử dụng tài nguyên là một mối quan tâm, việc tinh chỉnh cấu hình CoreDNS có thể
hữu ích. Để biết thêm chi tiết, hãy xem
[tài liệu về scaling CoreDNS](https://github.com/coredns/deployment/blob/master/kubernetes/Scaling_CoreDNS.md).

## Tiếp theo (What's next)

Bạn có thể cấu hình [CoreDNS](https://coredns.io) để hỗ trợ nhiều use case hơn hẳn so với
kube-dns bằng cách sửa cấu hình CoreDNS ("Corefile"). Để biết thêm thông tin, hãy xem
[tài liệu](https://coredns.io/plugins/kubernetes/) của plugin CoreDNS `kubernetes`, hoặc
đọc bài
[Custom DNS Entries for Kubernetes](https://coredns.io/2017/05/08/custom-dns-entries-for-kubernetes/)
trên blog của CoreDNS.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở checkpoint này:

1. Trên cluster lab của bạn (dựng bằng kubeadm), bạn tiếp quản thêm một cluster cũ vẫn chạy
   `kube-dns`. Có thể dùng `kubeadm` phiên bản hiện hành để nâng cấp và giữ nguyên
   `kube-dns` không? Vì sao?
2. Khi kubeadm chuyển một cluster từ kube-dns sang CoreDNS, cấu hình stub domain và
   upstream name server mà đội cũ đã đặt trong ConfigMap của kube-dns có mất không? Cơ chế
   nào quyết định điều đó?
3. Bạn muốn nâng cấp riêng CoreDNS lên một phiên bản mới hơn phiên bản kubeadm cài kèm.
   Điều này có được phép không, và thứ gì bạn bắt buộc phải giữ lại trong quá trình đó?
4. Câu bẫy: CoreDNS là một thành phần mã nguồn nằm trong repo Kubernetes, phiên bản của nó
   luôn trùng với phiên bản cluster — đúng hay sai?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Từ Kubernetes 1.21, kubeadm đã loại bỏ hỗ trợ `kube-dns`; với kubeadm hiện
   hành, ứng dụng DNS duy nhất được hỗ trợ là CoreDNS. Nâng cấp bằng kubeadm đồng nghĩa
   cluster sẽ chuyển sang CoreDNS.
2. **Không mất.** Khi nâng cấp, `kubeadm` sinh Corefile **dựa trên ConfigMap của
   `kube-dns`**, và giữ nguyên các cấu hình stub domain cùng upstream name server — đây là
   hành vi được bài nêu rõ, không cần thao tác thủ công.
3. **Được phép.** Bài nói rõ CoreDNS có thể được nâng cấp thủ công khi bạn chỉ muốn nâng
   cấp riêng CoreDNS hoặc dùng image tùy chỉnh, theo hướng dẫn Upgrading CoreDNS trên
   GitHub. Thứ bắt buộc phải giữ lại là **cấu hình CoreDNS hiện có ("Corefile")**.
4. **Sai.** CoreDNS là dự án độc lập do CNCF chủ trì (giống Kubernetes nhưng không nằm
   trong Kubernetes), có phiên bản riêng. Mỗi phiên bản kubeadm cài kèm một phiên bản
   CoreDNS xác định — phải tra bảng *CoreDNS version in Kubernetes*, không suy từ số phiên
   bản cluster. Trực giác "cùng một số phiên bản" sai vì hai dự án phát hành độc lập.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

# Romana cho NetworkPolicy (Romana for NetworkPolicy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/romana-network-policy/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy),
bài 8/14 · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy). Riêng bài này
**không sinh ra bước thực hành nào**.

Lộ trình xếp bài này vào diện **"đọc như tài liệu lịch sử"**: Romana đã ngừng phát triển. Đọc để
nhận diện khi tiếp quản một cluster cũ còn dùng nó, **không** đọc như hướng dẫn triển khai còn
dùng được. Đây là trang provider duy nhất trong danh sách của bài
[243](243-network-policy-provider-vi.md) mà lộ trình bắt đọc.

**Phải hiểu ở lần đọc này:**

- Vai trò của bài trong lộ trình: nhận diện, không triển khai. Không có bước nào ở đây được chạy
  lên cluster lab.
- Dấu hiệu của một tài liệu đã bị bỏ rơi, thấy ngay ở cấu trúc bài: cả ba mục *Trước khi bạn bắt
  đầu*, *Cài đặt Romana với kubeadm* và *Áp dụng network policy* đều **chỉ trỏ ra link ngoài** —
  bài không tự chứa một quy trình nào.
- Chi tiết kỹ thuật duy nhất đáng nhớ, ở mục *Áp dụng network policy*: có **hai** đường viết
  policy — *Romana network policies* theo định dạng riêng của Romana, và **NetworkPolicy API**
  của Kubernetes. Khi tiếp quản cluster cũ, phải biết quy tắc đang được viết bằng đường nào.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Cài đặt Romana với kubeadm* và các link tới repo `romana/romana` | quy trình cài của một dự án đã ngừng phát triển | không có bước nào phải làm — CNI thực thi NetworkPolicy của cluster lab đã chốt ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) |
| Định dạng *Romana network policies* và trang ví dụ của nó | định dạng riêng của một sản phẩm, không phải API Kubernetes | mở đúng hai link đó **chỉ khi** bạn thật sự tiếp quản một cluster còn chạy Romana |
| Mục *Tiếp theo* dẫn sang bài [201](201-declare-network-policy-vi.md) | bài đó đã đọc ở bước 6/14 của chính giai đoạn 21 | bài [201](201-declare-network-policy-vi.md), đã đọc trước bài này |

---

Trang này hướng dẫn cách sử dụng Romana cho NetworkPolicy.

## Trước khi bạn bắt đầu (Before you begin)

Hoàn thành các bước 1, 2 và 3 trong [hướng dẫn bắt đầu với kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/).

## Cài đặt Romana với kubeadm (Installing Romana with kubeadm)

Làm theo [hướng dẫn cài đặt dạng container hóa](https://github.com/romana/romana/tree/master/containerize) dành cho kubeadm.

## Áp dụng network policy (Applying network policies)

Để áp dụng network policy, hãy dùng một trong các cách sau:

* [Romana network policies](https://github.com/romana/romana/wiki/Romana-policies).
    * [Ví dụ về Romana network policy](https://github.com/romana/core/blob/master/doc/policy.md).
* NetworkPolicy API.

## Tiếp theo (What's next)

Sau khi đã cài đặt Romana, bạn có thể làm theo bài [Khai báo Network Policy](201-declare-network-policy-vi.md) để thử nghiệm NetworkPolicy của Kubernetes.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Nhìn vào cấu trúc bài — ba mục nội dung của nó chứa gì? Dấu hiệu đó nói lên điều gì về tình
   trạng của tài liệu này, và vì sao lộ trình xếp nó vào diện "đọc như tài liệu lịch sử"?
2. **Câu bẫy.** Bạn tiếp quản một cluster cũ đang chạy Romana và thấy lưu lượng giữa các Pod bị
   chặn theo quy tắc nào đó. Chạy `kubectl get networkpolicy --all-namespaces` là đủ để nhìn thấy
   toàn bộ quy tắc đang có hiệu lực chưa?
3. Cluster lab gồm `lab-k8s-master`, `lab-k8s-worker1` và `lab-k8s-worker2` đã có CNI thực thi
   NetworkPolicy từ [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md). Bài này sinh ra bước
   nào bạn phải chạy trên ba máy đó không? Vì sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Cả ba mục — *Trước khi bạn bắt đầu*, *Cài đặt Romana với kubeadm*, *Áp dụng network policy* —
   **chỉ chứa link ra ngoài**, không chứa một bước nào của riêng nó. **Đó là dấu hiệu của tài
   liệu đã bị bỏ rơi**: nội dung thật nằm ở repo của sản phẩm, mà sản phẩm thì đã ngừng phát
   triển. Vì thế lộ trình chỉ dùng nó để **nhận diện**, không để triển khai.
2. **Chưa chắc đủ.** Bài nêu **hai** đường áp dụng policy: *Romana network policies* theo định
   dạng riêng của Romana, và NetworkPolicy API của Kubernetes. `kubectl get networkpolicy` chỉ
   thấy đường thứ hai. Quy tắc viết bằng định dạng riêng của Romana **nằm ngoài API Kubernetes**
   nên không hiện ra ở đó — phải tra bằng công cụ của chính Romana.
3. **Không có bước nào.** Bài này chỉ đọc để nhận diện, và cài Romana đồng nghĩa với **đổi CNI**
   của cluster — trong khi CNI thực thi NetworkPolicy đã được chốt từ Lab 5b và các lab sau dựa
   vào trạng thái đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

# Sử dụng Kube-router cho NetworkPolicy (Use Kube-router for NetworkPolicy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/kube-router-network-policy/

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

Đây là trang **ngắn nhất** nhóm nhưng lại là trang mô tả **cơ chế** rõ nhất. Cluster lab đã chạy
Calico từ [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md), nên chỉ bài
[245](245-calico-network-policy-vi.md) khớp cluster của bạn. **Không cài kube-router** vào cluster
lab: đổi CNI sẽ phá [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot) mà các lab sau dựa vào. Đọc
để biết kube-router khác các provider kia ở đâu, không để làm theo.

**Phải hiểu ở lần đọc này:**

- Đặc trưng của trang này là nó nói thẳng ra **vòng điều khiển** của provider: addon kube-router
  đi kèm một **Network Policy Controller** *theo dõi (watch)* Kubernetes API server để phát hiện
  mọi thay đổi về **NetworkPolicy và Pod**. Hai đối tượng, không phải một — vì tập Pod khớp
  selector thay đổi thì quy tắc cũng phải đổi theo.
- Kết quả của vòng điều khiển đó là **quy tắc `iptables` và `ipset`** trên node, cho phép hoặc
  chặn lưu lượng đúng theo chỉ dẫn của policy. kube-router là provider duy nhất trong nhóm nêu tên
  **`ipset`** bên cạnh `iptables`.
- Mục *Trước khi bạn bắt đầu* không ràng buộc trình cài đặt: cluster dựng bằng **Kops, Bootkube,
  Kubeadm hay bất kỳ trình cài đặt nào** đều dùng được, và quy trình cài giao cho tài liệu *thử
  Kube-router với các trình cài đặt cluster* của chính dự án.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hướng dẫn *thử Kube-router với các trình cài đặt cluster* mà mục *Cài đặt addon Kube-router* trỏ tới | đó là quy trình cài một CNI thật | **không chạy trên cluster lab** — cluster đã dùng Calico từ [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) và đổi CNI phá [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot). Chỉ mở khi bạn tiếp quản một cluster đã chạy kube-router |
| Đọc thẳng các quy tắc `iptables` và `ipset` mà provider sinh ra trên node | cluster lab thực thi policy bằng Calico nên không có quy tắc của kube-router để đối chiếu | ranh giới "Kubernetes định nghĩa object, plugin hiện thực nó" đã đọc ở bài [183](183-network-plugins-vi.md) và kiểm chứng bằng lưu lượng ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) phần B3 và B5 |

---

Trang này hướng dẫn cách sử dụng [Kube-router](https://github.com/cloudnativelabs/kube-router) cho NetworkPolicy.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes đang chạy. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng bất kỳ trình cài đặt cluster nào như Kops, Bootkube, Kubeadm, v.v.

## Cài đặt addon Kube-router (Installing Kube-router addon)

Addon Kube-router đi kèm một Network Policy Controller theo dõi (watch) Kubernetes API server để phát hiện mọi thay đổi về NetworkPolicy và pod, rồi cấu hình các quy tắc iptables và ipset nhằm cho phép hoặc chặn lưu lượng theo đúng chỉ dẫn của các policy. Vui lòng làm theo hướng dẫn [thử Kube-router với các trình cài đặt cluster](https://www.kube-router.io/docs/user-guide/#try-kube-router-with-cluster-installers) để cài đặt addon Kube-router.

## Tiếp theo (What's next)

Sau khi đã cài đặt addon Kube-router, bạn có thể làm theo bài [Khai báo Network Policy](201-declare-network-policy-vi.md) để thử nghiệm NetworkPolicy của Kubernetes.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Network Policy Controller của kube-router theo dõi **những loại đối tượng nào** trên API
   server, và vì sao chỉ theo dõi NetworkPolicy thôi là không đủ để giữ đúng quy tắc trên node?
2. **Câu bẫy.** Trang không có lệnh cài nào, và mục *Trước khi bạn bắt đầu* nói dùng trình cài đặt
   cluster nào cũng được — Kops, Bootkube, Kubeadm. Vậy có suy ra được rằng kube-router là **một
   tùy chọn có sẵn** của các trình cài đặt đó không?
3. Cluster lab của bạn đang thực thi NetworkPolicy bằng Calico, và bạn đã viết một file YAML
   NetworkPolicy ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md). Nếu một ngày cluster đổi
   sang kube-router, **thứ gì trong file đó phải sửa** và thứ gì đổi ở phía dưới?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Theo dõi **cả NetworkPolicy và Pod** — bài viết đúng như vậy. Chỉ theo dõi NetworkPolicy là
   không đủ vì policy trỏ vào Pod qua **selector**: cùng một policy không đổi, nhưng Pod mới sinh
   ra hoặc bị xóa thì **tập địa chỉ mà quy tắc phải áp lên đã khác**. Muốn quy tắc `iptables` và
   `ipset` trên node luôn khớp thực tế thì phải nhìn cả hai loại đối tượng.
2. **Không.** Bài chỉ nói kube-router **không kén trình cài đặt** — cluster dựng bằng Kops,
   Bootkube hay Kubeadm đều dùng được — rồi giao việc cài cho hướng dẫn *thử Kube-router với các
   trình cài đặt cluster* của chính dự án. Chỗ dễ nhầm: "chạy được với mọi trình cài đặt" là nói
   về **tương thích**, không phải nói kube-router có sẵn bên trong trình cài đặt nào.
3. **Không phải sửa gì trong file YAML.** NetworkPolicy là object API chuẩn của Kubernetes; mọi
   provider trong nhóm này đều khép vòng bằng cùng một bài [201](201-declare-network-policy-vi.md)
   để thử **chính object đó**. Thứ đổi nằm bên dưới: phần **thực thi** — thay vì Calico, sẽ là
   Network Policy Controller của kube-router dịch policy thành quy tắc `iptables` và `ipset` trên
   node. Và trên cluster lab thì **không đổi**: đó là đổi CNI, phá
   [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot).

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

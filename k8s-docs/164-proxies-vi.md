# Các loại proxy trong Kubernetes (Proxies in Kubernetes)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/cluster-administration/proxies/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 5](LO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), bài 16/16 · Kiểm chứng
ở Lab 5b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài cuối của giai đoạn, và ngắn nhất. Nó không dạy gì mới — nó **xếp lại năm thứ đều được gọi
là "proxy"** để bạn không nhầm chúng với nhau. Từ "proxy" đã xuất hiện ở rất nhiều bài trước với
những nghĩa khác nhau; đọc bài này để đóng lại giai đoạn 5 cho gọn.

**Phải hiểu ở lần đọc này:**

- Năm loại proxy và **nơi mỗi loại chạy**: `kubectl proxy` (trên máy người dùng hoặc trong một
  pod), apiserver proxy (trong chính tiến trình apiserver), kube-proxy (trên **mỗi node**),
  proxy/load balancer đứng trước apiserver, và cloud load balancer cho Service `type:
  LoadBalancer`.
- **kube-proxy không hiểu HTTP.** Nó proxy UDP, TCP và SCTP, có cân bằng tải, và **chỉ được dùng
  để truy cập các service**.
- `kubectl proxy`: client nói **HTTP** với nó, nó nói **HTTPS** với apiserver, tự định vị
  apiserver và **thêm các header xác thực** thay bạn.
- apiserver proxy là một **bastion tích hợp sẵn trong apiserver**, nối người dùng ngoài cluster
  tới các cluster IP vốn không truy cập được, dùng được cho **Node, Pod hoặc Service**, và **cân
  bằng tải khi truy cập một Service**.
- Ranh giới trách nhiệm: người dùng thường chỉ cần quan tâm hai loại đầu, quản trị viên cluster
  lo ba loại còn lại. Và **chuyển hướng (redirect) đã bị loại bỏ**, proxy thay thế nó.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chi tiết loại 4 — proxy/load balancer đứng trước nhiều apiserver | chỉ có nghĩa với control plane HA nhiều apiserver | giai đoạn 8 |
| Chi tiết loại 5 — cloud load balancer, mức hỗ trợ SCTP tùy nhà cung cấp | cluster lab không có cloud provider | không cần |

---

Trang này giải thích các loại proxy được sử dụng với Kubernetes.

## Proxy (Proxies)

Có một số loại proxy khác nhau mà bạn có thể gặp khi sử dụng Kubernetes:

1.  [kubectl proxy](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#directly-accessing-the-rest-api):

    - chạy trên máy tính của người dùng hoặc trong một pod
    - proxy từ một địa chỉ localhost đến Kubernetes apiserver
    - kết nối từ client đến proxy dùng HTTP
    - kết nối từ proxy đến apiserver dùng HTTPS
    - tự định vị apiserver
    - thêm các header xác thực (authentication headers)

2.  [apiserver proxy](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster-services/#discovering-builtin-services):

    - là một bastion được tích hợp sẵn trong apiserver
    - kết nối người dùng ở bên ngoài cluster đến các cluster IP mà nếu không có nó thì có thể không truy cập được
    - chạy trong các tiến trình của apiserver
    - kết nối từ client đến proxy dùng HTTPS (hoặc HTTP nếu apiserver được cấu hình như vậy)
    - kết nối từ proxy đến đích có thể dùng HTTP hoặc HTTPS, do proxy lựa chọn dựa trên thông tin sẵn có
    - có thể được dùng để truy cập một Node, Pod hoặc Service
    - thực hiện cân bằng tải (load balancing) khi được dùng để truy cập một Service

3.  [kube proxy](https://kubernetes.io/docs/concepts/services-networking/service/#ips-and-vips):

    - chạy trên mỗi node
    - proxy các giao thức UDP, TCP và SCTP
    - không hiểu HTTP
    - cung cấp cân bằng tải
    - chỉ được dùng để truy cập các service

4.  Một Proxy/Load-balancer đứng trước (các) apiserver:

    - sự tồn tại và cách hiện thực khác nhau tùy từng cluster (ví dụ nginx)
    - nằm giữa tất cả các client và một hoặc nhiều apiserver
    - đóng vai trò bộ cân bằng tải (load balancer) nếu có nhiều apiserver.

5.  Cloud Load Balancer trên các service bên ngoài:

    - được cung cấp bởi một số nhà cung cấp đám mây (ví dụ AWS ELB, Google Cloud Load Balancer)
    - được tạo tự động khi Kubernetes service có type là `LoadBalancer`
    - thường chỉ hỗ trợ UDP/TCP
    - việc hỗ trợ SCTP tùy thuộc vào cách hiện thực bộ cân bằng tải của nhà cung cấp đám mây
    - cách hiện thực khác nhau tùy theo nhà cung cấp đám mây.

Người dùng Kubernetes thường sẽ không cần quan tâm đến bất cứ điều gì ngoài hai loại đầu tiên. Quản trị viên cluster
thường sẽ đảm bảo rằng các loại còn lại được thiết lập đúng.

## Yêu cầu chuyển hướng (Requesting redirects)

Proxy đã thay thế khả năng chuyển hướng (redirect). Chuyển hướng đã bị loại bỏ (deprecated).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Trên `k8s-master` bạn chạy `kubectl proxy` rồi `curl` vào địa chỉ localhost của nó và lấy
   được dữ liệu từ API mà không phải gửi token nào. Vì sao? Đoạn từ proxy tới apiserver có được
   mã hóa không?
2. kube-proxy có định tuyến được request theo path HTTP — ví dụ đưa `/api` sang một Service khác
   — không?
3. Bạn cần chạm tới một Pod chưa có Service nào trỏ vào, từ một máy ngoài cluster. Loại proxy
   nào trong bài làm được việc đó, và nó chạy ở đâu?
4. Trong năm loại, loại nào chạy trên **mỗi** node, và loại nào chạy bên trong tiến trình
   apiserver?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì `kubectl proxy` **tự thêm các header xác thực** vào request thay bạn. Về mã hóa thì hai
   chặng khác nhau: **client → proxy dùng HTTP**, còn **proxy → apiserver dùng HTTPS**. Hệ quả
   thực tế của hai tính chất đó: bất kỳ ai chạm được tới địa chỉ mà proxy đang lắng nghe đều
   phát request đi với thông tin xác thực của bạn, nên đừng mở nó ra ngoài localhost.
2. **Không.** Bài ghi thẳng ba đặc điểm của kube proxy: nó **proxy các giao thức UDP, TCP và
   SCTP**, **không hiểu HTTP**, và **chỉ được dùng để truy cập các service**. Định tuyến theo
   path là việc của Ingress hoặc Gateway, không phải của kube-proxy — dù cả ba đều bị gọi chung
   là "proxy".
3. **apiserver proxy.** Nó là một **bastion được tích hợp sẵn trong apiserver**, kết nối người
   dùng ở bên ngoài cluster đến các cluster IP mà nếu không có nó thì có thể không truy cập
   được, và **có thể được dùng để truy cập một Node, Pod hoặc Service**. Nó **chạy trong các
   tiến trình của apiserver**.
4. **kube-proxy chạy trên mỗi node.** **apiserver proxy chạy trong các tiến trình của
   apiserver.** Ba loại còn lại nằm ngoài: `kubectl proxy` chạy trên máy người dùng hoặc trong
   một pod, proxy/load balancer đứng trước apiserver và cloud load balancer đều là thành phần
   bên ngoài cluster.

</details>

Đây là bài cuối của giai đoạn 5. Trả lời trôi chảy cả bốn câu nghĩa là bạn đã đủ nền cho **Lab
5b — NetworkPolicy, Ingress và CNI** (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)),
nơi cluster đổi CNI, cài ingress controller và trả [nợ lab](labs/README.md#5-sổ-nợ-lab) của bài
[84](84-network-policies-vi.md). Câu nào còn vướng thì quay lại đúng mục tương ứng trước khi vào
lab.

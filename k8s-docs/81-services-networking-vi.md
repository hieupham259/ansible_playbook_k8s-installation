# Service, cân bằng tải và mạng (Services, Load Balancing, and Networking)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/>
>
> Các khái niệm và tài nguyên đằng sau hoạt động mạng trong Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), bài 1/16 · Kiểm chứng ở
[Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md).

Bài này là **trang mục lục của cả giai đoạn**, không phải bài kỹ thuật. Nó nêu tên gần như mọi
khái niệm mạng rồi trỏ sang bài khác. Đừng cố hiểu hết ở đây; thứ duy nhất phải nắm chắc là mô
hình mạng và ranh giới trách nhiệm giữa Kubernetes và phần mềm bên ngoài.

**Phải hiểu ở lần đọc này:**

- Mỗi Pod có **một địa chỉ IP duy nhất trên toàn cluster** và một network namespace riêng dùng
  chung cho mọi container trong Pod đó; các container cùng Pod nói với nhau qua `localhost`.
- Mạng Pod bảo đảm **mọi Pod nói chuyện trực tiếp với mọi Pod khác**, cùng node hay khác node,
  **không qua proxy và không qua NAT**; agent trên node (system daemon, kubelet) cũng tới được
  mọi Pod trên node đó.
- Khác biệt so với hệ container cũ: không cần tạo link giữa container, cũng không cần ánh xạ
  port container sang port host — Pod được đối xử gần như VM hay host vật lý.
- Ranh giới trách nhiệm: Kubernetes chỉ hiện thực một vài phần, còn lại chỉ định nghĩa API —
  network namespace do phần mềm hiện thực [CRI](44-cri-vi.md) dựng, mạng Pod do CNI plugin
  hiện thực, service proxy mặc định là kube-proxy nhưng có thể bị thay bởi proxy riêng của CNI.
- Hệ quả trực tiếp của ranh giới đó: NetworkPolicy do chính hiện thực mạng Pod làm, nên trên
  một CNI không hỗ trợ thì **API vẫn tồn tại nhưng không có tác dụng**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Service API và EndpointSlice | mới là tên gọi, cơ chế nằm ở hai bài kế | bài [82](82-service-vi.md), [83](83-endpoint-slices-vi.md) |
| Cú pháp và ngữ nghĩa của NetworkPolicy | chưa cần cách viết, chỉ cần biết nó phụ thuộc CNI | bài [84](84-network-policies-vi.md) |
| Gateway API và `type: LoadBalancer` | cần Service trước, và cần hạ tầng ngoài cluster | bài [11](11-ingress-vi.md), [13](13-gateway-vi.md) |
| Ghi chú Pod dùng host-network trên Windows | lab không có node Windows | giai đoạn 15 |

---

## Mô hình mạng của Kubernetes (The Kubernetes network model) {#the-kubernetes-network-model}

Mô hình mạng của Kubernetes được xây dựng từ nhiều thành phần:

* Mỗi [Pod](./46-pods-vi.md) trong một cluster nhận được một địa chỉ IP riêng,
  duy nhất trên toàn cluster.

  * Một Pod có network namespace riêng tư của nó, được chia sẻ bởi
    tất cả các container bên trong Pod đó. Các tiến trình chạy trong
    những container khác nhau thuộc cùng một Pod có thể giao tiếp với
    nhau qua `localhost`.

* _Mạng Pod_ (pod network, còn gọi là mạng cluster) xử lý việc giao tiếp
  giữa các Pod. Nó đảm bảo rằng (trừ khi có sự phân đoạn mạng có chủ đích):

  * Tất cả các Pod có thể giao tiếp với mọi Pod khác, dù chúng nằm
    trên cùng một [node](./23-nodes-vi.md) hay trên các node khác nhau.
    Các Pod có thể giao tiếp trực tiếp với nhau,
    mà không cần dùng proxy hay chuyển đổi địa chỉ mạng (NAT).

    Trên Windows, quy tắc này không áp dụng cho các Pod dùng host-network.

  * Các agent trên một node (chẳng hạn các system daemon, hoặc kubelet)
    có thể giao tiếp với tất cả các Pod trên node đó.

* [Service](82-service-vi.md) API
  cho phép bạn cung cấp một địa chỉ IP hoặc hostname ổn định (tồn tại lâu dài) cho một dịch vụ
  được hiện thực bởi một hoặc nhiều Pod backend, trong đó từng Pod
  cấu thành dịch vụ có thể thay đổi theo thời gian.

  * Kubernetes tự động quản lý các đối tượng
    [EndpointSlice](./83-endpoint-slices-vi.md)
    để cung cấp thông tin về các Pod hiện đang đứng sau một Service.

  * Một hiện thực service proxy giám sát tập các đối tượng Service và
    EndpointSlice, và lập trình data plane để định tuyến
    lưu lượng của Service tới các backend của nó, bằng cách dùng các API của
    hệ điều hành hoặc của nhà cung cấp đám mây (cloud provider) để chặn hoặc ghi lại các gói tin.

* [Gateway](./13-gateway-vi.md) API
  (hoặc tiền thân của nó là [Ingress](./11-ingress-vi.md))
  cho phép bạn làm cho các Service có thể truy cập được bởi các client nằm ngoài cluster.

  * Một cơ chế đơn giản hơn nhưng ít khả năng cấu hình hơn cho việc
    đi vào cluster (cluster ingress) có sẵn qua
    [`type: LoadBalancer`](82-service-vi.md#loadbalancer)
    của Service API, khi dùng một nhà cung cấp đám mây (cloud provider) được hỗ trợ.

* [NetworkPolicy](84-network-policies-vi.md) là một
  API tích hợp sẵn của Kubernetes cho phép bạn kiểm soát lưu lượng giữa các Pod, hoặc giữa các Pod và
  thế giới bên ngoài.

Trong các hệ thống container cũ, không có kết nối tự động
giữa các container trên những host khác nhau, vì vậy thường phải
tạo liên kết (link) giữa các container một cách tường minh, hoặc ánh xạ port của container
sang port của host để các container trên host khác có thể
truy cập được chúng. Điều này không cần thiết trong Kubernetes; mô hình của Kubernetes là
các Pod có thể được đối xử gần giống như các VM hay host vật lý xét theo
các khía cạnh cấp phát port, đặt tên, khám phá dịch vụ (service discovery), cân bằng tải
(load balancing), cấu hình ứng dụng, và di trú (migration).

Chỉ một vài phần của mô hình này được chính Kubernetes hiện thực.
Với các phần còn lại, Kubernetes định nghĩa các API, còn
chức năng tương ứng do các thành phần bên ngoài cung cấp, một số
trong đó là tùy chọn:

* Việc thiết lập network namespace cho Pod được xử lý bởi phần mềm cấp hệ thống hiện thực
  [Container Runtime Interface](./44-cri-vi.md).

* Bản thân mạng Pod được quản lý bởi một
  [hiện thực mạng Pod](165-addons-vi.md#networking-and-network-policy).
  Trên Linux, hầu hết các container runtime dùng
  Container Networking Interface (CNI)
  để tương tác với hiện thực mạng Pod, vì vậy các hiện thực
  này thường được gọi là các _CNI plugin_.

* Kubernetes cung cấp một hiện thực mặc định cho service proxy,
  gọi là kube-proxy, nhưng một số hiện thực mạng Pod
  lại dùng service proxy riêng của chúng, được tích hợp
  chặt chẽ hơn với phần còn lại của hiện thực đó.

* NetworkPolicy nói chung cũng được hiện thực bởi hiện thực
  mạng Pod. (Một số hiện thực mạng Pod đơn giản không
  hiện thực NetworkPolicy, hoặc quản trị viên có thể chọn
  cấu hình mạng Pod không có hỗ trợ NetworkPolicy. Trong những
  trường hợp này, API vẫn sẽ tồn tại, nhưng sẽ không có tác dụng.)

* Có nhiều [hiện thực của Gateway API](https://gateway-api.sigs.k8s.io/implementations/),
  một số dành riêng cho các môi trường đám mây cụ thể, một số
  tập trung hơn vào các môi trường "bare metal", và số khác mang tính tổng quát hơn.

## Tiếp theo (What's next)

Hướng dẫn thực hành [Kết nối ứng dụng với Service (Connecting Applications with Services)](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)
giúp bạn tìm hiểu về Service và mạng Kubernetes qua một ví dụ thực hành.

[Mạng cluster (Cluster Networking)](157-networking-vi.md) giải thích cách thiết lập
mạng cho cluster của bạn, đồng thời cung cấp cái nhìn tổng quan về các công nghệ liên quan.

Để tìm hiểu về các khái niệm mạng cụ thể, xem:

* [Service](82-service-vi.md) - phơi bày (expose) một ứng dụng phía sau một endpoint duy nhất hướng ra bên ngoài
* [Ingress](./11-ingress-vi.md) - định tuyến HTTP/HTTPS có nhận biết giao thức, dựa trên URI, hostname và path
* [Gateway API](./13-gateway-vi.md) - cấp phát (provision) hạ tầng động và định tuyến lưu lượng nâng cao
* [Network Policies](84-network-policies-vi.md) - kiểm soát luồng lưu lượng ở mức địa chỉ IP hoặc port (tầng OSI 3 hoặc 4)
* [DNS cho Service và Pod](./10-dns-pod-service-vi.md) - khám phá các dịch vụ bên trong cluster của bạn bằng DNS

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Trong cluster lab, Pod CIDR là `10.244.0.0/16`. Một Pod trên `lab-k8s-worker1` `curl` thẳng địa
   chỉ IP của một Pod trên `lab-k8s-worker2`. Theo mô hình mạng, gói tin có bị NAT không, và thành
   phần nào chịu trách nhiệm làm cho đường đi đó chạy được?
2. Hai container trong cùng một Pod gọi nhau bằng cách nào, và vì sao chúng không cần Service?
3. Bạn `kubectl apply` một NetworkPolicy và API server trả lời `created`. Điều đó có nghĩa là
   traffic đã bị chặn chưa?
4. Trong danh sách các thành phần của mô hình mạng, phần nào do chính Kubernetes hiện thực, và
   phần nào Kubernetes chỉ định nghĩa API rồi để bên ngoài làm?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không NAT, và không qua proxy.** Mô hình mạng yêu cầu tất cả các Pod giao tiếp được với
   mọi Pod khác dù cùng node hay khác node, trực tiếp, không dùng proxy hay chuyển đổi địa chỉ
   mạng. Thứ làm cho điều đó thành hiện thực **không phải Kubernetes** mà là **hiện thực mạng
   Pod** — trên cluster lab là CNI plugin đang cài (xem
   [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa)).
2. Qua **`localhost`**. Pod có network namespace riêng và **mọi container trong Pod dùng chung
   namespace đó**, nên các tiến trình ở những container khác nhau trong cùng Pod thấy nhau như
   trên cùng một máy. Service là để cho **các Pod khác** tìm tới, không phải để nói chuyện bên
   trong một Pod.
3. **Chưa chắc.** NetworkPolicy nói chung được hiện thực bởi hiện thực mạng Pod. Bài ghi rõ:
   một số hiện thực mạng Pod đơn giản không hiện thực NetworkPolicy, hoặc quản trị viên chọn
   cấu hình mạng Pod không có hỗ trợ NetworkPolicy — trong những trường hợp đó **API vẫn tồn
   tại nhưng sẽ không có tác dụng**. Object được lưu vào API server không đồng nghĩa với việc
   có ai đó thực thi nó.
4. Kubernetes hiện thực **Service API, EndpointSlice và kube-proxy** (service proxy mặc định).
   Kubernetes **chỉ định nghĩa API** cho: thiết lập network namespace của Pod (do phần mềm hiện
   thực Container Runtime Interface làm), bản thân mạng Pod (do hiện thực mạng Pod / CNI plugin
   làm), NetworkPolicy (cũng do hiện thực mạng Pod làm), và Gateway API (do các bản hiện thực
   bên ngoài làm).

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

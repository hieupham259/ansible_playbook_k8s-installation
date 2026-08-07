# Các lựa chọn topology cho tính sẵn sàng cao (Options for Highly Available Topology)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/>
>
> Trang này giải thích hai lựa chọn cấu hình topology cho các cluster Kubernetes có tính sẵn sàng cao (high availability - HA).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 8](LO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm), bài 5/9 ·
Kiểm chứng ở Lab 8b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài ngắn nhất giai đoạn 8, và cũng là bài **quyết định**: lộ trình đánh dấu nó là *quyết định
trước khi dựng HA*. Chọn sai topology thì hai bài sau bạn làm nhầm việc — bài
[07](07-setup-ha-etcd-with-kubeadm-vi.md) chỉ cần nếu chọn external etcd, còn bài
[08](08-high-availability-vi.md) rẽ nhánh theo đúng lựa chọn này. Bạn đã thấy sơ đồ tiến trình
của cả hai mô hình ở [22 — Kiến trúc cluster](22-architecture-vi.md#cách-kubeadm-bố-trí-control-plane);
bài này bổ sung phần còn thiếu là **cái giá của từng mô hình**. Đọc để cầm được một bảng đánh
đổi, không phải để nhớ hình vẽ.

**Phải hiểu ở lần đọc này:**

- Điểm chung của cả hai topology: mỗi control plane node đều chạy một instance `kube-apiserver`,
  `kube-scheduler` và `kube-controller-manager`, và `kube-apiserver` được cung cấp cho các
  worker node **thông qua một load balancer**.
- *Topology etcd xếp chồng*: mỗi control plane node tạo một etcd member cục bộ, và member đó
  **chỉ giao tiếp với `kube-apiserver` của chính node đó**. Đây là **mặc định của kubeadm** —
  `kubeadm init` và `kubeadm join --control-plane` tự tạo member.
- Cái giá của stacked: **hỏng theo cặp** — một node sập là mất đồng thời một etcd member *và*
  một instance control plane, tính dự phòng suy giảm. Bù bằng cách chạy **tối thiểu ba** control
  plane node xếp chồng.
- *Topology etcd bên ngoài*: etcd member chạy trên host tách biệt, và **mỗi etcd host giao tiếp
  với `kube-apiserver` của từng control plane node** — quan hệ nhiều-nhiều, khác hẳn stacked.
  Nhờ tách rời, mất một control plane instance hoặc một etcd member ít ảnh hưởng hơn.
- Cái giá của external: **gấp đôi số host** — tối thiểu ba host cho control plane **và** ba host
  cho etcd, chưa tính worker. Đổi lại là quản lý phức tạp hơn stacked, kể cả việc sao chép dữ liệu.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ghi chú "kubeadm khởi tạo etcd cluster theo cách tĩnh" và link *Hướng dẫn Clustering* của etcd | cách etcd hình thành cluster là chủ đề vận hành etcd | CP4 etcd backup |
| Cụm từ "tính dự phòng" ở mức số lượng member sống sót | cần quorum và bầu leader của etcd | CP4 etcd backup |
| Hai hình vẽ topology trên kubernetes.io | chỉ minh họa lại phần chữ | không cần |
| Cách dựng thật từng topology | là nội dung hai bài kế | bài [07](07-setup-ha-etcd-with-kubeadm-vi.md) và [08](08-high-availability-vi.md) |

---

Trang này giải thích hai lựa chọn cấu hình topology cho các cluster Kubernetes có tính sẵn sàng cao (high availability - HA) của bạn.

Bạn có thể thiết lập một cluster HA:

- Với các control plane node xếp chồng (stacked), trong đó các etcd node nằm cùng vị trí (colocated) với các control plane node
- Với các etcd node bên ngoài (external), trong đó etcd chạy trên các node tách biệt khỏi control plane

Bạn nên cân nhắc kỹ lưỡng các ưu điểm và nhược điểm của từng topology trước khi thiết lập một cluster HA.

> **Ghi chú:**
> kubeadm khởi tạo (bootstrap) etcd cluster theo cách tĩnh (static). Hãy đọc
> [Hướng dẫn Clustering](https://github.com/etcd-io/etcd/blob/release-3.4/Documentation/op-guide/clustering.md#static)
> của etcd để biết thêm chi tiết.

## Topology etcd xếp chồng (Stacked etcd topology)

Một cluster HA xếp chồng là một [topology](https://en.wikipedia.org/wiki/Network_topology) trong đó cluster lưu trữ
dữ liệu phân tán do etcd cung cấp được xếp chồng lên trên cluster hình thành từ các node do kubeadm quản lý
đang chạy các thành phần control plane.

Mỗi control plane node chạy một instance của `kube-apiserver`, `kube-scheduler` và `kube-controller-manager`.
`kube-apiserver` được cung cấp cho các worker node thông qua một bộ cân bằng tải (load balancer).

Mỗi control plane node tạo một etcd member cục bộ và etcd member này chỉ giao tiếp với
`kube-apiserver` của chính node đó. Điều tương tự cũng áp dụng cho các instance `kube-controller-manager`
và `kube-scheduler` cục bộ.

Topology này gắn kết (couple) các control plane và etcd member trên cùng các node. Nó đơn giản hơn khi thiết lập so với một cluster
có các etcd node bên ngoài, và đơn giản hơn khi quản lý việc sao chép dữ liệu (replication).

Tuy nhiên, một cluster xếp chồng có nguy cơ hỏng hóc theo cặp (failed coupling). Nếu một node bị sập, cả một etcd member lẫn một instance
control plane đều bị mất, và tính dự phòng (redundancy) bị suy giảm. Bạn có thể giảm thiểu rủi ro này bằng cách thêm nhiều control plane node hơn.

Do đó, bạn nên chạy tối thiểu ba control plane node xếp chồng cho một cluster HA.

Đây là topology mặc định trong kubeadm. Một etcd member cục bộ được tạo tự động
trên các control plane node khi sử dụng `kubeadm init` và `kubeadm join --control-plane`.

![Topology etcd xếp chồng](https://kubernetes.io/images/kubeadm/kubeadm-ha-topology-stacked-etcd.svg)

## Topology etcd bên ngoài (External etcd topology)

Một cluster HA với etcd bên ngoài là một [topology](https://en.wikipedia.org/wiki/Network_topology)
trong đó cluster lưu trữ dữ liệu phân tán do etcd cung cấp nằm bên ngoài cluster hình thành từ
các node đang chạy các thành phần control plane.

Giống như topology etcd xếp chồng, mỗi control plane node trong topology etcd bên ngoài chạy
một instance của `kube-apiserver`, `kube-scheduler` và `kube-controller-manager`.
Và `kube-apiserver` được cung cấp cho các worker node thông qua một bộ cân bằng tải. Tuy nhiên,
các etcd member chạy trên các host tách biệt, và mỗi etcd host giao tiếp với
`kube-apiserver` của từng control plane node.

Topology này tách rời (decouple) control plane và etcd member. Do đó, nó mang lại một thiết lập HA trong đó
việc mất một instance control plane hoặc một etcd member có ít tác động hơn và không ảnh hưởng
đến tính dự phòng của cluster nhiều như topology HA xếp chồng.

Tuy nhiên, topology này yêu cầu số lượng host gấp đôi so với topology HA xếp chồng.
Cần tối thiểu ba host cho các control plane node và ba host cho các etcd node
đối với một cluster HA theo topology này.

![Topology etcd bên ngoài](https://kubernetes.io/images/kubeadm/kubeadm-ha-topology-external-etcd.svg)

## Tiếp theo (What's next)

- [Thiết lập một cluster có tính sẵn sàng cao với kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 8:

1. Cluster lab của bạn — `k8s-master` 192.168.100.111 cộng `k8s-worker1` và `k8s-worker2` —
   đang nằm ở topology nào? Vì sao nó vẫn không phải là một cluster HA?
2. Trong topology stacked, etcd member trên node cp1 có phục vụ `kube-apiserver` của cp2 không?
   Trong topology external thì quan hệ đó thế nào?
3. Một node control plane sập. Vì sao mất mát trong cluster stacked nặng hơn trong cluster
   external, dù cả hai đều mất đúng một máy?
4. Bạn được cấp đúng sáu VM. Dựng được một cluster HA theo topology external etcd không?
5. Trong cả hai topology, cái gì đứng trước các `kube-apiserver`, và ai là bên được phục vụ
   qua nó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Stacked** — bài nói đây là **topology mặc định trong kubeadm**, và một etcd member cục bộ
   được tạo tự động trên control plane node khi dùng `kubeadm init`. Nhưng nó **không HA** vì
   chỉ có **một** control plane node: bài yêu cầu **tối thiểu ba control plane node xếp chồng
   cho một cluster HA**. Với một node, mất node đó là mất cả etcd member lẫn toàn bộ control plane.
2. **Không.** Trong stacked, mỗi control plane node tạo một etcd member cục bộ và member này
   **chỉ giao tiếp với `kube-apiserver` của chính node đó** — điều tương tự áp dụng cho
   `kube-controller-manager` và `kube-scheduler` cục bộ. Trong external thì ngược lại: **mỗi
   etcd host giao tiếp với `kube-apiserver` của từng control plane node**. Đây là chỗ dễ nhầm
   nhất của bài, vì "ba etcd hợp thành một cluster" khiến ta tưởng ai cũng nói chuyện với ai;
   quan hệ chung một etcd cluster và quan hệ apiserver–member là hai chuyện khác nhau.
3. Vì stacked **gắn kết control plane và etcd member trên cùng các node**, nên nó có nguy cơ
   **hỏng hóc theo cặp**: một node sập là **mất đồng thời một etcd member và một instance
   control plane**, tính dự phòng suy giảm ở cả hai tầng cùng lúc. External **tách rời** hai
   thứ đó, nên mất một instance control plane hoặc một etcd member **có ít tác động hơn và
   không ảnh hưởng đến tính dự phòng nhiều như topology xếp chồng**.
4. **Vừa đủ phần control plane và etcd, nhưng chưa có worker.** Bài yêu cầu tối thiểu **ba host
   cho các control plane node và ba host cho các etcd node** — sáu VM đã dùng hết cho hai nhóm
   đó. Đây chính là điều bài gọi là "yêu cầu số lượng host gấp đôi so với topology HA xếp chồng",
   và cũng là lý do [bản đồ lab](labs/README.md#4-bản-đồ-lab) cho hai lab HA dùng **bộ VM riêng**
   thay vì chuỗi snapshot chính.
5. Một **bộ cân bằng tải (load balancer)**. Bài nói ở cả hai mục: `kube-apiserver` **được cung
   cấp cho các worker node thông qua một load balancer** — câu này xuất hiện y hệt trong phần
   stacked và phần external, tức là load balancer **không** phải đặc thù của topology nào mà là
   thành phần bắt buộc của mọi cluster HA. Đó là lý do bài [08](08-high-availability-vi.md) đặt
   việc dựng load balancer vào mục *Các bước đầu tiên cho cả hai phương pháp*.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

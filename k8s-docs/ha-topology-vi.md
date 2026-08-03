# Các lựa chọn topology cho tính sẵn sàng cao (Options for Highly Available Topology)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/>
>
> Trang này giải thích hai lựa chọn cấu hình topology cho các cluster Kubernetes có tính sẵn sàng cao (high availability - HA).

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

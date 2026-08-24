# Cài đặt các Add-on (Installing Addons)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/cluster-administration/addons/

> **Ghi chú:**
>
> Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án đó, vốn được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi. [Thông tin thêm.](https://kubernetes.io/docs/concepts/cluster-administration/addons/#third-party-content-disclaimer)

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 12](00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao), bài 5/8 ·
Kiểm chứng ở Lab 12 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là **danh mục**, không phải bài giảng. Không ai học thuộc nó. Đọc để nhớ **các nhóm chức
năng** và một vài cái tên bạn chắc chắn sẽ gặp lại — phần còn lại là bảng tra cho ngày bạn phải
chọn công cụ. Nhóm mạng chiếm phần lớn độ dài; lướt qua tên rồi đi tiếp.

**Phải hiểu ở lần đọc này:**

- Add-on **mở rộng** chức năng của Kubernetes, tức chúng nằm ngoài phần core mà bạn cài bằng
  kubeadm. Và trang này **không nhằm liệt kê đầy đủ tất cả** — công cụ không có mặt ở đây không
  có nghĩa là nó không tồn tại hay không dùng được.
- Năm nhóm chức năng của danh mục: **Mạng và Network Policy**, **Khám phá dịch vụ**, **Trực quan
  hóa & Điều khiển**, **Hạ tầng**, **Đo lường**.
- Trong nhóm mạng, phân biệt add-on **chỉ cung cấp mạng** với add-on **cung cấp cả network
  policy**: Flannel được mô tả đúng là "nhà cung cấp mạng overlay", còn Calico, Canal, Cilium,
  Weave Net, Antrea, OVN-Kubernetes… có kèm network policy.
- Nhóm **Đo lường** chỉ có đúng một mục — [kube-state-metrics](163-kube-state-metrics-vi.md) —
  khớp lại với bài bạn vừa đọc ở giai đoạn 11.
- Ràng buộc chung: toàn bộ danh sách là **dự án bên thứ ba**, dự án Kubernetes không chịu trách
  nhiệm; và các add-on cũ trong `cluster/addons` đã **ngừng sử dụng**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mô tả chi tiết từng nhà cung cấp mạng (ACI, Contiv, Contrail, NSX-T, Nuage, Terway…) | là bảng tra khi phải chọn CNI cho một môi trường cụ thể | CP6 DNS, CNI và kube-proxy |
| Multus, Knitter, Spiderpool, Nodus — nhiều interface, SR-IOV, RDMA, SFC | mạng chuyên biệt cho NFV và HPC, ngoài phạm vi admin phổ thông | không cần |
| Dashboard và Headlamp | giao diện web, không cần thiết để vận hành bằng `kubectl` | không cần |
| KubeVirt trong nhóm *Hạ tầng* | chạy máy ảo trên Kubernetes là một hướng riêng | không cần |
| *Các Add-on cũ* trong `cluster/addons` | đã deprecated | không cần |

---

Add-on mở rộng chức năng của Kubernetes.

Trang này liệt kê một số add-on hiện có kèm liên kết đến hướng dẫn cài đặt
tương ứng của từng add-on. Danh sách này không nhằm liệt kê đầy đủ tất cả.

## Mạng và Network Policy (Networking and Network Policy) {#networking-and-network-policy}

* [ACI](https://www.github.com/noironetworks/aci-containers) cung cấp mạng container
  tích hợp và bảo mật mạng với Cisco ACI.
* [Antrea](https://antrea.io/) hoạt động ở Layer 3/4 để cung cấp các dịch vụ mạng và
  bảo mật cho Kubernetes, tận dụng Open vSwitch làm data plane cho mạng.
  Antrea là một [dự án CNCF ở cấp Sandbox](https://www.cncf.io/projects/antrea/).
* [Calico](https://www.tigera.io/project-calico/) là một nhà cung cấp giải pháp mạng và
  network policy. Calico hỗ trợ một tập hợp linh hoạt các tùy chọn mạng để bạn có thể
  chọn phương án hiệu quả nhất cho tình huống của mình, bao gồm mạng non-overlay
  và overlay, có hoặc không dùng BGP. Calico sử dụng cùng một engine để
  thực thi network policy cho các host, pod, và (nếu dùng Istio & Envoy)
  các ứng dụng ở tầng service mesh.
* [Canal](https://projectcalico.docs.tigera.io/getting-started/kubernetes/flannel/flannel)
  kết hợp Flannel và Calico, cung cấp mạng và network policy.
* [Cilium](https://github.com/cilium/cilium) là một giải pháp mạng, khả năng quan sát
  (observability) và bảo mật với data plane dựa trên eBPF. Cilium cung cấp một
  mạng Layer 3 phẳng đơn giản với khả năng trải rộng trên nhiều cluster
  ở chế độ định tuyến native hoặc chế độ overlay/đóng gói (encapsulation), và có thể
  thực thi network policy ở tầng L3-L7 bằng mô hình bảo mật dựa trên danh tính
  (identity-based) tách biệt khỏi việc đánh địa chỉ mạng. Cilium có thể đóng vai trò
  thay thế cho kube-proxy; nó cũng cung cấp thêm các tính năng quan sát và bảo mật tùy chọn.
  Cilium là một [dự án CNCF ở cấp Graduated](https://www.cncf.io/projects/cilium/).
* [CNI-Genie](https://github.com/cni-genie/CNI-Genie) cho phép Kubernetes kết nối
  liền mạch với một CNI plugin tùy chọn, chẳng hạn Calico, Canal, Flannel hoặc Weave.
  CNI-Genie là một [dự án CNCF ở cấp Sandbox](https://www.cncf.io/projects/cni-genie/).
* [Contiv](https://contivpp.io/) cung cấp mạng có thể cấu hình (L3 native dùng BGP,
  overlay dùng vxlan, L2 cổ điển, và Cisco-SDN/ACI) cho nhiều trường hợp sử dụng khác nhau
  cùng một framework chính sách phong phú. Dự án Contiv hoàn toàn
  [mã nguồn mở](https://github.com/contiv). [Trình cài đặt](https://github.com/contiv/install)
  cung cấp cả hai tùy chọn cài đặt dựa trên kubeadm và không dựa trên kubeadm.
* [Contrail](https://www.juniper.net/us/en/products-services/sdn/contrail/contrail-networking/),
  dựa trên [Tungsten Fabric](https://tungsten.io), là một nền tảng ảo hóa mạng
  đa đám mây (multi-cloud) và quản lý chính sách, mã nguồn mở. Contrail và Tungsten
  Fabric được tích hợp với các hệ thống điều phối (orchestration) như Kubernetes, OpenShift,
  OpenStack và Mesos, đồng thời cung cấp các chế độ cô lập cho máy ảo, container/pod
  và các workload chạy trên bare metal.
* [Flannel](https://github.com/flannel-io/flannel#deploying-flannel-manually) là
  một nhà cung cấp mạng overlay có thể dùng với Kubernetes.
* [Gateway API](./13-gateway-vi.md) là một dự án mã nguồn mở được cộng đồng
  [SIG Network](https://github.com/kubernetes/community/tree/main/sig-network) quản lý,
  cung cấp một API giàu khả năng biểu đạt, dễ mở rộng và hướng theo vai trò (role-oriented) để mô hình hóa mạng cho dịch vụ.
* [Knitter](https://github.com/ZTE/Knitter/) là một plugin hỗ trợ nhiều network
  interface trong một Pod Kubernetes.
* [kube-router](https://github.com/cloudnativelabs/kube-router) là một giải pháp
  trọn gói (turnkey) mã nguồn mở cho mạng Kubernetes với mục tiêu mang lại
  sự đơn giản trong vận hành và hiệu năng cao. Nó tận dụng Kubernetes API,
  BGP và Golang cho control path, và các thành phần mạng cơ bản của Linux (IPVS,
  nftables, v.v.) cho data path. Nó cung cấp một lựa chọn thay thế với chi phí thấp và
  được dùng trong cả k0s lẫn k3s.
* [Multus](https://github.com/k8snetworkplumbingwg/multus-cni) là một Multi plugin hỗ trợ
  nhiều mạng trong Kubernetes, hỗ trợ tất cả các CNI plugin
  (ví dụ Calico, Cilium, Contiv, Flannel), bên cạnh các workload dựa trên SRIOV, DPDK, OVS-DPDK và
  VPP trong Kubernetes.
* [OVN-Kubernetes](https://github.com/ovn-org/ovn-kubernetes/) là một nhà cung cấp mạng
  cho Kubernetes dựa trên [OVN (Open Virtual Network)](https://github.com/ovn-org/ovn/),
  một hiện thực mạng ảo ra đời từ dự án Open vSwitch (OVS).
  OVN-Kubernetes cung cấp một hiện thực mạng dựa trên overlay cho Kubernetes,
  bao gồm một hiện thực cân bằng tải và network policy dựa trên OVS.
* [Nodus](https://github.com/akraino-edge-stack/icn-nodus) là một CNI
  controller plugin dựa trên OVN để cung cấp Service function chaining (SFC) theo hướng cloud native.
* [NSX-T](https://docs.vmware.com/en/VMware-NSX-T-Data-Center/index.html) Container Plug-in (NCP)
  cung cấp tích hợp giữa VMware NSX-T và các trình điều phối container như
  Kubernetes, cũng như tích hợp giữa NSX-T và các nền tảng CaaS/PaaS
  dựa trên container như Pivotal Container Service (PKS) và OpenShift.
* [Nuage](https://github.com/nuagenetworks/nuage-kubernetes/blob/v5.1.1-1/docs/kubernetes-1-installation.rst)
  là một nền tảng SDN cung cấp mạng dựa trên chính sách giữa các Pod Kubernetes
  và các môi trường ngoài Kubernetes, kèm khả năng quan sát và giám sát bảo mật.
* [Romana](https://github.com/romana) là một giải pháp mạng Layer 3 cho các mạng pod,
  đồng thời hỗ trợ API [NetworkPolicy](./84-network-policies-vi.md).
* [Spiderpool](https://github.com/spidernet-io/spiderpool) là một giải pháp mạng underlay và RDMA
  cho Kubernetes. Spiderpool được hỗ trợ trên các môi trường bare metal, máy ảo
  và đám mây công cộng (public cloud).
* [Terway](https://github.com/AliyunContainerService/terway/) là một bộ CNI plugin
  dựa trên các sản phẩm mạng VPC và ECS của AlibabaCloud. Nó cung cấp mạng VPC native
  và network policy trong các môi trường AlibabaCloud.
* [Weave Net](https://github.com/rajch/weave#using-weave-on-kubernetes)
  cung cấp mạng và network policy, tiếp tục hoạt động ở cả hai phía
  khi mạng bị phân mảnh (network partition), và không yêu cầu cơ sở dữ liệu bên ngoài.

## Khám phá dịch vụ (Service Discovery)

* [CoreDNS](https://coredns.io) là một DNS server linh hoạt, dễ mở rộng, có thể được
  [cài đặt](https://github.com/coredns/helm)
  làm DNS trong cluster cho các pod.

## Trực quan hóa &amp; Điều khiển (Visualization &amp; Control)

* [Dashboard](https://github.com/kubernetes/dashboard#kubernetes-dashboard)
  là một giao diện web dạng dashboard cho Kubernetes.
* [Headlamp](https://headlamp.dev/) là một giao diện người dùng (UI) Kubernetes có thể mở rộng,
  có thể được triển khai trong cluster hoặc dùng như một ứng dụng desktop.

## Hạ tầng (Infrastructure)

* [KubeVirt](https://kubevirt.io/user-guide/#/installation/installation) là một add-on
  để chạy máy ảo trên Kubernetes. Thường chạy trên các cluster bare-metal.
* [Node problem detector](https://github.com/kubernetes/node-problem-detector)
  chạy trên các node Linux và báo cáo các sự cố hệ thống dưới dạng
  [Event](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1/) hoặc
  [Node condition](https://kubernetes.io/docs/concepts/architecture/nodes#condition).

## Đo lường (Instrumentation)

* [kube-state-metrics](./163-kube-state-metrics-vi.md)

## Các Add-on cũ (Legacy Add-ons)

Có một số add-on khác được ghi lại trong thư mục
[cluster/addons](https://git.k8s.io/kubernetes/cluster/addons) đã ngừng sử dụng (deprecated).

Những add-on nào còn được bảo trì tốt nên được liên kết vào trang này. Chào đón các PR!

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 12:

1. Cluster lab đang chạy Flannel từ Lab 00. Theo danh mục này, Flannel thuộc nhóm nào, và mô tả
   của nó thiếu khả năng gì so với mô tả của Calico hay Cilium?
2. **Câu bẫy.** Bạn cần một add-on cho việc X, tra trang này không thấy tên nào phù hợp. Kết luận
   đúng là gì? Trang này tự nói gì về độ đầy đủ của nó?
3. Nhóm *Đo lường* chỉ có kube-state-metrics. Nó cung cấp loại tín hiệu nào, và vì sao nó nằm ở
   danh mục add-on chứ không phải thành phần core?
4. Node problem detector nằm ở nhóm *Hạ tầng*. Nó chạy ở đâu và báo cáo sự cố dưới dạng gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Flannel thuộc nhóm **Mạng và Network Policy**, nhưng mô tả của nó chỉ là "**một nhà cung cấp
   mạng overlay** có thể dùng với Kubernetes" — **không nhắc network policy**. Trong khi đó Calico
   được mô tả là "nhà cung cấp giải pháp mạng **và network policy**", Cilium "có thể thực thi
   network policy ở tầng L3-L7", Canal "kết hợp Flannel và Calico, cung cấp mạng và network
   policy". Đó chính là lý do lộ trình phải đổi CNI ở Lab 5b để NetworkPolicy được thực thi thật —
   xem [sổ nợ lab](labs/README.md#5-sổ-nợ-lab).
2. Kết luận đúng là **danh sách này không đầy đủ**, không phải "add-on đó không tồn tại". Trang tự
   ghi: "Danh sách này **không nhằm liệt kê đầy đủ tất cả**", và kết thúc bằng lời mời gửi PR để
   bổ sung những add-on còn được bảo trì tốt. Trực giác sai ở chỗ coi tài liệu chính thức là danh
   mục duyệt trước; nó chỉ là tập hợp những gì cộng đồng đã đóng góp vào.
3. Nó cung cấp **metric về trạng thái của các đối tượng Kubernetes**, sinh ra bằng cách kết nối
   tới API server — đúng như bài [163](163-kube-state-metrics-vi.md) mô tả. Nó nằm ở danh mục
   add-on vì đó chính là định nghĩa của add-on trong trang này: thứ **mở rộng** chức năng
   Kubernetes, phải cài thêm, và là dự án bên thứ ba mà dự án Kubernetes không chịu trách nhiệm.
4. Nó **chạy trên các node Linux** và báo cáo các sự cố hệ thống dưới dạng **Event** hoặc **Node
   condition** — tức là đưa thông tin sức khỏe của máy vào đúng những kênh mà Kubernetes API đã có
   sẵn, thay vì tạo ra một kênh riêng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

# Cài đặt các Add-on (Installing Addons)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/cluster-administration/addons/

> **Ghi chú:**
>
> Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án đó, vốn được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi. [Thông tin thêm.](https://kubernetes.io/docs/concepts/cluster-administration/addons/#third-party-content-disclaimer)

Add-on mở rộng chức năng của Kubernetes.

Trang này liệt kê một số add-on hiện có kèm liên kết đến hướng dẫn cài đặt
tương ứng của từng add-on. Danh sách này không nhằm liệt kê đầy đủ tất cả.

## Mạng và Network Policy (Networking and Network Policy)

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
  [Node condition](https://kubernetes.io/docs/concepts/architecture/nodes/#condition).

## Đo lường (Instrumentation)

* [kube-state-metrics](./163-kube-state-metrics-vi.md)

## Các Add-on cũ (Legacy Add-ons)

Có một số add-on khác được ghi lại trong thư mục
[cluster/addons](https://git.k8s.io/kubernetes/cluster/addons) đã ngừng sử dụng (deprecated).

Những add-on nào còn được bảo trì tốt nên được liên kết vào trang này. Chào đón các PR!

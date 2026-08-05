# Mạng trên Windows (Networking on Windows)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/windows-networking/>

Kubernetes hỗ trợ chạy node trên Linux hoặc Windows. Bạn có thể trộn cả hai loại node
trong cùng một cluster.
Trang này cung cấp cái nhìn tổng quan về mạng dành riêng cho hệ điều hành Windows.

## Mạng container trên Windows (Container networking on Windows) {#networking}

Mạng cho các container Windows được cung cấp thông qua các
[CNI plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/).
Về mặt mạng, các container Windows hoạt động tương tự như máy ảo. Mỗi container có một
bộ điều hợp mạng ảo (virtual network adapter — vNIC) được kết nối tới một switch ảo
Hyper-V (vSwitch). Host Networking Service (HNS) và Host Compute Service (HCS) phối hợp
với nhau để tạo container và gắn các vNIC của container vào mạng. HCS chịu trách nhiệm
quản lý container, trong khi HNS chịu trách nhiệm quản lý các tài nguyên mạng như:

* Mạng ảo (bao gồm việc tạo các vSwitch)
* Endpoint / vNIC
* Namespace
* Các policy bao gồm đóng gói gói tin (packet encapsulation), quy tắc cân bằng tải
  (load-balancing), ACL và quy tắc NAT.

HNS và vSwitch của Windows hiện thực cơ chế namespace và có thể tạo các NIC ảo khi cần
cho một Pod hoặc container. Tuy nhiên, nhiều cấu hình như DNS, route và metric được lưu
trong cơ sở dữ liệu registry của Windows thay vì dưới dạng file bên trong `/etc` — cách
mà Linux lưu các cấu hình đó. Registry của Windows dành cho container tách biệt với
registry của host, vì vậy những khái niệm như ánh xạ `/etc/resolv.conf` từ host vào
container không có cùng tác dụng như trên Linux. Các cấu hình này phải được thiết lập
bằng các API của Windows chạy trong ngữ cảnh của chính container đó. Do đó, các hiện thực
CNI cần gọi HNS thay vì dựa vào ánh xạ file để truyền thông tin mạng vào Pod hoặc container.

## Các chế độ mạng (Network modes)

Windows hỗ trợ năm driver/chế độ mạng khác nhau: L2bridge, L2tunnel,
Overlay (Beta), Transparent và NAT. Trong một cluster hỗn hợp với các worker node
Windows và Linux, bạn cần chọn một giải pháp mạng tương thích trên cả
Windows lẫn Linux. Bảng sau liệt kê các plugin out-of-tree được hỗ trợ trên Windows,
kèm khuyến nghị về thời điểm nên dùng từng CNI:

| Driver mạng | Mô tả | Thay đổi gói tin của container | Network plugin | Đặc điểm của network plugin |
| -------------- | ----------- | ------------------------------ | --------------- | ------------------------------ |
| L2bridge       | Các container được gắn vào một vSwitch ngoài (external vSwitch). Các container được gắn vào mạng underlay, mặc dù mạng vật lý không cần học địa chỉ MAC của container vì chúng được ghi lại (rewrite) khi vào/ra. | MAC được ghi lại thành MAC của host, IP có thể được ghi lại thành IP của host bằng policy HNS OutboundNAT. | [win-bridge](https://www.cni.dev/plugins/current/main/win-bridge/), [Azure-CNI](https://github.com/Azure/azure-container-networking/blob/master/docs/cni.md), [Flannel host-gateway](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md#host-gw) dùng win-bridge | win-bridge dùng chế độ mạng L2bridge, kết nối các container vào underlay của các host, cho hiệu năng tốt nhất. Yêu cầu các route do người dùng định nghĩa (user-defined routes — UDR) để có kết nối giữa các node. |
| L2Tunnel | Đây là một trường hợp đặc biệt của l2bridge, nhưng chỉ được dùng trên Azure. Tất cả gói tin được gửi tới host ảo hóa, nơi SDN policy được áp dụng. | MAC được ghi lại, IP nhìn thấy được trên mạng underlay | [Azure-CNI](https://github.com/Azure/azure-container-networking/blob/master/docs/cni.md) | Azure-CNI cho phép tích hợp container với Azure vNET, và cho phép chúng tận dụng tập các khả năng mà [Azure Virtual Network cung cấp](https://azure.microsoft.com/en-us/services/virtual-network/). Ví dụ: kết nối an toàn tới các dịch vụ Azure hoặc dùng Azure NSG. Xem [azure-cni để có một số ví dụ](https://docs.microsoft.com/azure/aks/concepts-network#azure-cni-advanced-networking) |
| Overlay | Các container được cấp một vNIC kết nối tới một vSwitch ngoài. Mỗi mạng overlay có subnet IP riêng, được định nghĩa bởi một IP prefix tùy chỉnh. Driver mạng overlay dùng đóng gói VXLAN. | Được đóng gói với một header bên ngoài. | [win-overlay](https://www.cni.dev/plugins/current/main/win-overlay/), [Flannel VXLAN](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md#vxlan) (dùng win-overlay) | Nên dùng win-overlay khi muốn các mạng container ảo được cô lập khỏi underlay của các host (ví dụ vì lý do bảo mật). Cho phép tái sử dụng IP cho các mạng overlay khác nhau (có tag VNID khác nhau) nếu bạn bị hạn chế về IP trong datacenter. Tùy chọn này yêu cầu [KB4489899](https://support.microsoft.com/help/4489899) trên Windows Server 2019. |
| Transparent (trường hợp sử dụng đặc biệt cho [ovn-kubernetes](https://github.com/openvswitch/ovn-kubernetes)) | Yêu cầu một vSwitch ngoài. Các container được gắn vào một vSwitch ngoài, cho phép giao tiếp bên trong Pod (intra-pod) thông qua các mạng logic (logical switch và router). | Gói tin được đóng gói qua tunnel [GENEVE](https://datatracker.ietf.org/doc/draft-gross-geneve/) hoặc [STT](https://datatracker.ietf.org/doc/draft-davie-stt/) để tới các Pod không nằm trên cùng host. <br/> Gói tin được chuyển tiếp hoặc bị loại bỏ dựa trên thông tin metadata của tunnel do ovn network controller cung cấp. <br/> NAT được thực hiện cho giao tiếp bắc-nam (north-south). | [ovn-kubernetes](https://github.com/openvswitch/ovn-kubernetes) | [Triển khai qua ansible](https://github.com/openvswitch/ovn-kubernetes/tree/master/contrib). Các ACL phân tán có thể được áp dụng thông qua các policy của Kubernetes. Hỗ trợ IPAM. Có thể đạt cân bằng tải mà không cần kube-proxy. NAT được thực hiện mà không dùng iptables/netsh. |
| NAT (*không dùng trong Kubernetes*) | Các container được cấp một vNIC kết nối tới một vSwitch nội bộ. DNS/DHCP được cung cấp bởi một thành phần nội bộ tên là [WinNAT](https://techcommunity.microsoft.com/t5/virtualization/windows-nat-winnat-capabilities-and-limitations/ba-p/382303) | MAC và IP được ghi lại thành MAC/IP của host. | [nat](https://github.com/Microsoft/windows-container-networking/tree/master/plugins/nat) | Được liệt kê ở đây cho đầy đủ |

Như đã nêu ở trên, [CNI plugin](https://github.com/flannel-io/cni-plugin)
của [Flannel](https://github.com/coreos/flannel)
cũng được [hỗ trợ](https://github.com/flannel-io/cni-plugin#windows-support-experimental) trên Windows thông qua
[VXLAN network backend](https://github.com/coreos/flannel/blob/master/Documentation/backends.md#vxlan) (**hỗ trợ Beta**; ủy quyền cho win-overlay)
và [host-gateway network backend](https://github.com/coreos/flannel/blob/master/Documentation/backends.md#host-gw) (hỗ trợ stable; ủy quyền cho win-bridge).

Plugin này hỗ trợ ủy quyền (delegate) cho một trong các CNI plugin tham chiếu (win-overlay,
win-bridge), hoạt động phối hợp với Flannel daemon trên Windows (Flanneld) để
tự động gán subnet lease cho node và tạo mạng HNS. Plugin này đọc
file cấu hình riêng của nó (cni.conf), và tổng hợp nó với các biến môi trường
từ file subnet.env do FlannelD sinh ra. Sau đó nó ủy quyền cho một trong
các CNI plugin tham chiếu để thiết lập đường mạng (network plumbing), và gửi cấu hình
chính xác chứa subnet đã gán cho node tới IPAM plugin (ví dụ: `host-local`).

Với các object Node, Pod và Service, các luồng mạng sau được hỗ trợ cho
lưu lượng TCP/UDP:

* Pod → Pod (IP)
* Pod → Pod (Tên)
* Pod → Service (Cluster IP)
* Pod → Service (PQDN, nhưng chỉ khi không chứa dấu ".")
* Pod → Service (FQDN)
* Pod → bên ngoài (IP)
* Pod → bên ngoài (DNS)
* Node → Pod
* Pod → Node

## Quản lý địa chỉ IP (IP address management — IPAM) {#ipam}

Các tùy chọn IPAM sau được hỗ trợ trên Windows:

* [host-local](https://github.com/containernetworking/plugins/tree/master/plugins/ipam/host-local)
* [azure-vnet-ipam](https://github.com/Azure/azure-container-networking/blob/master/docs/ipam.md) (chỉ dành cho azure-cni)
* [Windows Server IPAM](https://docs.microsoft.com/windows-server/networking/technologies/ipam/ipam-top) (tùy chọn dự phòng nếu không thiết lập IPAM nào)

## Direct Server Return (DSR) {#dsr}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [stable]`

Chế độ cân bằng tải trong đó việc chỉnh sửa địa chỉ IP và LBNAT diễn ra trực tiếp tại port vSwitch của container;
lưu lượng service đến nơi với IP nguồn được đặt là IP của Pod xuất phát.
Điều này mang lại các tối ưu hiệu năng bằng cách cho phép lưu lượng trả về vốn được định tuyến qua bộ cân bằng tải (load balancer)
bỏ qua bộ cân bằng tải và phản hồi trực tiếp cho client;
giảm tải cho bộ cân bằng tải và cũng giảm độ trễ tổng thể. Để biết thêm thông tin, hãy đọc
[Direct Server Return (DSR) in a nutshell](https://techcommunity.microsoft.com/blog/networkingblog/direct-server-return-dsr-in-a-nutshell/693710).

## Cân bằng tải và Service (Load balancing and Services)

Service trong Kubernetes là một khái niệm trừu tượng
định nghĩa một tập hợp logic các Pod và một phương thức để truy cập chúng qua mạng.
Trong một cluster có các node Windows, bạn có thể dùng các loại Service sau:

* `NodePort`
* `ClusterIP`
* `LoadBalancer`
* `ExternalName`

Mạng container trên Windows khác với mạng Linux ở một số điểm quan trọng.
[Tài liệu của Microsoft về Windows Container Networking](https://docs.microsoft.com/en-us/virtualization/windowscontainers/container-networking/architecture)
cung cấp thêm chi tiết và kiến thức nền.

Trên Windows, bạn có thể dùng các thiết lập sau để cấu hình Service và hành vi
cân bằng tải:

*Bảng: Các thiết lập Service trên Windows (Windows Service Settings)*

| Tính năng | Mô tả | Bản build Windows OS tối thiểu được hỗ trợ | Cách bật |
| ------- | ----------- | -------------------------- | ------------- |
| Session affinity | Đảm bảo các kết nối từ một client cụ thể luôn được chuyển tới cùng một Pod mỗi lần. | Windows Server 2022 | Đặt `service.spec.sessionAffinity` thành "ClientIP" |
| Direct Server Return (DSR) | Xem ghi chú về [DSR](#dsr) ở trên. | Windows Server 2019 | Đặt đối số dòng lệnh sau (giả sử phiên bản v1.36): ` --enable-dsr=true` |
| Preserve-Destination | Bỏ qua DNAT đối với lưu lượng service, nhờ đó giữ nguyên IP ảo của service đích trong các gói tin đến Pod backend. Cũng vô hiệu hóa chuyển tiếp node-node. | Windows Server, phiên bản 1903 | Đặt `"preserve-destination": "true"` trong các annotation của service và bật DSR trong kube-proxy. |
| Mạng dual-stack IPv4/IPv6 | Giao tiếp IPv4-tới-IPv4 nguyên bản song song với IPv6-tới-IPv6 tới, từ, và bên trong một cluster | Windows Server 2019 | Xem [IPv4/IPv6 dual-stack](https://kubernetes.io/docs/concepts/services-networking/dual-stack/#windows-support) |
| Giữ nguyên IP client | Đảm bảo IP nguồn của lưu lượng ingress đến được giữ nguyên. Cũng vô hiệu hóa chuyển tiếp node-node. |  Windows Server 2019  | Đặt `service.spec.externalTrafficPolicy` thành "Local" và bật DSR trong kube-proxy |

## Hạn chế (Limitations)

Các chức năng mạng sau đây _không_ được hỗ trợ trên các node Windows:

* Chế độ host networking
* Truy cập NodePort cục bộ từ chính node đó (vẫn hoạt động với các node khác hoặc client bên ngoài)
* Nhiều hơn 64 Pod backend (hoặc địa chỉ đích duy nhất) cho một Service
* Giao tiếp IPv6 giữa các Pod Windows kết nối vào mạng overlay
* Local Traffic Policy trong chế độ không phải DSR
* Giao tiếp ra bên ngoài bằng giao thức ICMP qua plugin `win-overlay`, `win-bridge`, hoặc plugin Azure-CNI.
  Cụ thể, data plane của Windows ([VFP](https://www.microsoft.com/research/project/azure-virtual-filtering-platform/))
  không hỗ trợ chuyển vị (transposition) gói tin ICMP, và điều này có nghĩa là:
  * Các gói ICMP hướng tới các đích trong cùng một mạng (chẳng hạn giao tiếp Pod tới Pod qua ping)
    hoạt động như mong đợi;
  * Các gói TCP/UDP hoạt động như mong đợi;
  * Các gói ICMP cần đi xuyên qua một mạng ở xa (ví dụ giao tiếp từ Pod ra internet bên ngoài qua ping)
    không thể được chuyển vị và do đó sẽ không được định tuyến trở lại nguồn của chúng;
  * Vì các gói TCP/UDP vẫn có thể được chuyển vị, bạn có thể thay `ping <destination>` bằng
    `curl <destination>` khi debug kết nối với thế giới bên ngoài.

Các hạn chế khác:

* Các network plugin tham chiếu của Windows là win-bridge và win-overlay không hiện thực
  [CNI spec](https://github.com/containernetworking/cni/blob/master/SPEC.md) v0.4.0,
  do thiếu hiện thực `CHECK`.
* Plugin Flannel VXLAN CNI có các hạn chế sau trên Windows:
  * Kết nối node-pod chỉ khả dụng cho các Pod cục bộ với Flannel v0.12.0 (trở lên).
  * Flannel bị giới hạn chỉ dùng VNI 4096 và UDP port 4789. Xem tài liệu chính thức về backend
    [Flannel VXLAN](https://github.com/coreos/flannel/blob/master/Documentation/backends.md#vxlan)
    để biết thêm chi tiết về các tham số này.

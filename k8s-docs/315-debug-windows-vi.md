# Mẹo debug Windows (Windows debugging tips)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-cluster/windows/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** tài liệu tra cứu thuộc nhánh `/docs/tasks/`
([Checkpoint tiếp nối](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)) — đọc kèm
[Giai đoạn 15 — Windows](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows)
(các bài [174](174-windows-vi.md), [175](175-windows-intro-vi.md),
[176](176-windows-user-guide-vi.md), [89](89-windows-networking-vi.md)) và bài
[216 — Adding Windows nodes](216-adding-windows-nodes-vi.md). Cluster lab toàn node Linux,
nên bài này chỉ cần khi môi trường của bạn có node Windows; Lab 15 (tùy chọn) mới là nơi
thực hành.

**Phải hiểu ở lần đọc này:**

- Hai lỗi cấp node hay gặp: Pod kẹt ở "Container Creating" hoặc restart liên tục thường do
  **pause image không tương thích phiên bản Windows** (với containerd, pause image nằm ở
  field `plugins.plugins.cri.sandbox_image` trong `config.toml`); `ErrImgPull` /
  `ImagePullBackOff` thường do Pod bị lập lịch lên node Windows **không tương thích** phiên
  bản.
- Đặc thù mạng Windows: chạy trên máy ảo thì phải **bật MAC spoofing** trên adapter mạng của
  VM; Pod Windows **không có rule outbound cho ICMP** (ping ra ngoài không được — thay bằng
  `curl` vì TCP/UDP được hỗ trợ); `ExceptionList` trong `cni.conf` liệt kê các dải địa chỉ
  không bị outbound NAT — IP ngoài cluster muốn truy vấn được thì **không** được nằm trong
  danh sách này.
- Hai giới hạn đã biết (known limitation) của networking stack trên Windows: node Windows
  không truy cập được Service kiểu `NodePort` từ chính nó (từ node khác hoặc client ngoài
  thì được), và node không truy cập được service IP (Pod Windows thì được).
- Với Flannel: node xóa đi join lại phải xóa `C:\k\SourceVip.json` và
  `C:\k\SourceVipRequest.json`; Pod không khởi chạy được vì thiếu
  `/run/flannel/subnet.env` nghĩa là flanneld chưa chạy đúng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Các lệnh sửa chi tiết bằng PowerShell/HNS (`Get-HnsNetwork`, `start.ps1`, chạy lại `flanneld.exe`, biến proxy máy) | chỉ có ý nghĩa khi đứng trên một node Windows thật | Lab 15 (tùy chọn) khi bạn thêm VM Windows Server vào cluster |
| `wincat` trong pause container cho `kubectl port-forward` | chi tiết lịch sử của bản Kubernetes cũ | tra cứu tại chỗ khi gặp đúng thông báo lỗi này |

---

## Khắc phục sự cố cấp node (Node-level troubleshooting) {#troubleshooting-node}

1. Các Pod của tôi kẹt ở "Container Creating" hoặc restart lặp đi lặp lại

   Hãy chắc chắn rằng pause image của bạn tương thích với phiên bản hệ điều hành Windows.
   Xem [Pause container](175-windows-intro-vi.md#pause-container)
   để biết pause image mới nhất / được khuyến nghị và/hoặc tìm hiểu thêm thông tin.

   > **Ghi chú:**
   > Nếu dùng containerd làm container runtime, pause image được chỉ định trong field
   > `plugins.plugins.cri.sandbox_image` của file cấu hình config.toml.

2. Các pod của tôi hiện trạng thái `ErrImgPull` hoặc `ImagePullBackOff`

   Hãy chắc chắn rằng Pod của bạn được lập lịch (schedule) lên một node Windows
   [tương thích](https://docs.microsoft.com/virtualization/windowscontainers/deploy-containers/version-compatibility).

   Thông tin thêm về cách chỉ định node tương thích cho Pod của bạn có trong
   [hướng dẫn này](176-windows-user-guide-vi.md#ensuring-os-specific-workloads-land-on-the-appropriate-container-host).

## Khắc phục sự cố mạng (Network troubleshooting) {#troubleshooting-network}

1. Các Pod Windows của tôi không có kết nối mạng

   Nếu bạn dùng máy ảo, hãy chắc chắn rằng MAC spoofing đã được **bật** trên tất cả các
   adapter mạng của VM.

2. Các Pod Windows của tôi không ping được các tài nguyên bên ngoài

   Pod Windows không được lập trình sẵn các rule outbound cho giao thức ICMP. Tuy nhiên,
   TCP/UDP được hỗ trợ. Khi muốn chứng minh khả năng kết nối tới các tài nguyên bên ngoài
   cluster, hãy thay `ping <IP>` bằng các lệnh `curl <IP>` tương ứng.

   Nếu bạn vẫn gặp vấn đề, nhiều khả năng cấu hình mạng của bạn trong
   [cni.conf](https://github.com/Microsoft/SDN/blob/master/Kubernetes/flannel/l2bridge/cni/config/cni.conf)
   đáng được chú ý thêm. Bạn luôn có thể chỉnh sửa file tĩnh này. Thay đổi cấu hình sẽ áp
   dụng cho mọi tài nguyên Kubernetes mới.

   Một trong các yêu cầu về mạng của Kubernetes
   (xem [mô hình Kubernetes](157-networking-vi.md))
   là giao tiếp trong cluster phải diễn ra không cần NAT nội bộ. Để đáp ứng yêu cầu này, có
   một danh sách
   [ExceptionList](https://github.com/Microsoft/SDN/blob/master/Kubernetes/flannel/l2bridge/cni/config/cni.conf#L20)
   cho mọi giao tiếp mà bạn không muốn xảy ra outbound NAT. Tuy nhiên, điều này cũng có
   nghĩa là bạn cần loại IP bên ngoài mà bạn đang muốn truy vấn ra khỏi `ExceptionList`. Chỉ
   khi đó traffic xuất phát từ các pod Windows của bạn mới được SNAT đúng cách để nhận được
   phản hồi từ thế giới bên ngoài. Về mặt này, `ExceptionList` trong `cni.conf` của bạn nên
   trông như sau:

   ```conf
   "ExceptionList": [
                   "10.244.0.0/16",  # Subnet của cluster
                   "10.96.0.0/12",   # Subnet của Service
                   "10.127.130.0/24" # Subnet quản trị (host)
               ]
   ```

3. Node Windows của tôi không truy cập được Service kiểu `NodePort`

   Việc truy cập NodePort cục bộ từ chính node đó sẽ thất bại. Đây là một giới hạn đã biết
   (known limitation). Truy cập NodePort vẫn hoạt động từ các node khác hoặc từ client bên
   ngoài.

4. Các vNIC và HNS endpoint của container đang bị xóa

   Vấn đề này có thể xảy ra khi tham số `hostname-override` không được truyền cho
   [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/).
   Để xử lý, người dùng cần truyền hostname cho kube-proxy như sau:

   ```powershell
   C:\k\kube-proxy.exe --hostname-override=$(hostname)
   ```

5. Node Windows của tôi không truy cập được các service của tôi qua service IP

   Đây là một giới hạn đã biết của networking stack trên Windows. Tuy nhiên, các Pod Windows
   vẫn truy cập được Service IP.

6. Không tìm thấy network adapter nào khi khởi động kubelet

   Networking stack của Windows cần một adapter ảo (virtual adapter) để mạng Kubernetes hoạt
   động. Nếu các lệnh sau không trả về kết quả nào (trong một shell chạy quyền admin), việc
   tạo mạng ảo — điều kiện tiên quyết để kubelet hoạt động — đã thất bại:

   ```powershell
   Get-HnsNetwork | ? Name -ieq "cbr0"
   Get-NetAdapter | ? Name -Like "vEthernet (Ethernet*"
   ```

   Trong trường hợp adapter mạng của host không phải là "Ethernet", thường nên sửa tham số
   [InterfaceName](https://github.com/microsoft/SDN/blob/master/Kubernetes/flannel/start.ps1#L7)
   của script `start.ps1`. Nếu không, hãy xem output của script `start-kubelet.ps1` để kiểm
   tra có lỗi trong quá trình tạo mạng ảo hay không.

7. Phân giải DNS không hoạt động đúng

   Xem các giới hạn DNS đối với Windows trong
   [mục này](10-dns-pod-service-vi.md#dns-windows).

8. `kubectl port-forward` thất bại với lỗi "unable to do port forwarding: wincat not found"

   Tính năng này được hiện thực trong Kubernetes 1.15 bằng cách đưa `wincat.exe` vào pause
   infrastructure container `mcr.microsoft.com/oss/kubernetes/pause:3.6`. Hãy chắc chắn bạn
   dùng một phiên bản Kubernetes được hỗ trợ. Nếu bạn muốn tự build pause infrastructure
   container của riêng mình, hãy nhớ đưa vào
   [wincat](https://github.com/kubernetes/kubernetes/tree/master/build/pause/windows/wincat).

9. Việc cài đặt Kubernetes của tôi thất bại vì node Windows Server của tôi nằm sau proxy

   Nếu bạn nằm sau một proxy, các biến môi trường PowerShell sau phải được định nghĩa:

   ```powershell
   [Environment]::SetEnvironmentVariable("HTTP_PROXY", "http://proxy.example.com:80/", [EnvironmentVariableTarget]::Machine)
   [Environment]::SetEnvironmentVariable("HTTPS_PROXY", "http://proxy.example.com:443/", [EnvironmentVariableTarget]::Machine)
   ```

### Khắc phục sự cố Flannel (Flannel troubleshooting)

1. Với Flannel, các node của tôi gặp vấn đề sau khi join lại cluster

   Bất cứ khi nào một node từng bị xóa được join lại vào cluster, flannelD sẽ cố gán một pod
   subnet mới cho node đó. Người dùng nên xóa các file cấu hình pod subnet cũ tại các đường
   dẫn sau:

   ```powershell
   Remove-Item C:\k\SourceVip.json
   Remove-Item C:\k\SourceVipRequest.json
   ```

2. Flanneld kẹt ở "Waiting for the Network to be created"

   Có rất nhiều báo cáo về [vấn đề này](https://github.com/coreos/flannel/issues/1066);
   nhiều khả năng đây là vấn đề về thời điểm (timing) khi management IP của mạng flannel
   được thiết lập. Một cách xử lý tạm là chạy lại `start.ps1` hoặc chạy lại thủ công như
   sau:

   ```powershell
   [Environment]::SetEnvironmentVariable("NODE_NAME", "<Windows_Worker_Hostname>")
   C:\flannel\flanneld.exe --kubeconfig-file=c:\k\config --iface=<Windows_Worker_Node_IP> --ip-masq=1 --kube-subnet-mgr=1
   ```

3. Các Pod Windows của tôi không khởi chạy được vì thiếu `/run/flannel/subnet.env`

   Điều này cho thấy Flannel đã không khởi chạy đúng. Bạn có thể thử khởi động lại
   `flanneld.exe`, hoặc chép thủ công các file từ `/run/flannel/subnet.env` trên Kubernetes
   master sang `C:\run\flannel\subnet.env` trên node worker Windows và sửa dòng
   `FLANNEL_SUBNET` thành một số khác. Ví dụ, nếu muốn node subnet là 10.244.4.1/24:

   ```env
   FLANNEL_NETWORK=10.244.0.0/16
   FLANNEL_SUBNET=10.244.4.1/24
   FLANNEL_MTU=1500
   FLANNEL_IPMASQ=true
   ```

### Điều tra thêm (Further investigation)

Nếu các bước trên không giải quyết được vấn đề của bạn, bạn có thể tìm trợ giúp về việc chạy
Windows container trên các node Windows trong Kubernetes qua:

* Chủ đề [Windows Server Container](https://stackoverflow.com/questions/tagged/windows-server-container) trên StackOverflow
* Diễn đàn chính thức của Kubernetes [discuss.kubernetes.io](https://discuss.kubernetes.io/)
* Kênh Slack của Kubernetes [#SIG-Windows](https://kubernetes.slack.com/messages/sig-windows)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một lần đọc tra cứu:

1. Cluster lab của bạn toàn node Linux chạy trên VM. Nếu sau này bạn thêm một node Windows
   cũng chạy trên máy ảo, thiết lập nào ở tầng ảo hóa phải bật thì Pod Windows mới có kết
   nối mạng, và trên Windows node dùng containerd thì pause image được khai báo ở đâu?
2. **Câu bẫy.** Từ một Pod Windows, `ping 8.8.8.8` không có phản hồi. Có thể kết luận Pod
   mất kết nối ra ngoài không? Nếu không, bạn kiểm chứng bằng cách nào, và nếu `curl` cũng
   thất bại thì file cấu hình nào cùng danh sách nào cần xem — theo hướng thêm hay bớt dải
   địa chỉ?
3. Đứng trên chính node Windows, bạn không truy cập được Service kiểu `NodePort` lẫn service
   IP. Có phải cluster đang hỏng không, và Pod Windows trên node đó thì sao?
4. Một node Windows dùng Flannel bị xóa khỏi cluster rồi join lại thì trục trặc mạng. Phải
   xóa những file nào, và triệu chứng "Pod không khởi chạy được vì thiếu
   `/run/flannel/subnet.env`" nói lên điều gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Phải **bật MAC spoofing trên tất cả adapter mạng của VM** — không bật thì Pod Windows
   không có kết nối mạng. Với containerd, pause image nằm ở field
   **`plugins.plugins.cri.sandbox_image`** trong file **`config.toml`**; pause image không
   tương thích phiên bản Windows là nguyên nhân điển hình khiến Pod kẹt ở
   "Container Creating" hoặc restart liên tục.
2. **Không thể kết luận như vậy.** Pod Windows **không có rule outbound cho ICMP**, nhưng
   TCP/UDP được hỗ trợ — kiểm chứng bằng **`curl <IP>`** thay cho `ping <IP>`. Nếu `curl`
   cũng thất bại, xem **`ExceptionList` trong `cni.conf`** theo hướng **bớt**: IP bên ngoài
   mà bạn truy vấn phải **không nằm trong** `ExceptionList` thì traffic mới được SNAT đúng
   để nhận phản hồi. Trực giác "ping chết là mất mạng" sai vì ở đây ping chết theo thiết kế.
3. **Không phải cluster hỏng** — cả hai đều là **giới hạn đã biết** của networking stack
   trên Windows: NodePort không truy cập được từ chính node đó (từ node khác hoặc client
   ngoài thì được), và node không truy cập được service IP. Trong khi đó, **Pod Windows
   trên node vẫn truy cập được Service IP** bình thường.
4. Xóa **`C:\k\SourceVip.json`** và **`C:\k\SourceVipRequest.json`** — vì khi node join
   lại, flannelD cố gán pod subnet mới trong khi cấu hình subnet cũ vẫn còn. Thiếu
   `/run/flannel/subnet.env` nghĩa là **Flannel đã không khởi chạy đúng**: khởi động lại
   `flanneld.exe`, hoặc chép file từ master sang `C:\run\flannel\subnet.env` trên worker
   Windows rồi sửa `FLANNEL_SUBNET` sang subnet mong muốn.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng; khi thực sự vận hành node
Windows, làm Lab 15 (tùy chọn) và đọc lại nhóm bài
[Giai đoạn 15](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows).

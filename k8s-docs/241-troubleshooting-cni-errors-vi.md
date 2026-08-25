# Khắc phục sự cố lỗi liên quan tới CNI plugin (Troubleshooting CNI plugin-related errors)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/troubleshooting-cni-plugin-related-errors/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 27 — Di chuyển khỏi dockershim (cluster cũ)](00-ALO-TRINH-ADMIN.md#giai-đoạn-27--di-chuyển-khỏi-dockershim-cluster-cũ),
bài 6/6, **bài cuối giai đoạn** · Phần II không có lab riêng: tự chấm bằng **Checkpoint** ghi ở cuối
mục giai đoạn 27 trong lộ trình. Phần đọc được trên cluster lab: trường `cniVersion` trong file cấu
hình CNI trên node — bạn đã mở đúng chỗ đó ở
[Lab 5b phần B2](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md#b2-cni-đang-chạy-đọc-cấu-hình-thật-trên-node).

**Bài này khác năm bài trước của giai đoạn 27: nó không nói về dockershim.** Nó nằm trong mục di
chuyển khỏi dockershim vì hai lỗi dưới đây nổ ra đúng vào đợt nhiều người đổi runtime, nhưng nguyên
nhân là **lệch phiên bản giữa CNI plugin và file cấu hình CNI** — thứ có thể gặp trên bất kỳ cluster
nào chạy containerd, kể cả cluster đã dùng containerd từ đầu như cluster lab. Vì vậy đây là bài duy
nhất của giai đoạn mà bạn nên đọc kỹ ngay cả khi bỏ qua phần còn lại. Riêng cửa sổ phiên bản cụ thể
(containerd v1.6.0–v1.6.3) thì thuộc về quá khứ: baseline của Lab 00 nằm ngoài cửa sổ đó, nên đừng
cố tái hiện lỗi trên `lab-k8s-worker2`.

**Phải hiểu ở lần đọc này:**

- Cả bài quy về **một nguyên nhân**: phiên bản CNI plugin trên node không khớp với phiên bản đặc tả
  khai trong file cấu hình CNI. Cửa sổ cụ thể là containerd **v1.6.0–v1.6.3** khi CNI plugin chưa
  được nâng cấp và/hoặc phiên bản cấu hình CNI không được khai; đội containerd cho biết đã giải
  quyết từ **v1.6.4**.
- Hai triệu chứng nổ ra ở **hai thời điểm khác nhau của vòng đời Pod**: *Incompatible CNI versions*
  xuất hiện trong log containerd **khi khởi động Pod**, do phiên bản trong cấu hình **mới hơn**
  phiên bản plugin hỗ trợ; còn *Failed to destroy network for sandbox* xuất hiện **khi dừng Pod**,
  do cấu hình **thiếu hẳn** phiên bản.
- Hậu quả của ca thứ hai là thứ dễ bỏ sót nhất: Pod **vẫn chạy được**, chỉ chết lúc gỡ mạng, và nó
  **kẹt ở trạng thái not-ready với một network namespace vẫn còn gắn kèm**. Sửa file cấu hình CNI để
  bổ sung thông tin phiên bản thì **lần dừng Pod tiếp theo** sẽ thành công.
- Quy trình sửa là **một vòng bảo trì node, làm từng node**: drain và cordon an toàn theo bài
  [255](255-safely-drain-node-vi.md) → **dừng container runtime và kubelet rồi mới** nâng cấp CNI
  plugin, thay plugin không phải CNI bằng CNI plugin, sửa file cấu hình để khai đúng một phiên bản
  đặc tả CNI mà plugin hỗ trợ, nâng cấp thành phần node và container runtime → khởi động lại runtime
  và kubelet, rồi `kubectl uncordon <nodename>`.
- Vì sao `loopback` là chỗ vướng riêng của containerd: containerd tự thêm interface `lo` vào pod
  bằng CNI plugin `loopback`, và **phần cấu hình cho loopback được containerd làm nội bộ, đặt cứng ở
  CNI v1.0.0**. Hệ quả: khi containerd đời mới khởi động, phiên bản plugin `loopback` **phải là
  v1.0.0 trở lên**, nếu không chính cái interface tầm thường nhất lại là cái làm hỏng pod.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Khối bash sinh `/etc/cni/net.d/10-containerd-net.conflist` với plugin `bridge` và `portmap`, dải `10.88.0.0/16` và `2001:db8:4860::/64` | đó là cấu hình mạng bridge mặc định của containerd, **không** phải CNI mà cluster lab đang chạy; bài cũng dặn phải đổi dải IP theo kế hoạch đánh địa chỉ riêng. Điều cần lấy từ khối này chỉ là **chỗ đặt `cniVersion`** | cấu hình CNI thật của cluster lab đã đọc ở [Lab 5b phần B2](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md#b2-cni-đang-chạy-đọc-cấu-hình-thật-trên-node) |
| Bước "nâng cấp các thành phần của node, ví dụ kubelet, lên Kubernetes v1.24" trong danh sách sửa | là mốc lịch sử của thời điểm gỡ dockershim, không phải phiên bản đích của bạn | quy trình nâng cấp node đã học ở [giai đoạn 17](00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster) |
| Ghi chú về gói phát hành containerd mang định danh `cni` và cách loopback plugin được đóng gói kèm | là chi tiết đóng gói của một dự án bên ngoài | chỉ áp dụng khi tự cài hoặc nâng cấp containerd trên node |

---

Để tránh các lỗi liên quan tới CNI plugin, hãy xác nhận rằng bạn đang dùng hoặc nâng cấp lên một
container runtime đã được kiểm thử là hoạt động chính xác với phiên bản Kubernetes của bạn.

## Về các lỗi "Incompatible CNI versions" và "Failed to destroy network for sandbox" (About the "Incompatible CNI versions" and "Failed to destroy network for sandbox" errors)

Có các sự cố dịch vụ khi thiết lập (setup) và gỡ bỏ (tear down) mạng CNI cho pod trong containerd
v1.6.0-v1.6.3 khi các CNI plugin chưa được nâng cấp và/hoặc phiên bản cấu hình CNI không được
khai báo trong các file cấu hình CNI. Đội ngũ containerd cho biết: "các sự cố này đã được giải
quyết trong containerd v1.6.4."

Với containerd v1.6.0-v1.6.3, nếu bạn không nâng cấp các CNI plugin và/hoặc không khai báo
phiên bản cấu hình CNI, bạn có thể gặp các tình huống lỗi "Incompatible CNI versions" hoặc
"Failed to destroy network for sandbox" sau đây.

### Lỗi Incompatible CNI versions (Incompatible CNI versions error)

Nếu phiên bản CNI plugin của bạn không khớp đúng với phiên bản plugin trong cấu hình, do phiên
bản trong cấu hình mới hơn phiên bản plugin, log của containerd nhiều khả năng sẽ hiển thị một
thông báo lỗi khi khởi động pod, tương tự như:

```
incompatible CNI versions; config is \"1.0.0\", plugin supports [\"0.1.0\" \"0.2.0\" \"0.3.0\" \"0.3.1\" \"0.4.0\"]"
```

Để sửa sự cố này, hãy
[cập nhật các CNI plugin và các file cấu hình CNI của bạn](#updating-your-cni-plugins-and-cni-config-files).

### Lỗi Failed to destroy network for sandbox (Failed to destroy network for sandbox error)

Nếu phiên bản plugin bị thiếu trong cấu hình CNI plugin, pod vẫn có thể chạy. Tuy nhiên, việc
dừng pod sẽ sinh ra một lỗi tương tự như:

```
ERROR[2022-04-26T00:43:24.518165483Z] StopPodSandbox for "b" failed
error="failed to destroy network for sandbox \"bbc85f891eaf060c5a879e27bba9b6b06450210161dfdecfbb2732959fb6500a\": invalid version \"\": the version is empty"
```

Lỗi này khiến pod bị kẹt ở trạng thái not-ready với một network namespace vẫn còn gắn kèm. Để
khôi phục khỏi vấn đề này, hãy
[chỉnh sửa file cấu hình CNI](#updating-your-cni-plugins-and-cni-config-files) để bổ sung thông
tin phiên bản còn thiếu. Lần dừng pod tiếp theo sẽ thành công.

### Cập nhật các CNI plugin và các file cấu hình CNI của bạn (Updating your CNI plugins and CNI config files) {#updating-your-cni-plugins-and-cni-config-files}

Nếu bạn đang dùng containerd v1.6.0-v1.6.3 và gặp các lỗi "Incompatible CNI versions" hoặc
"Failed to destroy network for sandbox", hãy cân nhắc cập nhật các CNI plugin và chỉnh sửa các
file cấu hình CNI.

Dưới đây là tổng quan các bước điển hình cho từng node:

1. [Drain và cordon node một cách an toàn](255-safely-drain-node-vi.md).
1. Sau khi dừng các dịch vụ container runtime và kubelet, thực hiện các thao tác nâng cấp sau:

   - Nếu bạn đang chạy các CNI plugin, hãy nâng cấp chúng lên phiên bản mới nhất.
   - Nếu bạn đang dùng các plugin không phải CNI, hãy thay thế chúng bằng các CNI plugin. Dùng
     phiên bản mới nhất của các plugin.
   - Cập nhật file cấu hình plugin để chỉ định hoặc khớp với một phiên bản của đặc tả
     (specification) CNI mà plugin hỗ trợ, như minh họa trong mục
     ["Một file cấu hình containerd ví dụ"](#an-example-containerd-configuration-file) bên dưới.
   - Với `containerd`, hãy bảo đảm rằng bạn đã cài đặt phiên bản mới nhất (v1.0.0 trở lên)
     của CNI loopback plugin.
   - Nâng cấp các thành phần của node (ví dụ, kubelet) lên Kubernetes v1.24
   - Nâng cấp lên hoặc cài đặt phiên bản mới nhất của container runtime.

1. Đưa node trở lại cluster của bạn bằng cách khởi động lại container runtime và kubelet.
   Uncordon node (`kubectl uncordon <nodename>`).

## Một file cấu hình containerd ví dụ (An example containerd configuration file) {#an-example-containerd-configuration-file}

Ví dụ sau đây trình bày một cấu hình cho runtime `containerd` v1.6.x, hỗ trợ một phiên bản gần
đây của đặc tả CNI (v1.0.0).

Vui lòng xem tài liệu từ nhà cung cấp plugin và nhà cung cấp mạng của bạn để có thêm hướng dẫn
về việc cấu hình hệ thống của bạn.

Trên Kubernetes, containerd runtime thêm một giao diện loopback, `lo`, vào các pod theo hành vi
mặc định. Containerd runtime cấu hình giao diện loopback thông qua một CNI plugin tên là
`loopback`. Plugin `loopback` được phân phối như một phần của các gói phát hành `containerd`
mang định danh `cni`. `containerd` v1.6.0 trở lên bao gồm một loopback plugin tương thích CNI
v1.0.0 cùng với các CNI plugin mặc định khác. Việc cấu hình cho loopback plugin được containerd
thực hiện nội bộ, và được đặt dùng CNI v1.0.0. Điều này cũng có nghĩa là phiên bản của plugin
`loopback` phải là v1.0.0 trở lên khi phiên bản `containerd` mới này được khởi động.

Lệnh bash sau đây sinh ra một cấu hình CNI ví dụ. Ở đây, giá trị 1.0.0 cho phiên bản cấu hình
được gán vào trường `cniVersion` để dùng khi `containerd` gọi CNI bridge plugin.

```bash
cat << EOF | tee /etc/cni/net.d/10-containerd-net.conflist
{
 "cniVersion": "1.0.0",
 "name": "containerd-net",
 "plugins": [
   {
     "type": "bridge",
     "bridge": "cni0",
     "isGateway": true,
     "ipMasq": true,
     "promiscMode": true,
     "ipam": {
       "type": "host-local",
       "ranges": [
         [{
           "subnet": "10.88.0.0/16"
         }],
         [{
           "subnet": "2001:db8:4860::/64"
         }]
       ],
       "routes": [
         { "dst": "0.0.0.0/0" },
         { "dst": "::/0" }
       ]
     }
   },
   {
     "type": "portmap",
     "capabilities": {"portMappings": true},
     "externalSetMarkChain": "KUBE-MARK-MASQ"
   }
 ]
}
EOF
```

Hãy cập nhật các dải địa chỉ IP trong ví dụ trên bằng những dải dựa trên trường hợp sử dụng và
kế hoạch đánh địa chỉ mạng của bạn.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 27:

1. **Câu bẫy.** Một Pod đã chạy êm nhiều ngày, mạng thông, không log lỗi nào. Vậy chắc chắn file cấu
   hình CNI trên node đó không thiếu thông tin phiên bản, đúng không?
2. Hai lỗi của bài đều là "sai phiên bản", nhưng nguyên nhân khác nhau và nổ ra ở hai thời điểm khác
   nhau. Chỉ ra khác biệt của cả hai vế.
3. Cluster lab chạy containerd cùng CNI plugin bạn tự cài ở Lab 5b. Theo bài, điều kiện nào quyết
   định việc hai lỗi này có thể xảy ra trên `lab-k8s-worker2` hay không — và bạn đọc điều kiện đó ở
   đâu trên node?
4. Vì sao quy trình sửa bắt buộc phải drain node và **dừng container runtime cùng kubelet trước**
   khi thay CNI plugin, thay vì thay nóng? Và vì sao riêng với containerd còn phải để mắt tới plugin
   `loopback`?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không chắc chắn gì cả — đây đúng là cái bẫy của bài.** Khi cấu hình CNI **thiếu** phiên bản
   plugin, bài nói rõ **pod vẫn có thể chạy**; lỗi chỉ bật ra ở **lúc dừng pod**, dưới dạng *Failed
   to destroy network for sandbox* với thông điệp `invalid version "": the version is empty`. Nghĩa
   là một cluster có thể mang lỗi này im lặng suốt thời gian dài, và chỉ lộ ra đúng lúc bạn cần xoay
   vòng workload. Hậu quả cũng không êm: **pod kẹt ở trạng thái not-ready với network namespace vẫn
   còn gắn kèm**.
2. **Nguyên nhân:** *Incompatible CNI versions* là **có** phiên bản trong cấu hình nhưng phiên bản
   đó **mới hơn** thứ plugin hỗ trợ — thông điệp lỗi liệt kê thẳng danh sách phiên bản plugin hỗ
   trợ; còn *Failed to destroy network for sandbox* là cấu hình **thiếu hẳn** phiên bản. **Thời
   điểm:** cái đầu nổ **khi khởi động pod**, cái sau nổ **khi dừng pod**. Cùng một họ nguyên nhân,
   nhưng một cái chặn ngay từ đầu còn một cái để bạn chạy rồi mới kẹt.
3. Điều kiện là **cửa sổ phiên bản containerd v1.6.0–v1.6.3**, cộng với việc CNI plugin chưa được
   nâng cấp và/hoặc **`cniVersion` không được khai** trong file cấu hình CNI. Đội containerd cho
   biết **các sự cố này đã được giải quyết trong containerd v1.6.4**, nên cluster chạy containerd
   mới hơn cửa sổ đó không nằm trong phạm vi lỗi. Hai thứ cần đọc trên node: **phiên bản containerd**
   và **trường `cniVersion` trong file cấu hình CNI** — đúng file bạn đã mở ở
   [Lab 5b phần B2](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md#b2-cni-đang-chạy-đọc-cấu-hình-thật-trên-node).
4. Vì bài xếp toàn bộ việc sửa vào **một vòng bảo trì node**: bước 1 là
   [drain và cordon node an toàn](255-safely-drain-node-vi.md), bước 2 chỉ bắt đầu **sau khi đã dừng
   các dịch vụ container runtime và kubelet**, bước 3 là khởi động lại chúng rồi `kubectl uncordon`.
   Thay plugin ngay dưới chân một runtime đang tạo và gỡ sandbox là tự chuốc đúng hai lỗi mà bài
   đang chữa. Còn `loopback`: containerd **tự cấu hình plugin này nội bộ và đặt cứng ở CNI v1.0.0**,
   nên nó không nằm trong file cấu hình bạn sửa tay và rất dễ bị bỏ quên — **phiên bản plugin
   `loopback` phải là v1.0.0 trở lên** khi containerd đời mới khởi động, nếu không mọi pod đều hỏng
   ở đúng cái interface `lo`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là **bài cuối của
[giai đoạn 27](00-ALO-TRINH-ADMIN.md#giai-đoạn-27--di-chuyển-khỏi-dockershim-cluster-cũ)** — trước
khi sang giai đoạn 28, tự chấm bằng **Checkpoint của giai đoạn 27**: chạy lệnh của bài
[239](239-find-out-runtime-vi.md) trên `lab-k8s-master` để xác định container runtime đang dùng, rồi
nói lại thành lời dockershim là gì, vì sao bị gỡ, và phải kiểm những gì trước khi chuyển một node
sang containerd.

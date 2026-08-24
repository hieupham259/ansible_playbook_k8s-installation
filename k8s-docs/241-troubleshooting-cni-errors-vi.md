# Khắc phục sự cố lỗi liên quan tới CNI plugin (Troubleshooting CNI plugin-related errors)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/troubleshooting-cni-plugin-related-errors/

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

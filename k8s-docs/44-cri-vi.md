# Container Runtime Interface (CRI)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/containers/cri/>

CRI là một giao diện plugin (plugin interface) cho phép kubelet sử dụng nhiều loại
container runtime khác nhau mà không cần phải biên dịch lại các thành phần của
cluster.

Bạn cần có một container runtime hoạt động trên mỗi Node trong cluster của mình, để
kubelet có thể khởi chạy các Pod và các container của chúng.

Container Runtime Interface (CRI) là giao thức chính cho việc giao tiếp giữa kubelet
và Container Runtime.

Kubernetes Container Runtime Interface (CRI) định nghĩa giao thức
[gRPC](https://grpc.io) chính cho việc giao tiếp giữa các
[thành phần của node (node components)](./22-architecture-vi.md#node-components) là
kubelet và container runtime.

## API {#api}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [stable]`

kubelet đóng vai trò là client khi kết nối tới container runtime qua gRPC. Các
endpoint của dịch vụ runtime và dịch vụ image phải sẵn có trong container runtime;
chúng có thể được cấu hình riêng trong kubelet bằng
[cờ dòng lệnh](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
`--container-runtime-endpoint`.

Với Kubernetes v1.26 trở đi, kubelet yêu cầu container runtime phải hỗ trợ API CRI
`v1`. Nếu một container runtime không hỗ trợ API `v1`, kubelet sẽ không đăng ký
node.

## Nâng cấp (Upgrading)

Khi nâng cấp phiên bản Kubernetes trên một node, kubelet sẽ khởi động lại. Nếu
container runtime không hỗ trợ API CRI `v1`, kubelet sẽ đăng ký thất bại và báo
lỗi. Nếu cần thực hiện re-dial gRPC do container runtime đã được nâng cấp, runtime
đó phải hỗ trợ API CRI `v1` thì kết nối mới thành công. Điều này có thể yêu cầu
khởi động lại kubelet sau khi container runtime đã được cấu hình đúng.

## Liệt kê theo kiểu streaming (List streaming) {#list-streaming}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Các RPC liệt kê tiêu chuẩn của CRI (`ListContainers`, `ListPodSandbox`,
`ListImages`) trả về toàn bộ kết quả trong một phản hồi unary duy nhất. Trên các
node có số lượng container lớn (ví dụ, hơn khoảng 10.000 tính cả container đang
chạy lẫn đã dừng), các phản hồi này có thể vượt quá giới hạn kích thước message
mặc định 16 MiB của gRPC, khiến kubelet thất bại khi đối chiếu (reconcile) trạng
thái với container runtime.

Khi feature gate `CRIListStreaming` được bật, kubelet sử dụng các RPC streaming
phía server (chẳng hạn `StreamContainers`, `StreamPodSandboxes`, `StreamImages`)
cho phép container runtime chia kết quả thành nhiều message phản hồi, vượt qua
giới hạn kích thước trên mỗi message. Điều này đặc biệt hữu ích cho:

- Các môi trường có tốc độ thay đổi container cao (các hệ thống CI/CD)
- Các workload xử lý batch quy mô lớn

Nếu container runtime không hỗ trợ các RPC streaming, kubelet sẽ tự động quay về
(fall back) các RPC unary tiêu chuẩn để đảm bảo tương thích ngược.

## Tiếp theo (What's next)

- Tìm hiểu thêm về [định nghĩa giao thức (protocol definition)](https://github.com/kubernetes/cri-api/blob/v0.33.1/pkg/apis/runtime/v1/api.proto) CRI

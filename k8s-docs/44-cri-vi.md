# Container Runtime Interface (CRI)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/containers/cri/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 2](00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime), bài 5/8 ·
Kiểm chứng ở Lab 2 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) bạn đã chạy `crictl info` và kiểm tra "CRI API là `v1`"
mà chưa biết vì sao điều đó quan trọng. Bài này trả lời.

**Phải hiểu ở lần đọc này:**

- CRI là **hợp đồng giữa kubelet và container runtime**, dùng gRPC. Nhờ nó, đổi runtime không
  phải biên dịch lại thành phần nào của cluster.
- **Kubelet là client, runtime là server.** Endpoint được chỉ định bằng
  `--container-runtime-endpoint` — chính giá trị bạn đã đặt trong `/etc/crictl.yaml` ở Lab 00.
- Runtime phải phục vụ **hai dịch vụ**: runtime service và image service.
- Từ Kubernetes v1.26, runtime **bắt buộc** hỗ trợ CRI API `v1`. Không hỗ trợ thì **kubelet
  không đăng ký node** — đây là nguyên nhân trực tiếp của một dạng sự cố `NotReady` sau nâng
  cấp.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Nâng cấp* và tình huống re-dial gRPC | chỉ gặp khi nâng cấp cluster thật | CP2 |
| *Liệt kê theo kiểu streaming* và `CRIListStreaming` (alpha) | chỉ có ý nghĩa ở node hơn 10.000 container | không cần |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Trong quan hệ CRI, ai là client và ai là server?
2. CRI giải quyết vấn đề gì? Không có nó thì việc đổi container runtime sẽ tốn kém ra sao?
3. Bạn nâng cấp một node lên Kubernetes mới, nhưng container runtime trên node đó chỉ hỗ trợ
   CRI API cũ. Triệu chứng bạn nhìn thấy là gì?
4. Ở Lab 00 bạn viết `runtime-endpoint: unix:///run/containerd/containerd.sock` vào
   `/etc/crictl.yaml`. Giá trị đó tương ứng với thứ gì trong bài này?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Kubelet là client**, kết nối tới **container runtime** đóng vai server, qua gRPC.
2. Nó cho phép kubelet dùng **nhiều loại container runtime khác nhau mà không phải biên dịch
   lại các thành phần của cluster**. Không có CRI, mỗi runtime mới sẽ đòi một bản kubelet
   riêng — tức là mỗi tổ hợp runtime × phiên bản Kubernetes là một bản build.
3. **Kubelet đăng ký node thất bại và báo lỗi**, nên node không lên `Ready`. Từ v1.26 kubelet
   yêu cầu runtime hỗ trợ CRI API `v1`; không có thì nó không đăng ký node.
4. Đó là **endpoint của dịch vụ runtime và image** mà CRI nói tới — cùng thứ mà kubelet nhận
   qua cờ `--container-runtime-endpoint`. `crictl` nói chuyện với runtime qua đúng giao thức
   CRI mà kubelet dùng, nên nó là công cụ debug đúng tầng khi node có vấn đề.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

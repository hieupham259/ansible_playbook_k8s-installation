# Công cụ dòng lệnh kubectl (The kubectl command-line tool)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/kubectl/>
>
> kubectl là công cụ dòng lệnh chính để giao tiếp với một cluster Kubernetes.
> Trang này cung cấp cái nhìn tổng quan về kubectl và vai trò của nó trong hệ sinh thái Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1b](LO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl),
bài 6/9 · Kiểm chứng ở Lab 1b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này là trang tổng quan, không phải tài liệu tham khảo lệnh. Đừng cố nhớ cú pháp ở đây —
cú pháp học bằng cách gõ trong lab.

**Phải hiểu ở lần đọc này:**

- `kubectl` chỉ là **một client của API**. Mọi lệnh, kể cả `kubectl logs`, đều biến thành HTTP
  request tới API server. Điều này khớp trực tiếp với bài [24](24-control-plane-node-communication-vi.md).
- Nó lấy địa chỉ server và thông tin xác thực từ **kubeconfig**; chạy trong Pod thì dùng token
  ServiceAccount được mount vào.
- Thứ tự tìm cấu hình: flag `--kubeconfig`, biến `KUBECONFIG`, rồi `$HOME/.kube/config`.
- **Declarative (`apply`) cho production, imperative (`create`, `run`) cho thử nghiệm** — ý
  này được khai triển đầy đủ ở bài [27](27-object-management-vi.md).
- Version skew: `kubectl` chạy được với control plane lệch tối đa **một** phiên bản minor.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nhóm thao tác *Vận hành cluster* (drain, cordon) | là quy trình bảo trì node | giai đoạn 12 và CP1 |
| Plugin và Krew | mở rộng tùy chọn | khi cần |
| JSONPath | học dần qua thực hành | dùng suốt lộ trình |

---

Kubernetes cung cấp một công cụ dòng lệnh (command line tool) để giao tiếp với
control plane của một cluster Kubernetes, thông qua Kubernetes API.

Công cụ `kubectl` giao tiếp với cluster của bạn thông qua [Kubernetes API](./21-kubernetes-api-vi.md).
Về cấu hình, `kubectl` tìm một file có tên `config` trong thư mục `$HOME/.kube`.
Bạn có thể chỉ định các file [kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
khác bằng cách đặt biến môi trường `KUBECONFIG` hoặc dùng flag
[`--kubeconfig`](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/).

## Vai trò của kubectl (Role of kubectl)

Công cụ `kubectl` là giao diện chính để tạo, xem xét (inspect), cập nhật và xóa các object của Kubernetes.
Nó bổ trợ cho [các thành phần của Kubernetes](./15-components-vi.md) chạy bên trong cluster của bạn
và [Kubernetes API](./21-kubernetes-api-vi.md) mà các thành phần đó hiện thực.
Dù bạn chạy `kubectl` từ laptop hay từ một Pod bên trong cluster, nó đều gửi request đến API server.
Các client khác, chẳng hạn [thư viện client](https://kubernetes.io/docs/reference/using-api/client-libraries/) và các web dashboard
như [Headlamp](https://headlamp.dev/), cũng giao tiếp thông qua cùng một API này.

## Cách kubectl hoạt động (How kubectl works)

Công cụ `kubectl` kết nối đến API server và xác thực (authenticate) bằng cluster, user và context được định nghĩa trong file
[kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) của bạn.
Khi bạn chạy `kubectl` từ bên ngoài cluster, nó dùng file kubeconfig để tìm địa chỉ API server và thông tin xác thực (credentials).
Khi `kubectl` chạy bên trong một Pod (ví dụ trong một pipeline CI/CD), nó có thể dùng cơ chế xác thực trong cluster (in-cluster authentication)
dựa trên token ServiceAccount được mount vào Pod.

Khi bạn chạy một lệnh, `kubectl` chuyển ý định của bạn thành một hoặc nhiều HTTP request gửi đến
[Kubernetes API](./21-kubernetes-api-vi.md). API server kiểm tra tính hợp lệ của từng request,
áp dụng nó vào trạng thái cluster được lưu trong etcd, và
trả về kết quả. Điều này có nghĩa là mọi hành động của `kubectl`, dù là tạo một Deployment hay đọc log,
đều đi theo cùng một con đường dựa trên API (API-driven).

Vì kubeconfig của bạn có thể định nghĩa nhiều cluster, user và context, bạn có thể dùng `kubectl` để
chuyển qua lại giữa các cluster mà không cần cấu hình lại môi trường. Chạy `kubectl config use-context` để
đổi context đang hoạt động.

## Bạn có thể làm gì với kubectl (What you can do with kubectl)

Công cụ `kubectl` hỗ trợ nhiều thao tác, chia thành các nhóm lớn sau:

* **Quản lý resource** – Tạo, cập nhật và xóa các object như Pod, Deployment và Service.
  Dùng `kubectl apply` để quản lý theo kiểu khai báo (declarative) từ các file cấu hình.
* **Xem xét trạng thái cluster** – Liệt kê và mô tả (describe) các object, xem các event và kiểm tra mức sử dụng tài nguyên.
* **Debug** – Xem log từ các container, thực thi lệnh bên trong một container đang chạy, hoặc port-forward đến một Pod.
* **Vận hành cluster** – Drain node để bảo trì, cordon node để ngăn workload mới được xếp lên, và quản lý cấu hình cluster.
* **Viết script và tự động hóa** – Định dạng đầu ra thành JSON, YAML hoặc các cột tùy chỉnh bằng [JSONPath](https://kubernetes.io/docs/reference/kubectl/jsonpath/) để dùng trong các script và pipeline.

Về cú pháp, tài liệu tham khảo lệnh và các ví dụ, xem [tài liệu tham khảo kubectl](https://kubernetes.io/docs/reference/kubectl/).

## Khai báo so với mệnh lệnh (Declarative vs imperative)

Với các workload production, hãy ưu tiên [quản lý object theo kiểu khai báo](./27-object-management-vi.md)
bằng `kubectl apply` với các file cấu hình được quản lý phiên bản (version-controlled).
Quản lý theo kiểu khai báo giúp bạn theo dõi thay đổi, cộng tác và tích hợp với các quy trình GitOps.
Các câu lệnh mệnh lệnh (imperative commands) (như `kubectl create` hoặc `kubectl run`) hữu ích cho việc phát triển và thử nghiệm,
nhưng khó tái lập và khó kiểm toán (audit) hơn.

## Mở rộng kubectl bằng plugin (Extending kubectl with plugins)

Bạn có thể mở rộng `kubectl` bằng các [plugin](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/) bổ sung
các sub-command mới. Plugin là các file thực thi (binary) độc lập tuân theo quy ước đặt tên `kubectl-<plugin-name>`.
Cộng đồng Kubernetes duy trì nhiều plugin, và bạn có thể quản lý chúng bằng
trình quản lý plugin [Krew](https://krew.sigs.k8s.io/).

## Tương thích phiên bản (Version compatibility)

Công cụ `kubectl` hỗ trợ độ lệch phiên bản (version skew) cộng hoặc trừ một phiên bản minor so với
control plane của cluster. Ví dụ, `kubectl` v1.32 hoạt động được với control plane ở các phiên bản v1.31, v1.32 và v1.33.
Dùng phiên bản tương thích giúp tránh hành vi không mong muốn. Xem
[chính sách lệch phiên bản (version skew policy)](https://kubernetes.io/releases/version-skew-policy/) để biết chi tiết.

## Tiếp theo (What's next)

* Đọc [tài liệu tham khảo kubectl](https://kubernetes.io/docs/reference/kubectl/) để biết cú pháp và chi tiết các lệnh.
* [Cài đặt kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) trên máy của bạn.
* Tìm hiểu về [Kubernetes API](./21-kubernetes-api-vi.md) mà `kubectl` sử dụng.
* Xem lại [các thành phần của Kubernetes](./15-components-vi.md) cấu thành một cluster.
* Khám phá [Quản lý object](./27-object-management-vi.md) và cấu hình khai báo.
* Xem [chính sách lệch phiên bản](https://kubernetes.io/releases/version-skew-policy/) để biết các tổ hợp phiên bản được hỗ trợ.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Khi bạn chạy `kubectl logs`, `kubectl` có mở kết nối trực tiếp tới kubelet hay tới etcd
   không? (Nối với bài [24](24-control-plane-node-communication-vi.md).)
2. Cluster lab của bạn ở `v1.35`. Những phiên bản `kubectl` nào được hỗ trợ?
3. `kubectl` tìm cấu hình ở đâu, theo thứ tự ưu tiên nào?
4. Một Pod trong cluster cần gọi Kubernetes API. Nó lấy thông tin xác thực từ đâu, khi trong
   Pod không có `$HOME/.kube/config`?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** `kubectl` chỉ nói chuyện với API server; chính API server mới mở kết nối tới
   kubelet để lấy log. Bài này nói mọi hành động của `kubectl`, dù là tạo Deployment hay đọc
   log, đều đi theo cùng một con đường dựa trên API.
2. `v1.34`, `v1.35` và `v1.36` — lệch tối đa một phiên bản minor về mỗi phía.
3. Flag `--kubeconfig` trước, rồi biến môi trường `KUBECONFIG`, cuối cùng mới là mặc định
   `$HOME/.kube/config`.
4. Từ **token ServiceAccount được mount vào Pod** — cơ chế xác thực trong cluster. Cùng cơ chế
   mà bài [24](24-control-plane-node-communication-vi.md) mô tả khi nói Kubernetes tự động đưa
   certificate gốc và bearer token vào Pod lúc khởi tạo.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

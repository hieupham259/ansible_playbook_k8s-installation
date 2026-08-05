# Hướng dẫn tăng cường bảo mật - Cấu hình Scheduler (Hardening Guide - Scheduler Configuration)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/hardening-guide/scheduler/>
>
> Thông tin về cách làm cho Kubernetes scheduler an toàn hơn.

Kubernetes scheduler (bộ lập lịch) là
một trong những thành phần quan trọng của
control plane.

Tài liệu này trình bày cách cải thiện trạng thái bảo mật (security posture) của Scheduler.

Một scheduler bị cấu hình sai có thể gây ra các hệ quả về bảo mật.
Một scheduler như vậy có thể nhắm vào các node cụ thể và trục xuất (evict) các workload hoặc ứng dụng đang chia sẻ node và tài nguyên của node đó.
Điều này có thể hỗ trợ kẻ tấn công thực hiện [tấn công Yo-Yo](https://arxiv.org/abs/2105.00542): một cuộc tấn công nhắm vào autoscaler có lỗ hổng.

## Cấu hình kube-scheduler (kube-scheduler configuration)

### Các tùy chọn dòng lệnh về xác thực và phân quyền của Scheduler (Scheduler authentication & authorization command line options)

Khi thiết lập cấu hình xác thực, cần đảm bảo rằng
cơ chế xác thực của kube-scheduler nhất quán với cơ chế xác thực của kube-api-server.
Nếu bất kỳ yêu cầu nào thiếu header xác thực, việc xác thực nên được thực hiện thông qua kube-api-server,
[cho phép mọi hoạt động xác thực trong cluster được nhất quán](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/#original-request-username-and-group).

- `authentication-kubeconfig`: Hãy đảm bảo cung cấp một kubeconfig phù hợp để
  scheduler có thể lấy các tùy chọn cấu hình xác thực từ API Server.
  File kubeconfig này nên được bảo vệ bằng quyền truy cập file (file permission) nghiêm ngặt.
- `authentication-tolerate-lookup-failure`: Đặt giá trị này thành `false` để đảm bảo
  scheduler _luôn luôn_ tra cứu cấu hình xác thực của nó từ API server.
- `authentication-skip-lookup`: Đặt giá trị này thành `false` để đảm bảo
  scheduler _luôn luôn_ tra cứu cấu hình xác thực của nó từ API server.
- `authorization-always-allow-paths`: Các đường dẫn này nên trả về dữ liệu phù hợp
  với phân quyền ẩn danh (anonymous authorization). Mặc định là `/healthz,/readyz,/livez`.
- `profiling`: Đặt thành `false` để tắt các endpoint profiling — chúng cung cấp thông tin gỡ lỗi
  nhưng không nên được bật trên các cluster production vì chúng tiềm ẩn rủi ro từ chối dịch vụ (denial of service)
  hoặc rò rỉ thông tin. Đối số `--profiling` đã bị loại bỏ dần (deprecated) và giờ đây có thể được cung cấp thông qua
  [KubeScheduler DebuggingConfiguration](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/#DebuggingConfiguration).
  Profiling có thể được tắt thông qua cấu hình kube-scheduler bằng cách đặt `enableProfiling` thành `false`.
- `requestheader-client-ca-file`: Tránh truyền đối số này.

### Các tùy chọn dòng lệnh về mạng của Scheduler (Scheduler networking command line options)

- `bind-address`: Trong hầu hết các trường hợp, kube-scheduler không cần được truy cập từ bên ngoài.
  Đặt địa chỉ bind thành `localhost` là một thực hành an toàn.
- `permit-address-sharing`: Đặt giá trị này thành `false` để tắt việc chia sẻ kết nối thông qua `SO_REUSEADDR`.
  `SO_REUSEADDR` có thể dẫn đến việc tái sử dụng các kết nối đã kết thúc đang ở trạng thái `TIME_WAIT`.
- `permit-port-sharing`: Mặc định là `false`. Hãy dùng giá trị mặc định trừ khi bạn chắc chắn hiểu rõ các hệ quả bảo mật.

### Các tùy chọn dòng lệnh về TLS của Scheduler (Scheduler TLS command line options)

- `tls-cipher-suites`: Luôn cung cấp danh sách các bộ mã hóa (cipher suite) ưu tiên.
  Điều này đảm bảo việc mã hóa không bao giờ diễn ra với các bộ mã hóa không an toàn.

## Cấu hình lập lịch cho các scheduler tùy chỉnh (Scheduling configurations for custom schedulers)

Khi sử dụng các scheduler tùy chỉnh dựa trên mã nguồn lập lịch của Kubernetes, quản trị viên cluster cần thận trọng với
các plugin sử dụng các [điểm mở rộng (extension point)](https://kubernetes.io/docs/reference/scheduling/config/#extension-points) `queueSort`, `prefilter`, `filter`, hoặc `permit`.
Các điểm mở rộng này kiểm soát nhiều giai đoạn khác nhau của quá trình lập lịch,
và cấu hình sai có thể ảnh hưởng đến hành vi của kube-scheduler trong cluster của bạn.

### Những điểm cần cân nhắc chính (Key considerations)

- Tại một thời điểm, chỉ có thể bật đúng một plugin sử dụng điểm mở rộng `queueSort`.
  Bất kỳ plugin nào sử dụng `queueSort` đều nên được xem xét kỹ lưỡng.
- Các plugin triển khai điểm mở rộng `prefilter` hoặc `filter` có khả năng đánh dấu tất cả các node là không thể lập lịch (unschedulable).
  Điều này có thể khiến việc lập lịch cho các pod mới bị đình trệ hoàn toàn.
- Các plugin triển khai điểm mở rộng `permit` có thể ngăn chặn hoặc trì hoãn việc gán (binding) một Pod.
  Những plugin như vậy nên được quản trị viên cluster xem xét kỹ càng.

Khi sử dụng một plugin không thuộc danh sách [plugin mặc định](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins),
hãy cân nhắc tắt các điểm mở rộng `queueSort`, `filter` và `permit` như sau:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: my-scheduler
    plugins:
      # Tắt các plugin cụ thể cho từng điểm mở rộng khác nhau
      # Bạn có thể tắt tất cả plugin của một điểm mở rộng bằng "*"
      queueSort:
        disabled:
        - name: "*"             # Tắt tất cả các plugin queueSort
      # - name: "PrioritySort"  # Tắt một plugin queueSort cụ thể
      filter:
        disabled:
        - name: "*"                 # Tắt tất cả các plugin filter
      # - name: "NodeResourcesFit"  # Tắt một plugin filter cụ thể
      permit:
        disabled:
        - name: "*"               # Tắt tất cả các plugin permit
      # - name: "TaintToleration" # Tắt một plugin permit cụ thể
```

Cấu hình này tạo ra một scheduler profile tên là ` my-scheduler`.
Bất cứ khi nào `.spec` của một Pod không có giá trị cho `.spec.schedulerName`, kube-scheduler sẽ chạy cho Pod đó,
sử dụng cấu hình chính và các plugin mặc định của nó.
Nếu bạn định nghĩa một Pod với `.spec.schedulerName` được đặt thành `my-scheduler`, kube-scheduler sẽ chạy
nhưng với một cấu hình tùy chỉnh; trong cấu hình tùy chỉnh đó,
các điểm mở rộng `queueSort`, `filter` và `permit` bị tắt.
Nếu bạn sử dụng KubeSchedulerConfiguration này, không chạy bất kỳ scheduler tùy chỉnh nào,
và sau đó định nghĩa một Pod với `.spec.schedulerName` được đặt thành `nonexistent-scheduler`
(hoặc bất kỳ tên scheduler nào khác không tồn tại trong cluster của bạn), sẽ không có event nào được sinh ra cho pod đó.

## Không cho phép gán nhãn node (Disallow labeling nodes)

Quản trị viên cluster nên đảm bảo rằng người dùng cluster không thể gán nhãn (label) cho các node.
Một tác nhân độc hại có thể sử dụng `nodeSelector` để lập lịch các workload lên những node mà các workload đó không nên xuất hiện.

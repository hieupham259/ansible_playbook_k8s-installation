# Hướng dẫn tăng cường bảo mật - Cấu hình Scheduler (Hardening Guide - Scheduler Configuration)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/hardening-guide/scheduler/>
>
> Thông tin về cách làm cho Kubernetes scheduler an toàn hơn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 15/15 · Kiểm chứng ở [Lab 13](labs/LAB-13-DRA.md).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Riêng bài này là ngoại lệ dễ chịu: nó **không** alpha, và phần lớn nội
dung áp dụng được cho mọi cluster — kể cả cluster lab 3 VM trên VMware, vì mọi cluster đều có
kube-scheduler. Phần duy nhất lab không dùng tới là mục về **scheduler tùy chỉnh**.

Đây là **phần hoãn lại từ giai đoạn 9**. Lộ trình cố tình tách nó khỏi cụm hardening để đọc
sau khi bạn đã biết scheduler làm gì (giai đoạn 7) và đã thấy các điểm mở rộng bị dùng thế nào
trong nhóm PodGroup vừa đọc. Nếu bạn nhảy thẳng vào giai đoạn 13 mà chưa qua giai đoạn 7 và 9,
hãy đọc [bài 137](137-kube-scheduler-vi.md) và
[bài 147](147-scheduling-framework-vi.md) trước.

**Phải hiểu ở lần đọc này:**

- Vì sao scheduler là vấn đề **bảo mật** chứ không chỉ là hiệu năng: một scheduler bị cấu hình
  sai có thể **nhắm vào node cụ thể và trục xuất workload đang chia sẻ node đó**, và hỗ trợ
  tấn công Yo-Yo nhắm vào autoscaler.
- Nhóm cờ xác thực/phân quyền: đặt `authentication-tolerate-lookup-failure` và
  `authentication-skip-lookup` thành `false` để scheduler **luôn** tra cứu cấu hình xác thực
  từ API server; bảo vệ file `authentication-kubeconfig` bằng quyền truy cập file nghiêm ngặt;
  tránh truyền `requestheader-client-ca-file`; tắt profiling (nay qua `enableProfiling` trong
  cấu hình kube-scheduler).
- Nhóm cờ mạng và TLS: `bind-address` nên là `localhost` vì kube-scheduler thường **không cần
  truy cập từ bên ngoài**; `permit-address-sharing` đặt `false`; giữ `permit-port-sharing` ở
  mặc định `false`; luôn khai `tls-cipher-suites` tường minh.
- Bốn điểm mở rộng cần soi kỹ khi chạy scheduler tùy chỉnh và lý do của từng cái: `queueSort`
  (**chỉ một plugin được bật tại một thời điểm**), `prefilter`/`filter` (**có thể đánh dấu tất
  cả node là unschedulable**, làm đình trệ hoàn toàn việc lập lịch Pod mới), `permit` (**ngăn
  hoặc trì hoãn việc bind Pod**).
- Mục cuối, ngắn nhất mà thực dụng nhất: **không cho người dùng cluster gán nhãn node**, vì kẻ
  xấu chỉ cần `nodeSelector` là đẩy được workload lên node lẽ ra không được chạm tới.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Đoạn YAML tắt plugin theo từng điểm mở rộng | chỉ dùng khi thực sự chạy scheduler tùy chỉnh | khi công việc thực sự cần |
| Chi tiết từng điểm mở rộng của scheduling framework | nền đã học | giai đoạn 7 — [bài 147](147-scheduling-framework-vi.md) |
| Tấn công Yo-Yo và cơ chế autoscaler bị lợi dụng | nằm ngoài phạm vi bài | giai đoạn 12 — [bài 171](171-node-autoscaling-vi.md) |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Vì sao bài yêu cầu đặt `authentication-skip-lookup` và `authentication-tolerate-lookup-failure`
   thành `false`? Điều gì hỏng nếu để scheduler tự xoay xở khi không tra cứu được?
2. Trong bốn điểm mở rộng mà bài cảnh báo, điểm nào chỉ cho phép **đúng một** plugin được bật,
   và vì sao một plugin `filter` viết ẩu lại nguy hiểm hơn vẻ ngoài của nó?
3. Trên cluster lab của bạn, người dùng thường không có quyền gán nhãn node. Vì sao đó là biện
   pháp **bảo mật**, chứ không chỉ là giữ cho cluster gọn gàng?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Để **cơ chế xác thực của kube-scheduler luôn nhất quán với kube-apiserver**. Bài nêu nguyên
   tắc: nếu một yêu cầu thiếu header xác thực thì việc xác thực nên được thực hiện thông qua
   kube-apiserver, để mọi hoạt động xác thực trong cluster nhất quán. Đặt hai cờ đó thành
   `false` buộc scheduler **luôn luôn** tra cứu cấu hình xác thực của nó từ API server, thay vì
   bỏ qua bước tra cứu hoặc âm thầm chấp nhận khi tra cứu thất bại — hai trường hợp đều tạo ra
   một thành phần control plane xác thực theo luật riêng.
2. **`queueSort`** — tại một thời điểm chỉ có thể bật **đúng một** plugin dùng điểm mở rộng
   này, nên bất kỳ plugin nào đụng tới nó đều phải xem xét kỹ. Còn plugin `prefilter` hoặc
   `filter`: chúng **có khả năng đánh dấu tất cả các node là unschedulable**, và hệ quả không
   phải là "một Pod bị chậm" mà là **việc lập lịch cho các Pod mới bị đình trệ hoàn toàn**.
   Một lỗi trong hàm lọc trông như lỗi cục bộ nhưng thực chất là mất khả năng lập lịch toàn
   cluster. Điểm mở rộng `permit` nguy hiểm theo cách khác: nó **ngăn hoặc trì hoãn việc bind**
   một Pod.
3. Vì nhãn node là **đầu vào của quyết định lập lịch**. Ai gán được nhãn thì dùng
   `nodeSelector` để **đưa workload lên đúng những node mà workload đó không nên xuất hiện** —
   ví dụ node control plane hoặc node dành cho workload nhạy cảm. Đây là đường vòng: kẻ tấn
   công không cần phá scheduler, chỉ cần nói dối scheduler về node. Cùng logic đó, bài coi
   scheduler bị cấu hình sai là rủi ro bảo mật vì nó có thể **nhắm vào node cụ thể và trục xuất
   các workload đang chia sẻ node và tài nguyên của node đó**.

</details>

Hết giai đoạn 13. Câu nào chưa trả lời được thì quay lại đúng mục tương ứng.
[Lab 13 — DRA](labs/LAB-13-DRA.md) là lab **tùy chọn**: nó kiểm năng lực DRA của cluster
bằng năm phép đo rồi mới rẽ nhánh, và chỉ chạy được nhánh thực hành nếu bạn có GPU hoặc thiết
bị chuyên dụng. Nếu cluster lab của bạn không có, checkpoint của giai
đoạn chỉ yêu cầu một điều: giải thích được DRA khác device plugin truyền thống ở điểm nào —
xem lại [bài 149](149-dynamic-resource-allocation-vi.md) nếu còn lấn cấn.

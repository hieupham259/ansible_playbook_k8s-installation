# Kiểm soát các chính sách quản lý CPU trên Node (Control CPU Management Policies on the Node)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/>
>
> Trang này hướng dẫn cách cấu hình và thay đổi chính sách của CPU Manager trong kubelet
> để cấp CPU độc quyền cho những workload nhạy cảm với độ trễ.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 25 — Quản trị tài nguyên theo namespace](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace),
bài 4/7, nối tiếp bài [74 — Các trình quản lý tài nguyên](74-resource-managers-vi.md). Các
trang CP không có lab riêng: thực hành trực tiếp trên cluster lab, và fault injection (đổi
chính sách sai quy trình để xem kubelet crashloop) chỉ làm trên `lab-k8s-worker2`.

Lý thuyết về CPU Manager, shared pool và điều kiện cấp CPU độc quyền bạn đã đọc kỹ ở bài
74; trang này là **runbook thao tác**: đổi chính sách trên một node đang chạy sao cho đúng
quy trình. Giá trị lớn nhất của bài nằm ở mục *Thay đổi chính sách CPU Manager*.

**Phải hiểu ở lần đọc này:**

- Hai chính sách `none` (mặc định) và `static`, đặt bằng flag `--cpu-manager-policy` hoặc
  field `cpuManagerPolicy` trong KubeletConfiguration (mục *Cấu hình*).
- Quy trình 5 bước bắt buộc khi đổi chính sách trên một node: drain node → dừng kubelet →
  xóa file state `/var/lib/kubelet/cpu_manager_state` → sửa cấu hình → khởi động lại
  kubelet; bỏ bước xóa file state thì kubelet crashloop với lỗi
  `could not restore state from checkpoint` (mục *Thay đổi chính sách CPU Manager*).
- Vì sao chính sách `static` bắt buộc dự trữ CPU lớn hơn 0 qua `--kube-reserved`,
  `--system-reserved` hoặc `--reserved-cpus`: nếu không, shared pool có thể trở nên rỗng
  (mục *Cấu hình chính sách `static`*).
- Container nào được cấp CPU độc quyền: thuộc Pod `Guaranteed` **và** có CPU `requests` là
  số nguyên; mọi container khác (kể cả `Guaranteed` với CPU lẻ) chạy trong shared pool.
- Các tùy chọn tinh chỉnh chính sách static bị khóa sau ba feature gate:
  `CPUManagerPolicyOptions` (tổng), `CPUManagerPolicyAlphaOptions` và
  `CPUManagerPolicyBetaOptions` (theo độ chín) — gate theo **nhóm**, không theo từng tùy
  chọn.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
|---|---|---|
| *Hỗ trợ Windows* | Cluster lab toàn node Linux; tính năng còn alpha và cần container runtime hỗ trợ | Chỉ khi vận hành node Windows — ngoài phạm vi lộ trình |
| Chi tiết hành vi từng tùy chọn static (`full-pcpus-only`, `distribute-cpus-across-numa`, `align-by-socket`, …) | Cần phần cứng nhiều NUMA node/socket mới thấy khác biệt; trang này cũng chỉ liệt kê rồi trỏ về bài 74 | Bài [74](74-resource-managers-vi.md#cpu-policy-static--options), mục *Các tùy chọn của chính sách static*, khi làm việc với phần cứng thật |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

Kubernetes che giấu khỏi người dùng nhiều khía cạnh về cách Pod thực thi trên node. Đây là
chủ ý thiết kế. Tuy nhiên, một số workload đòi hỏi những bảo đảm chặt chẽ hơn về độ trễ
và/hoặc hiệu năng để có thể hoạt động ở mức chấp nhận được. kubelet cung cấp các phương
thức để bật những chính sách sắp xếp (placement) workload phức tạp hơn, trong khi vẫn giữ
lớp trừu tượng không chứa các chỉ thị sắp xếp tường minh.

Để biết thông tin chi tiết về quản lý tài nguyên, hãy tham khảo tài liệu
[Quản lý tài nguyên cho Pod và Container](110-manage-resources-containers-vi.md).

Để biết thông tin chi tiết về cách kubelet hiện thực việc quản lý tài nguyên, hãy tham khảo
tài liệu [Node ResourceManagers](74-resource-managers-vi.md).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một
trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.26 hoặc mới hơn. Để kiểm tra phiên bản, hãy
chạy `kubectl version`.

Nếu bạn đang chạy một phiên bản Kubernetes cũ hơn, hãy xem tài liệu của đúng phiên bản mà
bạn đang thực sự chạy.

## Cấu hình các chính sách quản lý CPU (Configuring CPU management policies)

Theo mặc định, kubelet dùng [CFS quota](https://en.wikipedia.org/wiki/Completely_Fair_Scheduler)
để cưỡng chế giới hạn CPU của Pod. Khi node chạy nhiều Pod thiên về CPU (CPU-bound),
workload có thể bị chuyển sang các CPU core khác nhau tùy vào việc Pod có bị bóp
(throttle) hay không và những CPU core nào đang rảnh tại thời điểm lập lịch. Nhiều
workload không nhạy cảm với sự di chuyển này nên vẫn hoạt động tốt mà không cần can thiệp
gì.

Tuy nhiên, với những workload mà tính gắn kết cache CPU (CPU cache affinity) và độ trễ lập
lịch ảnh hưởng đáng kể đến hiệu năng, kubelet cho phép dùng các chính sách quản lý CPU
thay thế để quyết định một số ưu tiên sắp xếp trên node.

## Hỗ trợ Windows (Windows Support)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [alpha]`

Hỗ trợ CPU Manager có thể được bật trên Windows bằng feature gate
`WindowsCPUAndMemoryAffinity`, và nó đòi hỏi container runtime phải hỗ trợ. Sau khi bật
feature gate, hãy làm theo các bước bên dưới để cấu hình
[chính sách CPU Manager](#configuration).

## Cấu hình (Configuration) {#configuration}

Chính sách CPU Manager được đặt bằng flag `--cpu-manager-policy` của kubelet hoặc field
`cpuManagerPolicy` trong
[KubeletConfiguration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).
Có hai chính sách được hỗ trợ:

* [`none`](#none-policy): chính sách mặc định.
* [`static`](#static-policy): cho phép các Pod có những đặc điểm tài nguyên nhất định được
  cấp mức gắn kết CPU (CPU affinity) và tính độc quyền cao hơn trên node.

CPU Manager định kỳ ghi các cập nhật tài nguyên thông qua CRI để đối soát (reconcile) các
phép gán CPU trong bộ nhớ với cgroupfs. Tần suất đối soát được đặt qua một giá trị cấu
hình mới của kubelet là `--cpu-manager-reconcile-period`. Nếu không chỉ định, nó mặc định
bằng đúng chu kỳ của `--node-status-update-frequency`.

Hành vi của chính sách static có thể được tinh chỉnh bằng flag
`--cpu-manager-policy-options`. Flag này nhận một danh sách các tùy chọn chính sách dạng
`key=value`, phân tách bằng dấu phẩy. Nếu bạn tắt
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`CPUManagerPolicyOptions` thì bạn không thể tinh chỉnh các chính sách CPU Manager. Trong
trường hợp đó, CPU Manager chỉ hoạt động với các thiết lập mặc định của nó.

Bên cạnh feature gate tổng `CPUManagerPolicyOptions`, các tùy chọn chính sách được chia
thành hai nhóm: chất lượng alpha (ẩn theo mặc định) và chất lượng beta (hiện theo mặc
định). Hai nhóm này lần lượt được bảo vệ bởi các feature gate
`CPUManagerPolicyAlphaOptions` và `CPUManagerPolicyBetaOptions`. Khác với chuẩn thông
thường của Kubernetes, các feature gate này bảo vệ cả **nhóm** tùy chọn, bởi vì việc thêm
một feature gate cho từng tùy chọn riêng lẻ sẽ quá cồng kềnh.

## Thay đổi chính sách CPU Manager (Changing the CPU Manager Policy)

Vì chính sách CPU Manager chỉ có thể được áp dụng khi kubelet khởi tạo Pod mới, việc chỉ
đổi từ "none" sang "static" sẽ không có tác dụng với các Pod hiện có. Do đó, để thay đổi
chính sách CPU Manager trên một node một cách đúng đắn, hãy thực hiện các bước sau:

1. [Drain](255-safely-drain-node-vi.md) node đó.
2. Dừng kubelet.
3. Xóa file state cũ của CPU Manager. Đường dẫn của file này mặc định là
   `/var/lib/kubelet/cpu_manager_state`. Việc này xóa sạch trạng thái mà CPUManager duy
   trì, để các cpu-set do chính sách mới thiết lập không xung đột với nó.
4. Sửa cấu hình kubelet để đổi chính sách CPU Manager sang giá trị mong muốn.
5. Khởi động kubelet.

Lặp lại quy trình này cho từng node cần thay đổi chính sách CPU Manager. Bỏ qua quy trình
này sẽ khiến kubelet crashloop với lỗi sau:

```
could not restore state from checkpoint: configured policy "static" differs from state checkpoint policy "none", please drain this node and delete the CPU manager checkpoint file "/var/lib/kubelet/cpu_manager_state" before restarting Kubelet
```

> **Ghi chú:**
> Nếu tập hợp CPU đang online trên node thay đổi, node phải được drain và CPU Manager phải
> được reset thủ công bằng cách xóa file state `cpu_manager_state` trong thư mục gốc của
> kubelet.

### Cấu hình chính sách `none` (`none` policy configuration) {#none-policy}

Chính sách này không có mục cấu hình bổ sung nào.

### Cấu hình chính sách `static` (`static` policy configuration) {#static-policy}

Chính sách này quản lý một pool CPU dùng chung (shared pool) mà ban đầu chứa toàn bộ CPU
của node. Số lượng CPU có thể cấp phát độc quyền bằng tổng số CPU trên node trừ đi phần
CPU mà kubelet dự trữ qua các tùy chọn `--kube-reserved` hoặc `--system-reserved`. Từ
phiên bản 1.17, danh sách CPU dự trữ có thể được chỉ định tường minh bằng tùy chọn
`--reserved-cpus` của kubelet. Danh sách CPU tường minh do `--reserved-cpus` chỉ định được
ưu tiên hơn phần dự trữ CPU do `--kube-reserved` và `--system-reserved` chỉ định. Các CPU
được dự trữ bởi những tùy chọn này được lấy — theo số lượng nguyên — từ shared pool ban
đầu, theo thứ tự tăng dần của ID physical core. Shared pool này là tập các CPU mà mọi
container trong các Pod `BestEffort` và `Burstable` chạy trên đó. Các container trong Pod
`Guaranteed` có CPU `requests` là số lẻ (fractional) cũng chạy trên các CPU thuộc shared
pool. Chỉ những container vừa thuộc một Pod `Guaranteed` vừa có CPU `requests` là số
nguyên mới được gán CPU độc quyền.

> **Ghi chú:**
> kubelet yêu cầu phải có một phần dự trữ CPU lớn hơn 0, đặt qua `--kube-reserved` và/hoặc
> `--system-reserved` hoặc `--reserved-cpus`, khi chính sách static được bật. Lý do là một
> phần dự trữ CPU bằng 0 sẽ có thể khiến shared pool trở nên rỗng.

### Các tùy chọn của chính sách static (Static policy options) {#cpu-policy-static--options}

Bạn có thể bật/tắt các nhóm tùy chọn theo mức độ chín (maturity level) của chúng bằng các
feature gate sau:

* `CPUManagerPolicyBetaOptions` bật theo mặc định. Tắt gate này để ẩn các tùy chọn mức beta.
* `CPUManagerPolicyAlphaOptions` tắt theo mặc định. Bật gate này để hiện các tùy chọn mức
  alpha.

Bạn vẫn phải bật từng tùy chọn cụ thể bằng tùy chọn `CPUManagerPolicyOptions` của kubelet.

Các tùy chọn chính sách sau tồn tại cho chính sách static của `CPUManager`:

* `full-pcpus-only` (GA, hiện theo mặc định) (1.33 trở lên)
* `distribute-cpus-across-numa` (beta, hiện theo mặc định) (1.33 trở lên)
* `align-by-socket` (alpha, ẩn theo mặc định) (1.25 trở lên)
* `distribute-cpus-across-cores` (alpha, ẩn theo mặc định) (1.31 trở lên)
* `strict-cpu-reservation` (GA, hiện theo mặc định) (1.35 trở lên)
* `prefer-align-cpus-by-uncorecache` (GA, hiện theo mặc định) (1.36 trở lên)

Tùy chọn `full-pcpus-only` có thể được bật bằng cách thêm `full-pcpus-only=true` vào các
tùy chọn chính sách của CPUManager. Tương tự, tùy chọn `distribute-cpus-across-numa` có
thể được bật bằng cách thêm `distribute-cpus-across-numa=true` vào các tùy chọn chính sách
của CPUManager. Khi cả hai cùng được đặt, chúng có tính "cộng gộp" (additive) theo nghĩa:
CPU sẽ được phân bổ trên các NUMA node theo từng khối physical CPU trọn vẹn (full-pcpus)
thay vì từng core riêng lẻ. Tùy chọn chính sách `align-by-socket` có thể được bật bằng
cách thêm `align-by-socket=true` vào các tùy chọn chính sách của `CPUManager`. Nó cũng có
tính cộng gộp với các tùy chọn chính sách `full-pcpus-only` và
`distribute-cpus-across-numa`.

Tùy chọn `distribute-cpus-across-cores` có thể được bật bằng cách thêm
`distribute-cpus-across-cores=true` vào các tùy chọn chính sách của `CPUManager`. Hiện
tại, nó không thể được dùng cùng với các tùy chọn chính sách `full-pcpus-only` hoặc
`distribute-cpus-across-numa`.

Tùy chọn `strict-cpu-reservation` có thể được bật bằng cách thêm
`strict-cpu-reservation=true` vào các tùy chọn chính sách của CPUManager, sau đó xóa file
`/var/lib/kubelet/cpu_manager_state` và khởi động lại kubelet.

Tùy chọn `prefer-align-cpus-by-uncorecache` có thể được bật bằng cách thêm
`prefer-align-cpus-by-uncorecache` vào các tùy chọn chính sách của `CPUManager`. Nếu các
tùy chọn không tương thích được dùng cùng nhau, kubelet sẽ không khởi động được, kèm lỗi
được giải thích trong log.

Để biết thêm chi tiết về hành vi của từng tùy chọn mà bạn có thể cấu hình, hãy tham khảo
tài liệu [Node ResourceManagers](74-resource-managers-vi.md).

## Tiếp theo (What's next)

* Đọc về [các trình quản lý tài nguyên cấp Pod (Pod-level resource managers)](74-resource-managers-vi.md#pod-level-resource-managers).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở checkpoint giai đoạn 25:

1. Trên `lab-k8s-worker2` của bạn, bạn sửa `cpuManagerPolicy` từ `none` thành `static` trong
   file cấu hình kubelet rồi restart kubelet ngay, không làm gì thêm. Chuyện gì xảy ra và
   vì sao?
2. Vì sao kubelet từ chối chạy chính sách `static` nếu phần dự trữ CPU
   (`--kube-reserved`/`--system-reserved`/`--reserved-cpus`) bằng 0?
3. Câu bẫy: một Pod thuộc lớp QoS `Guaranteed` với `requests` CPU là `1500m` — dưới chính
   sách `static`, container của nó có được cấp CPU độc quyền không?
4. Khi cả `--reserved-cpus` lẫn `--kube-reserved`/`--system-reserved` cùng được đặt, danh
   sách CPU dự trữ thực tế lấy theo cái nào?
5. Kubelet dựa vào giá trị cấu hình nào để quyết định tần suất đối soát các phép gán CPU
   trong bộ nhớ với cgroupfs, và mặc định của nó là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **kubelet crashloop** với lỗi `could not restore state from checkpoint: configured
   policy "static" differs from state checkpoint policy "none"...`. Vì file state
   `/var/lib/kubelet/cpu_manager_state` vẫn ghi chính sách cũ, xung đột với chính sách mới.
   Quy trình đúng gồm 5 bước: drain node → dừng kubelet → xóa file state → sửa cấu hình →
   khởi động lại kubelet.
2. Vì **dự trữ bằng 0 có thể khiến shared pool trở nên rỗng**. Shared pool là nơi mọi
   container `BestEffort`, `Burstable` (và cả `Guaranteed` với CPU lẻ) phải chạy; nếu toàn
   bộ CPU bị cấp phát độc quyền hết thì các container đó không còn chỗ chạy.
3. **Không.** Điều kiện là "và" kép: thuộc Pod `Guaranteed` **và** CPU `requests` là **số
   nguyên**. `1500m` là số lẻ (fractional) nên container này chạy trong shared pool như
   thường. Trực giác "cứ Guaranteed là được độc quyền" là sai — lớp QoS mới chỉ là một nửa
   điều kiện.
4. Theo **`--reserved-cpus`**: danh sách CPU tường minh do `--reserved-cpus` chỉ định được
   ưu tiên hơn phần dự trữ tính từ `--kube-reserved` và `--system-reserved`.
5. Giá trị **`--cpu-manager-reconcile-period`**; nếu không chỉ định, nó mặc định bằng đúng
   chu kỳ của **`--node-status-update-frequency`**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

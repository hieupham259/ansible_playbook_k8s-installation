# Kiểm toán (Auditing)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối — nhánh `/docs/tasks/`](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 22 — Audit và mã hóa dữ liệu](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu), bài 1/6 ·
thực hành trực tiếp trên `k8s-master` của cluster VM [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

Bài này lấp mảnh cuối của chuỗi xử lý request đã học ở bài
[119 — Kiểm soát truy cập vào Kubernetes API](119-controlling-access-vi.md): sau authentication →
authorization → admission, audit là lớp ghi vết trả lời câu hỏi "ai đã làm gì, lúc nào, trên cái
gì" — thứ bạn cần khi điều tra sự cố bảo mật.

**Phải hiểu ở lần đọc này:**

- Audit event sinh ra bên trong `kube-apiserver`: mỗi request, ở mỗi stage thực thi, sinh một
  event; **policy quyết định ghi cái gì**, **backend quyết định lưu vào đâu**. Bốn stage:
  `RequestReceived`, `ResponseStarted` (chỉ với request chạy dài như watch), `ResponseComplete`,
  `Panic`.
- Bốn audit level (`None`, `Metadata`, `Request`, `RequestResponse`) và quy tắc so khớp: event
  được so với danh sách rule **theo thứ tự, rule khớp đầu tiên quyết định level**. Không có flag
  `--audit-policy-file` thì không event nào được ghi; policy 0 rule là bất hợp lệ; resource
  `pods` không khớp các subresource như `pods/log`.
- Log backend: ghi JSONlines vào file qua `--audit-log-path` (kèm `maxage`/`maxbackup`/`maxsize`
  để xoay vòng), và trên cluster kubeadm phải mount `hostPath` cho **cả** file policy lẫn thư mục
  log vào static Pod `kube-apiserver` thì bản ghi mới bền vững.
- Webhook backend: gửi event tới một HTTP API bên ngoài, file cấu hình theo định dạng kubeconfig
  (`--audit-webhook-config-file`); khác biệt mặc định: batching **bật** với `webhook`, **tắt**
  với `log`; ba mode buffer là `batch`, `blocking`, `blocking-strict`.
- Giám sát hệ audit bằng hai metric `apiserver_audit_event_total` và
  `apiserver_audit_error_total`; bật audit làm tăng tiêu thụ bộ nhớ của API server.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Danh sách flag tinh chỉnh batch/throttle và mục "Tinh chỉnh tham số" | chỉ cần khi cluster tải cao tới mức backend ghi không kịp; cluster lab không tới mức đó | tra cứu lại mục [Gom event theo lô](#batching) của chính bài này khi vận hành cluster tải lớn |
| Mục "Cắt bớt entry log" (`audit-log-truncate-*`) | tối ưu dung lượng lưu trữ, không thay đổi bản chất cơ chế policy → backend | cùng thời điểm với hàng trên, khi tinh chỉnh audit cho production |
| Link "Mutating webhook auditing annotations" ở mục Tiếp theo | chi tiết annotation mà admission webhook ghi thêm vào audit event | quay lại bài [173 — Thực hành tốt cho Admission Webhook](173-admission-webhooks-vi.md) (giai đoạn 9) khi cần lần vết webhook qua audit log |

---

_Kiểm toán_ (auditing) trong Kubernetes cung cấp một tập bản ghi theo trình tự thời gian, có ý
nghĩa đối với bảo mật, ghi lại chuỗi hành động diễn ra trong một cluster. Cluster kiểm toán các
hoạt động do người dùng sinh ra, do các ứng dụng sử dụng Kubernetes API, và do chính control
plane sinh ra.

Kiểm toán cho phép quản trị viên cluster trả lời các câu hỏi sau:

- chuyện gì đã xảy ra?
- nó xảy ra khi nào?
- ai là người khởi phát?
- nó xảy ra trên đối tượng nào?
- nó được quan sát thấy ở đâu?
- nó được khởi phát từ đâu?
- nó đang đi tới đâu?

Bản ghi kiểm toán (audit record) bắt đầu vòng đời của mình bên trong thành phần
[kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/).
Mỗi request, ở mỗi stage (giai đoạn) thực thi của nó, sinh ra một audit event; event này sau đó
được tiền xử lý theo một chính sách (policy) nhất định và được ghi vào một backend. Policy quyết
định những gì được ghi lại, còn backend lưu trữ bền vững các bản ghi. Các hiện thực backend hiện
có bao gồm file log và webhook.

Mỗi request có thể được ghi lại kèm một _stage_ tương ứng. Các stage được định nghĩa gồm:

- `RequestReceived` - Stage cho các event được sinh ra ngay khi audit handler nhận được request,
  và trước khi request được chuyển tiếp xuống chuỗi handler phía sau.
- `ResponseStarted` - Khi các header của response đã được gửi đi, nhưng trước khi body của
  response được gửi. Stage này chỉ được sinh ra cho các request chạy dài (ví dụ: watch).
- `ResponseComplete` - Body của response đã hoàn tất và không còn byte nào được gửi thêm.
- `Panic` - Các event được sinh ra khi xảy ra panic.

> **Ghi chú:**
> Cấu hình của một
> [Audit Event](https://kubernetes.io/docs/reference/config-api/apiserver-audit.v1/#audit-k8s-io-v1-Event)
> khác với API object
> [Event](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#event-v1-core).

Tính năng ghi log kiểm toán làm tăng mức tiêu thụ bộ nhớ của API server, vì một số ngữ cảnh cần
cho việc kiểm toán được lưu lại cho mỗi request. Mức tiêu thụ bộ nhớ phụ thuộc vào cấu hình ghi
log kiểm toán.

## Chính sách kiểm toán (Audit policy)

Audit policy định nghĩa các rule về việc event nào cần được ghi lại và chúng cần bao gồm những
dữ liệu gì. Cấu trúc của đối tượng audit policy được định nghĩa trong
[nhóm API `audit.k8s.io`](https://kubernetes.io/docs/reference/config-api/apiserver-audit.v1/#audit-k8s-io-v1-Policy).
Khi một event được xử lý, nó được so sánh lần lượt với danh sách rule theo thứ tự. Rule khớp
đầu tiên quyết định _audit level_ (mức kiểm toán) của event. Các audit level được định nghĩa
gồm:

- `None` - không ghi log các event khớp rule này.
- `Metadata` - ghi log event kèm metadata (người dùng gửi request, timestamp, resource, verb,
  v.v.) nhưng không ghi body của request hay response.
- `Request` - ghi log event kèm metadata và body của request nhưng không ghi body của response.
  Mức này không áp dụng cho các request non-resource.
- `RequestResponse` - ghi log event kèm metadata của request, body của request và body của
  response. Mức này không áp dụng cho các request non-resource.

Bạn có thể truyền một file chứa policy cho `kube-apiserver` bằng flag `--audit-policy-file`.
Nếu flag này bị bỏ qua, không event nào được ghi log. Lưu ý rằng trường `rules` **bắt buộc**
phải có trong file audit policy. Một policy không có rule nào (0 rule) bị coi là bất hợp lệ.

Dưới đây là một ví dụ file audit policy:

```yaml
apiVersion: audit.k8s.io/v1 # Trường này là bắt buộc.
kind: Policy
# Không sinh audit event cho mọi request ở stage RequestReceived.
omitStages:
  - "RequestReceived"
rules:
  # Ghi log các thay đổi Pod ở mức RequestResponse
  - level: RequestResponse
    resources:
    - group: ""
      # Resource "pods" không khớp các request tới bất kỳ subresource nào của pods,
      # điều này nhất quán với chính sách RBAC.
      resources: ["pods"]
  # Ghi log "pods/log", "pods/status" ở mức Metadata
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # Không ghi log các request tới ConfigMap có tên "controller-leader"
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # Không ghi log các request watch của "system:kube-proxy" trên endpoints hoặc services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # nhóm API core
      resources: ["endpoints", "services"]

  # Không ghi log các request đã xác thực tới một số đường dẫn URL non-resource.
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # So khớp wildcard.
    - "/version"

  # Ghi log body của request đối với các thay đổi ConfigMap trong kube-system.
  - level: Request
    resources:
    - group: "" # nhóm API core
      resources: ["configmaps"]
    # Rule này chỉ áp dụng cho các resource trong namespace "kube-system".
    # Chuỗi rỗng "" có thể được dùng để chọn các resource không thuộc namespace nào.
    namespaces: ["kube-system"]

  # Ghi log các thay đổi ConfigMap và Secret trong mọi namespace khác ở mức Metadata.
  - level: Metadata
    resources:
    - group: "" # nhóm API core
      resources: ["secrets", "configmaps"]

  # Ghi log tất cả các resource khác trong core và extensions ở mức Request.
  - level: Request
    resources:
    - group: "" # nhóm API core
    - group: "extensions" # KHÔNG được ghi kèm version của group.

  # Rule vét (catch-all) để ghi log mọi request khác ở mức Metadata.
  - level: Metadata
    # Các request chạy dài như watch rơi vào rule này sẽ không
    # sinh audit event ở stage RequestReceived.
    omitStages:
      - "RequestReceived"
```

Bạn có thể dùng một file audit policy tối giản để ghi log mọi request ở mức `Metadata`:

```yaml
# Ghi log mọi request ở mức Metadata.
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
```

Nếu bạn đang tự xây dựng audit profile của riêng mình, bạn có thể dùng audit profile của Google
Container-Optimized OS làm điểm khởi đầu. Bạn có thể xem script
[configure-helper.sh](https://github.com/kubernetes/kubernetes/blob/master/cluster/gce/gci/configure-helper.sh),
script này sinh ra một file audit policy. Bạn có thể thấy phần lớn nội dung file audit policy
bằng cách nhìn trực tiếp vào script.

Bạn cũng có thể tham khảo
[tài liệu tham chiếu cấu hình `Policy`](https://kubernetes.io/docs/reference/config-api/apiserver-audit.v1/#audit-k8s-io-v1-Policy)
để biết chi tiết về các trường được định nghĩa.

## Backend kiểm toán (Audit backends)

Audit backend lưu trữ bền vững các audit event ra một kho lưu trữ bên ngoài. Mặc định,
kube-apiserver cung cấp hai backend:

- Log backend, ghi các event vào hệ thống file
- Webhook backend, gửi các event tới một HTTP API bên ngoài

Trong mọi trường hợp, audit event tuân theo cấu trúc được Kubernetes API định nghĩa trong
[nhóm API `audit.k8s.io`](https://kubernetes.io/docs/reference/config-api/apiserver-audit.v1/#audit-k8s-io-v1-Event).

> **Ghi chú:**
> Trong trường hợp patch, body của request là một mảng JSON chứa các thao tác patch, chứ không
> phải một object JSON chứa API object Kubernetes tương ứng. Ví dụ, body request sau là một
> patch request hợp lệ tới `/apis/batch/v1/namespaces/some-namespace/jobs/some-job-name`:
>
> ```json
> [
>   {
>     "op": "replace",
>     "path": "/spec/parallelism",
>     "value": 0
>   },
>   {
>     "op": "remove",
>     "path": "/spec/template/spec/containers/0/terminationMessagePolicy"
>   }
> ]
> ```

### Log backend

Log backend ghi các audit event vào một file theo định dạng [JSONlines](https://jsonlines.org/).
Bạn có thể cấu hình log audit backend bằng các flag sau của `kube-apiserver`:

- `--audit-log-path` chỉ định đường dẫn file log mà log backend dùng để ghi các audit event.
  Không chỉ định flag này sẽ tắt log backend. `-` nghĩa là standard out
- `--audit-log-maxage` định nghĩa số ngày tối đa lưu giữ các file audit log cũ
- `--audit-log-maxbackup` định nghĩa số file audit log tối đa được lưu giữ
- `--audit-log-maxsize` định nghĩa kích thước tối đa tính bằng megabyte của file audit log
  trước khi nó được xoay vòng (rotate)

Nếu control plane của cluster chạy kube-apiserver dưới dạng một Pod, hãy nhớ mount `hostPath`
tới vị trí của file policy và file log, để các bản ghi kiểm toán được lưu bền vững. Ví dụ:

```yaml
  - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
  - --audit-log-path=/var/log/kubernetes/audit/audit.log
```

sau đó mount các volume:

```yaml
...
volumeMounts:
  - mountPath: /etc/kubernetes/audit-policy.yaml
    name: audit
    readOnly: true
  - mountPath: /var/log/kubernetes/audit/
    name: audit-log
    readOnly: false
```

và cuối cùng cấu hình `hostPath`:

```yaml
...
volumes:
- name: audit
  hostPath:
    path: /etc/kubernetes/audit-policy.yaml
    type: File

- name: audit-log
  hostPath:
    path: /var/log/kubernetes/audit/
    type: DirectoryOrCreate
```

### Webhook backend

Webhook audit backend gửi các audit event tới một web API từ xa, API này được giả định là một
dạng của Kubernetes API, bao gồm cả phương thức xác thực. Bạn có thể cấu hình webhook audit
backend bằng các flag sau của kube-apiserver:

- `--audit-webhook-config-file` chỉ định đường dẫn tới file chứa cấu hình webhook. Cấu hình
  webhook thực chất là một
  [kubeconfig](361-configure-access-multiple-clusters-vi.md)
  chuyên biệt.
- `--audit-webhook-initial-backoff` chỉ định khoảng thời gian chờ sau request thất bại đầu tiên
  trước khi thử lại. Các request tiếp theo được thử lại theo cơ chế exponential backoff (lùi
  theo cấp số nhân).

File cấu hình webhook dùng định dạng kubeconfig để chỉ định địa chỉ từ xa của dịch vụ và thông
tin xác thực (credentials) dùng để kết nối tới nó.

## Gom event theo lô (Event batching) {#batching}

Cả hai backend `log` và `webhook` đều hỗ trợ gom theo lô (batching). Dưới đây là danh sách các
flag khả dụng riêng cho từng backend. Theo mặc định, batching và throttling (tiết lưu) được
**bật** cho backend `webhook` và **tắt** cho backend `log`.

#### webhook

- `--audit-webhook-mode` định nghĩa chiến lược buffer. Một trong các giá trị sau:
  - `batch` - buffer các event và xử lý bất đồng bộ theo lô. Đây là mode mặc định cho backend
    `webhook`.
  - `blocking` - chặn các response của API server trong khi xử lý từng event riêng lẻ.
  - `blocking-strict` - Giống blocking, nhưng khi xảy ra lỗi trong quá trình ghi audit log ở
    stage RequestReceived, toàn bộ request tới kube-apiserver sẽ thất bại.

Các flag sau chỉ được dùng trong mode `batch`:

- `--audit-webhook-batch-buffer-size` định nghĩa số event được buffer trước khi gom lô. Nếu tốc
  độ event đến làm tràn buffer, các event sẽ bị loại bỏ. Giá trị mặc định là 10000.
- `--audit-webhook-batch-max-size` định nghĩa số event tối đa trong một lô. Giá trị mặc định
  là 400.
- `--audit-webhook-batch-max-wait` định nghĩa thời gian chờ tối đa trước khi gom lô vô điều
  kiện các event trong hàng đợi. Giá trị mặc định là 30 giây.
- `--audit-webhook-batch-throttle-enable` định nghĩa việc throttling khi gom lô có được bật hay
  không. Throttling được bật theo mặc định.
- `--audit-webhook-batch-throttle-qps` định nghĩa số lô trung bình tối đa được sinh ra mỗi
  giây. Giá trị mặc định là 10.
- `--audit-webhook-batch-throttle-burst` định nghĩa số lô tối đa được sinh ra tại cùng một thời
  điểm nếu lượng QPS cho phép trước đó chưa được dùng hết. Giá trị mặc định là 15.

#### log

- `--audit-log-mode` định nghĩa chiến lược buffer. Một trong các giá trị sau:
  - `batch` - buffer các event và xử lý bất đồng bộ theo lô. Batching không được khuyến nghị
    cho backend `log`.
  - `blocking` - chặn các response của API server trong khi xử lý từng event riêng lẻ. Đây là
    mode mặc định cho backend `log`.
  - `blocking-strict` - Giống blocking, nhưng khi xảy ra lỗi trong quá trình ghi audit log ở
    stage RequestReceived, toàn bộ request tới kube-apiserver sẽ thất bại.

Các flag sau chỉ được dùng trong mode `batch` (batching bị **tắt** theo mặc định cho backend
`log`, và khi batching bị tắt, mọi flag liên quan tới batching đều bị bỏ qua):

- `--audit-log-batch-buffer-size` định nghĩa số event được buffer trước khi gom lô. Nếu tốc độ
  event đến làm tràn buffer, các event sẽ bị loại bỏ.
- `--audit-log-batch-max-size` định nghĩa số event tối đa trong một lô.
- `--audit-log-batch-max-wait` định nghĩa thời gian chờ tối đa trước khi gom lô vô điều kiện
  các event trong hàng đợi.
- `--audit-log-batch-throttle-enable` định nghĩa việc throttling khi gom lô có được bật hay
  không.
- `--audit-log-batch-throttle-qps` định nghĩa số lô trung bình tối đa được sinh ra mỗi giây.
- `--audit-log-batch-throttle-burst` định nghĩa số lô tối đa được sinh ra tại cùng một thời
  điểm nếu lượng QPS cho phép trước đó chưa được dùng hết.

## Tinh chỉnh tham số (Parameter tuning)

Các tham số nên được đặt sao cho phù hợp với tải trên API server.

Ví dụ, nếu kube-apiserver nhận 100 request mỗi giây, và mỗi request chỉ được kiểm toán ở hai
stage `ResponseStarted` và `ResponseComplete`, bạn nên tính tới việc có ≅200 audit event được
sinh ra mỗi giây. Giả sử một lô chứa tối đa 100 event, bạn nên đặt mức throttling ít nhất là
2 truy vấn mỗi giây. Giả sử backend có thể mất tới 5 giây để ghi event, bạn nên đặt kích thước
buffer đủ giữ lượng event của 5 giây; tức là: 10 lô, hay 1000 event.

Tuy nhiên trong phần lớn trường hợp, các tham số mặc định là đủ và bạn không phải lo lắng về
việc đặt chúng thủ công. Bạn có thể theo dõi các metric Prometheus sau đây do kube-apiserver
phơi ra, cũng như theo dõi trong log, để giám sát trạng thái của hệ thống con kiểm toán.

- metric `apiserver_audit_event_total` chứa tổng số audit event đã được xuất.
- metric `apiserver_audit_error_total` chứa tổng số event bị loại bỏ do lỗi trong quá trình
  xuất.

### Cắt bớt entry log (Log entry truncation) {#truncate}

Cả hai backend log và webhook đều hỗ trợ giới hạn kích thước của các event được ghi log. Ví dụ,
dưới đây là danh sách flag khả dụng cho log backend:

- `audit-log-truncate-enabled` việc cắt bớt event và lô có được bật hay không.
- `audit-log-truncate-max-batch-size` kích thước tối đa tính bằng byte của lô gửi tới backend
  bên dưới.
- `audit-log-truncate-max-event-size` kích thước tối đa tính bằng byte của audit event gửi tới
  backend bên dưới.

Theo mặc định, truncate bị tắt ở cả `webhook` và `log`; quản trị viên cluster cần đặt
`audit-log-truncate-enabled` hoặc `audit-webhook-truncate-enabled` để bật tính năng này.

## Tiếp theo (What's next)

* Tìm hiểu về
  [annotation kiểm toán của mutating webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#mutating-webhook-auditing-annotations).
* Tìm hiểu thêm về các kiểu resource
  [`Event`](https://kubernetes.io/docs/reference/config-api/apiserver-audit.v1/#audit-k8s-io-v1-Event)
  và [`Policy`](https://kubernetes.io/docs/reference/config-api/apiserver-audit.v1/#audit-k8s-io-v1-Policy)
  bằng cách đọc tài liệu tham chiếu cấu hình Audit.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 22:

1. Trên `k8s-master` của cluster lab, bạn thêm flag `--audit-policy-file` trỏ tới một policy
   hợp lệ vào static Pod `kube-apiserver`, nhưng không thêm `--audit-log-path` lẫn
   `--audit-webhook-config-file`. Audit event có được lưu lại ở đâu không? Vì sao?
2. **Câu bẫy.** Policy của bạn có rule đầu tiên là `level: None` cho `resources: ["pods"]`, và
   một rule phía dưới là `level: RequestResponse` cũng khớp Pod. Một request `kubectl delete pod`
   được ghi ở mức nào? Cùng policy đó, một request đọc `pods/log` có bị rule `None` của `pods`
   chặn không?
3. Vì sao khi `kube-apiserver` chạy dưới dạng Pod (như trên cluster kubeadm của bạn), chỉ thêm
   hai flag `--audit-policy-file` và `--audit-log-path` là chưa đủ? Phải làm thêm gì để audit
   hoạt động và bản ghi không bị mất?
4. Nêu khác biệt về hành vi mặc định giữa backend `log` và backend `webhook` đối với batching,
   và giải thích mode `blocking-strict` khác `blocking` ở điểm nào.
5. Bạn nghi ngờ hệ audit đang làm rơi event. Hai metric Prometheus nào của kube-apiserver giúp
   bạn xác nhận, và mỗi metric cho biết điều gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không — không có bản ghi nào được lưu.** Policy chỉ quyết định *cái gì* được ghi; việc lưu
   trữ bền vững là của backend. Log backend bị tắt khi không có `--audit-log-path`, và webhook
   backend cần `--audit-webhook-config-file`. Không có backend nào được bật thì event sinh ra
   không được ghi đi đâu cả.
2. Request `delete pod` **không được ghi log**: event được so với danh sách rule theo thứ tự và
   **rule khớp đầu tiên quyết định level** — ở đây là `None`, các rule phía dưới không được xét
   tới nữa. Còn request đọc `pods/log` **không bị chặn bởi rule đó**: bài nói rõ resource
   `pods` không khớp request tới bất kỳ subresource nào của pods (nhất quán với RBAC), nên
   `pods/log` phải được khai riêng và sẽ khớp rule khác (hoặc rule vét) của policy.
3. Vì kube-apiserver chạy trong container: file policy trên host và thư mục log phải được
   **mount vào Pod**. Cần khai báo `volumeMounts` cho cả hai đường dẫn và `volumes` kiểu
   `hostPath` (`type: File` cho file policy, `type: DirectoryOrCreate` cho thư mục log) — không
   mount thì API server không đọc được policy, và log ghi trong container sẽ không bền vững
   trên host.
4. Batching và throttling **bật mặc định cho `webhook`** (mode mặc định `batch`) và **tắt mặc
   định cho `log`** (mode mặc định `blocking`; khi batching tắt, mọi flag batch bị bỏ qua và
   batching cũng không được khuyến nghị cho backend `log`). `blocking-strict` giống `blocking`
   nhưng nghiêm ngặt hơn: nếu ghi audit lỗi ở stage `RequestReceived` thì **toàn bộ request tới
   kube-apiserver thất bại**, thay vì chỉ bỏ qua lỗi audit.
5. `apiserver_audit_event_total` — tổng số audit event đã được xuất; và
   `apiserver_audit_error_total` — tổng số event **bị loại bỏ do lỗi** trong quá trình xuất.
   Metric thứ hai tăng nghĩa là hệ audit đang làm rơi event.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi sang bài
[208 — Mã hóa dữ liệu bí mật khi lưu trữ](208-encrypt-data-vi.md), bài 2/6 của
[giai đoạn 22](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu).

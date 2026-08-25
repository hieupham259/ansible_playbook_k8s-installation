# Di chuyển agent telemetry và bảo mật khỏi dockershim (Migrating telemetry and security agents from dockershim)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/migrating-telemetry-and-security-agents/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 27 — Di chuyển khỏi dockershim (cluster cũ)](00-ALO-TRINH-ADMIN.md#giai-đoạn-27--di-chuyển-khỏi-dockershim-cluster-cũ),
bài 5/6 · Phần II không có lab riêng: tự chấm bằng **Checkpoint** ghi ở cuối mục giai đoạn 27 trong
lộ trình. Bài này **không kiểm chứng được trên cluster lab**: không có agent nào bám vào Docker
socket để mà di chuyển. Script rà soát ở mục *Nhận diện các DaemonSet phụ thuộc vào Docker Engine*
thì chạy được, nhưng trên cluster lab nó **tất yếu trả về rỗng** — đó là xác nhận cluster lab nằm
ngoài phạm vi bài, không phải kiểm chứng nội dung bài.

**Đọc bài này như đọc hồ sơ bàn giao.** Lộ trình cho phép *bỏ qua toàn bộ giai đoạn 27 nếu cluster
của bạn đã dùng containerd*, và cluster lab thuộc nhóm đó. Giá trị của bài nằm ở chỗ khác: nó chỉ ra
nhóm phần mềm **dễ vỡ nhất** khi đổi runtime, và nhóm đó không phải ứng dụng của người dùng mà là
**agent telemetry và agent bảo mật** — thứ mà người bàn giao cluster hay quên nhắc. Lấy về hai thứ:
cơ chế nhận diện phụ thuộc, và ranh giới của cơ chế đó.

**Phải hiểu ở lần đọc này:**

- Vì sao agent telemetry lại đụng tới Docker Engine (mục *Tại sao một số agent telemetry giao tiếp
  với Docker Engine?*): lịch sử là Kubernetes ban đầu chỉ chạy với Docker; **tên Pod chỉ lấy được từ
  các thành phần Kubernetes**, còn **metric container thì không thuộc trách nhiệm của container
  runtime**, nên agent đời đầu phải truy vấn **cả hai phía** mới ra bức tranh đúng.
- Dạng phụ thuộc cụ thể là **chạy lệnh của bộ công cụ Docker Engine**: `docker ps`, `docker top` để
  liệt kê container và tiến trình, `docker logs` để nhận log dạng stream. Đổi runtime thì đúng những
  lệnh đó ngừng hoạt động.
- Điều kiện kỹ thuật để một Pod gọi được `dockerd` trên node: nó **buộc phải mount** hoặc filesystem
  chứa socket đặc quyền của Docker daemon, hoặc mount thẳng đường dẫn socket đó — nghĩa là luôn để
  lại dấu vết dưới dạng một volume `hostPath` (ví dụ `/var/run/docker.sock` trên image COS). Đó là
  lý do rà soát được bằng cách đọc `.spec.volumes[*].hostPath.path` của mọi Pod.
- **Giới hạn của script rà soát**, bài tự nói ra: nó chỉ bắt cách dùng phổ biến nhất. Pod có thể
  mount **thư mục cha `/var/run`** thay vì đường dẫn đầy đủ và vẫn tới được socket. Kết quả rỗng
  **không** đồng nghĩa với sạch.
- Hai nhóm đối tượng, hai cách rà khác nhau: agent chạy **dưới dạng DaemonSet** thì rà được từ phía
  API; agent **cài trực tiếp lên node** thì API server không nhìn thấy, và bài chỉ đúng một lối —
  liên hệ nhà cung cấp agent để xác minh.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Di chuyển khỏi dockershim* — danh sách bảy nhà cung cấp (Aqua, Datadog, Dynatrace, Falco, Prisma Cloud Compute, SignalFx, Yahoo Kubectl Flame) và tên Pod đặc trưng của từng hãng | là bảng tra theo sản phẩm, không phải cơ chế; chỉ áp dụng khi cluster bạn tiếp quản có đúng agent đó | chỉ khi thực sự cầm trong tay một cluster dockershim có agent của một trong các hãng này |
| Quy trình riêng của SignalFx: bỏ monitor `docker-container-stats`, bật và cấu hình `kubelet-metrics` | là chi tiết của một sản phẩm cụ thể; điều đáng nhớ chỉ là *tập metric thu được sẽ đổi, phải rà lại alert và dashboard* | chỉ áp dụng khi tiếp quản cluster dockershim đang chạy SignalFx Smart Agent |
| Link Google doc "phiên bản đang được cập nhật của hướng dẫn di chuyển" và ghi chú CNCF ở đầu trang | một cái là tài liệu sống ngoài kubernetes.io, một cái là chú thích biên tập của website | không cần |

---

> **Ghi chú:** Trang này liên kết tới các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án đó. Trang này tuân theo [nguyên tắc nội dung của website CNCF](https://github.com/cncf/foundation/blob/master/website-guidelines.md), liệt kê các mục theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

Hỗ trợ tích hợp trực tiếp với Docker Engine của Kubernetes đã bị ngừng (deprecated) và
đã bị loại bỏ. Hầu hết các ứng dụng không phụ thuộc trực tiếp vào runtime lưu trữ
container. Tuy nhiên, vẫn còn nhiều agent telemetry và giám sát (monitoring)
phụ thuộc vào Docker để thu thập metadata, log và
metric của container. Tài liệu này tổng hợp thông tin về cách phát hiện các
sự phụ thuộc này cũng như các liên kết hướng dẫn di chuyển những agent đó sang dùng các công cụ chung hoặc
runtime thay thế.

## Agent telemetry và bảo mật (Telemetry and security agents)

Trong một cluster Kubernetes, có một vài cách khác nhau để chạy agent telemetry hoặc
bảo mật. Một số agent phụ thuộc trực tiếp vào Docker Engine khi
chúng chạy dưới dạng DaemonSet hoặc trực tiếp trên node.

### Tại sao một số agent telemetry giao tiếp với Docker Engine? (Why do some telemetry agents communicate with Docker Engine?)

Về mặt lịch sử, Kubernetes được viết để hoạt động riêng với Docker Engine.
Kubernetes lo phần mạng và lập lịch, còn việc khởi chạy và chạy container
(bên trong Pod) trên node thì dựa vào Docker Engine. Một số thông tin
liên quan đến telemetry, chẳng hạn tên Pod, chỉ có thể lấy từ các thành phần
Kubernetes. Các dữ liệu khác, chẳng hạn metric của container, không thuộc trách nhiệm của
container runtime. Các agent telemetry đời đầu cần truy vấn cả container
runtime *và* Kubernetes để báo cáo được bức tranh chính xác. Theo thời gian, Kubernetes
có thêm khả năng hỗ trợ nhiều runtime, và hiện hỗ trợ bất kỳ runtime nào
tương thích với [giao diện container runtime (CRI)](https://kubernetes.io/docs/concepts/architecture/cri/).

Một số agent telemetry phụ thuộc cụ thể vào bộ công cụ của Docker Engine. Ví dụ, một agent
có thể chạy lệnh như
[`docker ps`](https://docs.docker.com/engine/reference/commandline/ps/)
hoặc [`docker top`](https://docs.docker.com/engine/reference/commandline/top/) để liệt kê
container và tiến trình, hoặc [`docker logs`](https://docs.docker.com/engine/reference/commandline/logs/)
để nhận log dạng stream. Nếu các node trong cluster hiện tại của bạn dùng
Docker Engine, và bạn chuyển sang một container runtime khác,
những lệnh này sẽ không còn hoạt động nữa.

### Nhận diện các DaemonSet phụ thuộc vào Docker Engine (Identify DaemonSets that depend on Docker Engine) {#identify-docker-dependency}

Nếu một Pod muốn gọi tới `dockerd` đang chạy trên node, Pod đó phải hoặc là:

- mount filesystem chứa socket đặc quyền (privileged) của Docker daemon, dưới dạng
  một volume; hoặc
- mount trực tiếp đường dẫn cụ thể của socket đặc quyền của Docker daemon, cũng dưới dạng volume.

Ví dụ: trên các image COS, Docker mở Unix domain socket của nó tại
`/var/run/docker.sock`. Điều này có nghĩa là spec của Pod sẽ bao gồm một
volume mount kiểu `hostPath` cho `/var/run/docker.sock`.

Đây là một script shell mẫu để tìm các Pod có mount ánh xạ trực tiếp tới
Docker socket. Script này in ra namespace và tên của Pod. Bạn có thể
bỏ phần `grep '/var/run/docker.sock'` để xem các mount khác.

```bash
kubectl get pods --all-namespaces \
-o=jsonpath='{range .items[*]}{"\n"}{.metadata.namespace}{":\t"}{.metadata.name}{":\t"}{range .spec.volumes[*]}{.hostPath.path}{", "}{end}{end}' \
| sort \
| grep '/var/run/docker.sock'
```

> **Ghi chú:** Có những cách khác để một Pod truy cập Docker trên host. Chẳng hạn, thư mục cha
> `/var/run` có thể được mount thay vì đường dẫn đầy đủ (như trong [ví dụ
> này](https://gist.github.com/itaysk/7bc3e56d69c4d72a549286d98fd557dd)).
> Script ở trên chỉ phát hiện các cách dùng phổ biến nhất.

### Phát hiện sự phụ thuộc vào Docker từ các agent trên node (Detecting Docker dependency from node agents)

Nếu các node trong cluster của bạn được tùy biến và cài thêm agent bảo mật và
telemetry trên node, hãy liên hệ nhà cung cấp agent
để xác minh xem agent đó có phụ thuộc gì vào Docker hay không.

### Các nhà cung cấp agent telemetry và bảo mật (Telemetry and security agent vendors)

Mục này nhằm tổng hợp thông tin về nhiều agent telemetry và
bảo mật khác nhau có thể phụ thuộc vào container runtime.

Chúng tôi duy trì phiên bản đang được cập nhật của hướng dẫn di chuyển cho các nhà cung cấp agent telemetry và bảo mật
trong [Google doc](https://docs.google.com/document/d/1ZFi4uKit63ga5sxEiZblfb-c23lFhvy6RXVPikS8wf0/edit#).
Vui lòng liên hệ nhà cung cấp để nhận hướng dẫn mới nhất về việc di chuyển khỏi dockershim.

## Di chuyển khỏi dockershim (Migration from dockershim)

### [Aqua](https://www.aquasec.com)

Không cần thay đổi gì: mọi thứ sẽ hoạt động trơn tru khi chuyển đổi runtime.

### [Datadog](https://www.datadoghq.com/product/)

Cách di chuyển:
[Docker deprecation in Kubernetes](https://docs.datadoghq.com/agent/guide/docker-deprecation/)
Pod truy cập Docker Engine có thể có tên chứa một trong các chuỗi:

- `datadog-agent`
- `datadog`
- `dd-agent`

### [Dynatrace](https://www.dynatrace.com/)

Cách di chuyển:
[Migrating from Docker-only to generic container metrics in Dynatrace](https://community.dynatrace.com/t5/Best-practices/Migrating-from-Docker-only-to-generic-container-metrics-in/m-p/167030#M49)

Thông báo hỗ trợ containerd: [Get automated full-stack visibility into
containerd-based Kubernetes
environments](https://www.dynatrace.com/news/blog/get-automated-full-stack-visibility-into-containerd-based-kubernetes-environments/)

Thông báo hỗ trợ CRI-O: [Get automated full-stack visibility into your CRI-O Kubernetes containers (Beta)](https://www.dynatrace.com/news/blog/get-automated-full-stack-visibility-into-your-cri-o-kubernetes-containers-beta/)

Pod truy cập Docker có thể có tên chứa:
- `dynatrace-oneagent`

### [Falco](https://falco.org)

Cách di chuyển:

[Migrate Falco from dockershim](https://falco.org/docs/getting-started/deployment/#docker-deprecation-in-kubernetes)
Falco hỗ trợ mọi runtime tương thích CRI (containerd được dùng trong cấu hình mặc định); tài liệu giải thích mọi chi tiết.
Pod truy cập Docker có thể có tên chứa:
- `falco`

### [Prisma Cloud Compute](https://docs.paloaltonetworks.com/prisma/prisma-cloud.html)

Xem [tài liệu của Prisma Cloud](https://docs.paloaltonetworks.com/prisma/prisma-cloud/prisma-cloud-admin-compute/install/install_kubernetes.html),
trong mục "Install Prisma Cloud on a CRI (non-Docker) cluster".
Pod truy cập Docker có thể có tên như:
- `twistlock-defender-ds`

### [SignalFx (Splunk)](https://www.splunk.com/en_us/investor-relations/acquisitions/signalfx.html)

SignalFx Smart Agent (đã ngừng hỗ trợ) dùng nhiều monitor khác nhau cho Kubernetes, bao gồm
`kubernetes-cluster`, `kubelet-stats/kubelet-metrics`, và `docker-container-stats`.
Monitor `kubelet-stats` trước đó đã bị nhà cung cấp ngừng hỗ trợ, thay bằng `kubelet-metrics`.
Monitor `docker-container-stats` là monitor bị ảnh hưởng bởi việc loại bỏ dockershim.
Không dùng `docker-container-stats` với container runtime khác Docker Engine.

Cách di chuyển khỏi agent phụ thuộc dockershim:
1. Xóa `docker-container-stats` khỏi danh sách [monitor đã cấu hình](https://github.com/signalfx/signalfx-agent/blob/main/docs/monitor-config.md).
   Lưu ý, nếu vẫn bật monitor này với runtime không phải dockershim, kết quả sẽ là metric bị báo cáo sai
   khi Docker được cài trên node, và không có metric nào khi Docker không được cài.
2. [Bật và cấu hình monitor `kubelet-metrics`](https://github.com/signalfx/signalfx-agent/blob/main/docs/monitors/kubelet-metrics.md).

> **Ghi chú:** Tập hợp các metric thu thập được sẽ thay đổi. Hãy rà soát lại các quy tắc cảnh báo (alerting rule) và dashboard của bạn.

Pod truy cập Docker có thể có tên kiểu như:

- `signalfx-agent`

### Yahoo Kubectl Flame

Flame không hỗ trợ container runtime nào khác ngoài Docker. Xem
[https://github.com/yahoo/kubectl-flame/issues/51](https://github.com/yahoo/kubectl-flame/issues/51)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 27. Cả bốn câu trả lời
được bằng lập luận, không cần cluster dockershim:

1. Bạn chạy script rà soát của bài trên `lab-k8s-master` và nó không in ra Pod nào. Bài cho phép kết
   luận gì từ kết quả rỗng đó, và **không** cho phép kết luận gì?
2. **Câu bẫy.** Một agent giám sát báo cáo được cả tên Pod lẫn metric CPU của từng container. Vậy nó
   phải hỏi Docker để lấy tên Pod, đúng không? Và metric container thì Docker cấp?
3. Vì sao chỉ cần đọc `.spec.volumes` của Pod là đã rà được nhóm DaemonSet phụ thuộc Docker, không
   cần đọc image hay dòng lệnh bên trong container?
4. Bài chia đối tượng rà soát thành hai nhóm và cho hai cách xử lý khác nhau. Hai nhóm đó là gì, và
   vì sao nhóm thứ hai không thể rà bằng `kubectl`?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Kết luận được: **không Pod nào mount thẳng `/var/run/docker.sock`** — cách dùng phổ biến nhất.
   **Không** kết luận được rằng cluster sạch phụ thuộc Docker, vì bài tự nêu hai lỗ hổng: có những
   cách khác để Pod truy cập Docker trên host, chẳng hạn **mount thư mục cha `/var/run`** thay vì
   đường dẫn đầy đủ — script bỏ sót ca đó; và **agent cài trực tiếp lên node** thì không phải Pod
   nên không xuất hiện trong kết quả. Script chỉ phát hiện **cách dùng phổ biến nhất**, đúng như bài
   ghi.
2. **Sai cả hai vế.** Bài nói ngược lại: **tên Pod chỉ có thể lấy từ các thành phần Kubernetes**,
   không phải từ Docker; còn **metric của container thì không thuộc trách nhiệm của container
   runtime**. Chính vì mỗi phía chỉ giữ một mảnh mà **agent telemetry đời đầu phải truy vấn cả
   container runtime lẫn Kubernetes** mới ghép được bức tranh đúng. Cái agent thực sự lấy từ Docker
   là thứ khác: **liệt kê container và tiến trình** (`docker ps`, `docker top`) và **log dạng
   stream** (`docker logs`) — và đó mới là phần chết khi đổi runtime.
3. Vì đường vào `dockerd` bị chặn ở tầng dưới: muốn gọi được daemon trên node, Pod **buộc phải
   mount** hoặc filesystem chứa socket đặc quyền của Docker daemon, hoặc thẳng đường dẫn socket ấy —
   **dưới dạng một volume**. Không có lối nào khác, nên mọi phụ thuộc kiểu này **đều để lại dấu vết
   trong `spec` của Pod**. Đọc `spec` rẻ và chắc hơn đọc image: `spec` là thứ API server đang giữ,
   còn hành vi bên trong container thì phải chạy mới biết.
4. Nhóm một là **agent chạy dưới dạng DaemonSet** trong cluster; nhóm hai là **agent bảo mật và
   telemetry được cài trực tiếp lên node**, trên những node đã được tùy biến. Nhóm hai không rà bằng
   `kubectl` được vì **chúng không phải object của Kubernetes** — không Pod, không DaemonSet, nên
   API server không biết chúng tồn tại. Bài chỉ đúng một lối cho nhóm này: **liên hệ nhà cung cấp
   agent** để xác minh agent có phụ thuộc gì vào Docker hay không.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

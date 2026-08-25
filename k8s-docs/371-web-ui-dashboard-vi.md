# Triển khai và Truy cập Kubernetes Dashboard (Deploy and Access the Kubernetes Dashboard)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
>
> Triển khai giao diện web (Kubernetes Dashboard) và truy cập nó.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 30 — Truy cập ứng dụng trong cluster](00-ALO-TRINH-ADMIN.md#giai-đoạn-30--truy-cập-ứng-dụng-trong-cluster),
bài 2/4 · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 30](00-ALO-TRINH-ADMIN.md#giai-đoạn-30--truy-cập-ứng-dụng-trong-cluster).

**Bài này là ngoại lệ: đọc để biết, không làm theo.** Lộ trình ghi rõ Kubernetes Dashboard **đã
deprecated** ở upstream — hai blockquote đầu trang nói dự án đã được lưu trữ (archive), không còn
được bảo trì, và khuyến nghị dùng [Headlamp](https://headlamp.dev/) cho các cài đặt mới. **Đừng cài
Dashboard vào cluster lab** và đừng coi nó là add-on chuẩn: giai đoạn 30 không có bước thực hành nào
cho bài này, và Checkpoint cũng không hỏi tới nó. Giá trị của bài nằm ở chỗ khác — nó cho thấy một
UI chạy **bên trong** cluster thì lấy quyền ở đâu và người dùng chạm tới nó bằng đường nào.

**Phải hiểu ở lần đọc này:**

- Hai blockquote mở đầu: Dashboard **đã deprecated và không còn được bảo trì**, dự án đã archive,
  upstream khuyến nghị Headlamp. Đây là lý do bài nằm ngoài mạch thực hành.
- Mục *Triển khai giao diện Dashboard*: Dashboard **không được triển khai mặc định** và hiện chỉ
  hỗ trợ cài bằng Helm. Nó là một add-on cài thêm, không phải thành phần có sẵn của cluster.
- Mục *Truy cập giao diện Dashboard*: Dashboard được triển khai với **cấu hình RBAC tối thiểu** và
  **chỉ hỗ trợ đăng nhập bằng Bearer Token**; cảnh báo ngay sau đó nói user mẫu trong hướng dẫn có
  **quyền quản trị** và chỉ dành cho mục đích học tập. Một token quản trị dán vào ô đăng nhập của
  một UI là một đường vào cluster với toàn quyền.
- Mục *Proxy dòng lệnh*: cách truy cập được nêu là `kubectl port-forward` tới Service
  `kubernetes-dashboard-kong-proxy`, và giao diện **chỉ** truy cập được từ chính máy nơi chạy lệnh.
  Đây đúng là đường `port-forward` trong ba đường mà Checkpoint giai đoạn 30 bắt phân biệt — không
  phải NodePort, cũng không phải apiserver proxy.
- Các màn hình ở mục *Sử dụng Dashboard*: *Admin overview* (Node, Namespace, PersistentVolume),
  *Workloads*, *Services*, *Storage*, *ConfigMap và Secret* — màn hình này **hiển thị các secret vốn
  bị ẩn theo mặc định** — và *Trình xem log* đi sâu vào log các container của **một** Pod duy nhất.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hai lệnh `helm repo add` và `helm upgrade --install` ở mục *Triển khai giao diện Dashboard* | Helm nằm trong [A1.4 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) — phần stack Lab 00 **không** cài | **không áp dụng cho cluster lab**: lộ trình yêu cầu không cài Dashboard, nên hai lệnh này không có chỗ chạy trong chuỗi lab |
| Toàn bộ mục *Triển khai ứng dụng chạy trong container* — trình hướng dẫn với App name, Container image, Number of pods, Service, và **Advanced options** | đó là thao tác trên UI của một dự án đã archive; mọi trường trong đó bạn đã biết dưới dạng field của manifest | các bài đã đọc: [63 — Deployment](63-deployment-vi.md), [82 — Service](82-service-vi.md), [40 — Image và imagePullSecrets](40-images-vi.md), [109 — Secret](109-secret-vi.md), [330 — command và args](330-define-command-argument-vi.md) |
| Hướng dẫn tạo user mẫu (link GitHub) và ghi chú về xác thực bằng kubeconfig | phần này là RBAC và ServiceAccount token, không phải nội dung của Dashboard | [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài [119 — Kiểm soát truy cập](119-controlling-access-vi.md); thực hành ở [Lab 9a](labs/LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md) |
| Ba ảnh chụp màn hình và các mục mô tả từng màn hình | Dashboard không được cài trên cluster lab nên không có gì để đối chiếu | **không áp dụng cho cluster lab**: cùng dữ liệu đó bạn đọc bằng `kubectl` như đã làm từ [giai đoạn 1b](00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl) |

---

> **Kubernetes Dashboard đã bị deprecate và không còn được bảo trì.**
>
> Dự án Kubernetes Dashboard đã được lưu trữ (archive) và không còn được bảo trì tích cực nữa.
> Với các cài đặt mới, bạn nên cân nhắc dùng [Headlamp](https://headlamp.dev/).

> **Ghi chú:** Với các bản triển khai chạy bên trong cluster tương tự Kubernetes Dashboard, hãy xem
> [hướng dẫn cài đặt Headlamp trong cluster](https://headlamp.dev/docs/latest/installation/in-cluster/).

Dashboard là một giao diện người dùng Kubernetes dạng web.
Bạn có thể dùng Dashboard để triển khai các ứng dụng chạy trong container lên một cluster Kubernetes,
gỡ lỗi (troubleshoot) ứng dụng container của mình, và quản lý các tài nguyên của cluster.
Bạn có thể dùng Dashboard để có cái nhìn tổng quan về các ứng dụng đang chạy trên cluster,
cũng như để tạo hoặc chỉnh sửa từng tài nguyên Kubernetes riêng lẻ
(chẳng hạn Deployment, Job, DaemonSet, v.v.).
Ví dụ, bạn có thể scale một Deployment, khởi động một bản cập nhật cuốn chiếu (rolling update),
khởi động lại một pod, hoặc triển khai ứng dụng mới bằng trình hướng dẫn triển khai (deploy wizard).

Dashboard cũng cung cấp thông tin về trạng thái của các tài nguyên Kubernetes trong cluster của bạn
và về bất kỳ lỗi nào có thể đã xảy ra.

![Giao diện Kubernetes Dashboard](https://kubernetes.io/images/docs/ui-dashboard.png)

## Triển khai giao diện Dashboard (Deploying the Dashboard UI)

> **Ghi chú:** Hiện tại Kubernetes Dashboard chỉ hỗ trợ cài đặt bằng Helm, vì cách này nhanh hơn
> và giúp chúng ta kiểm soát tốt hơn tất cả các dependency mà Dashboard cần để chạy.

Giao diện Dashboard không được triển khai mặc định. Để triển khai nó, hãy chạy lệnh sau:

```shell
# Thêm repository kubernetes-dashboard
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
# Triển khai một Helm Release tên là "kubernetes-dashboard" bằng chart kubernetes-dashboard
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

## Truy cập giao diện Dashboard (Accessing the Dashboard UI)

Để bảo vệ dữ liệu cluster của bạn, Dashboard được triển khai với một cấu hình RBAC tối thiểu theo
mặc định.
Hiện tại, Dashboard chỉ hỗ trợ đăng nhập bằng Bearer Token.
Để tạo một token cho phần demo này, bạn có thể làm theo hướng dẫn của chúng tôi về
[cách tạo một user mẫu](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md).

> **Cảnh báo:** User mẫu được tạo trong hướng dẫn đó sẽ có quyền quản trị (administrative privileges)
> và chỉ dành cho mục đích học tập.

### Proxy dòng lệnh (Command line proxy)

Bạn có thể bật quyền truy cập vào Dashboard bằng công cụ dòng lệnh `kubectl`,
bằng cách chạy lệnh sau:

```
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```

Kubectl sẽ làm cho Dashboard sẵn sàng tại [https://localhost:8443](https://localhost:8443).

Giao diện _chỉ_ có thể được truy cập từ chính máy nơi lệnh này được thực thi. Xem
`kubectl port-forward --help` để biết thêm các tùy chọn khác.

> **Ghi chú:** Phương thức xác thực bằng kubeconfig **không** hỗ trợ các nhà cung cấp danh tính
> (identity provider) bên ngoài hoặc xác thực dựa trên certificate X.509.

## Màn hình chào mừng (Welcome view)

Khi bạn truy cập Dashboard trên một cluster rỗng, bạn sẽ thấy trang chào mừng.
Trang này chứa một link tới chính tài liệu này cũng như một nút để triển khai ứng dụng đầu tiên của
bạn.
Ngoài ra, bạn có thể xem những ứng dụng hệ thống nào đang chạy mặc định trong
[namespace](242-namespaces-tasks-vi.md) `kube-system`
của cluster, ví dụ như chính Dashboard.

![Trang chào mừng của Kubernetes Dashboard](https://kubernetes.io/images/docs/ui-dashboard-zerostate.png)

## Triển khai ứng dụng chạy trong container (Deploying containerized applications)

Dashboard cho phép bạn tạo và triển khai một ứng dụng chạy trong container dưới dạng một Deployment
và một Service tùy chọn thông qua một trình hướng dẫn đơn giản.
Bạn có thể chỉ định thủ công các thông tin chi tiết của ứng dụng, hoặc tải lên một file _manifest_
dạng YAML hoặc JSON chứa cấu hình ứng dụng.

Nhấn nút **CREATE** ở góc trên bên phải của bất kỳ trang nào để bắt đầu.

### Chỉ định thông tin chi tiết của ứng dụng (Specifying application details)

Trình hướng dẫn triển khai yêu cầu bạn cung cấp các thông tin sau:

- **App name** (bắt buộc): Tên cho ứng dụng của bạn.
  Một [label](18-labels-vi.md) mang tên này
  sẽ được thêm vào Deployment và Service (nếu có) sẽ được triển khai.

  Tên ứng dụng phải là duy nhất trong
  [namespace](242-namespaces-tasks-vi.md) Kubernetes đã chọn.
  Nó phải bắt đầu bằng một ký tự chữ thường, kết thúc bằng một ký tự chữ thường hoặc một chữ số,
  và chỉ chứa chữ cái thường, chữ số và dấu gạch ngang (-). Nó bị giới hạn ở 24 ký tự.
  Khoảng trắng ở đầu và cuối sẽ bị bỏ qua.

- **Container image** (bắt buộc):
  URL của một [container image](40-images-vi.md) Docker công
  khai trên bất kỳ registry nào, hoặc một image riêng tư (thường được lưu trữ trên Google Container
  Registry hoặc Docker Hub).
  Phần khai báo container image phải kết thúc bằng dấu hai chấm.

- **Number of pods** (bắt buộc): Số lượng Pod mục tiêu mà bạn muốn ứng dụng của mình được triển khai.
  Giá trị này phải là một số nguyên dương.

  Một [Deployment](63-deployment-vi.md) sẽ được tạo
  để duy trì số lượng Pod mong muốn trên khắp cluster của bạn.

- **Service** (tùy chọn): Với một số phần của ứng dụng (ví dụ các frontend), bạn có thể muốn expose
  một [Service](82-service-vi.md) ra một địa chỉ IP
  bên ngoài, có thể là IP công khai nằm ngoài cluster của bạn (external Service).

  > **Ghi chú:** Với các external Service, bạn có thể cần mở một hoặc nhiều port để làm được điều đó.

  Các Service khác chỉ nhìn thấy được từ bên trong cluster được gọi là internal Service.

  Bất kể loại Service là gì, nếu bạn chọn tạo một Service và container của bạn lắng nghe trên một port
  (incoming), bạn cần chỉ định hai port.
  Service sẽ được tạo với ánh xạ từ port (incoming) sang target port mà container nhìn thấy.
  Service này sẽ định tuyến tới các Pod bạn đã triển khai. Các giao thức được hỗ trợ là TCP và UDP.
  Tên DNS nội bộ của Service này sẽ là giá trị bạn đã chỉ định ở phần tên ứng dụng phía trên.

Nếu cần, bạn có thể mở rộng mục **Advanced options**, nơi bạn có thể chỉ định thêm các thiết lập khác:

- **Description**: Đoạn văn bản bạn nhập ở đây sẽ được thêm vào Deployment dưới dạng một
  [annotation](20-annotations-vi.md)
  và được hiển thị trong phần thông tin chi tiết của ứng dụng.

- **Labels**: Các [label](18-labels-vi.md)
  mặc định được dùng cho ứng dụng của bạn là tên ứng dụng và phiên bản.
  Bạn có thể chỉ định thêm các label khác để áp dụng cho Deployment, Service (nếu có) và các Pod,
  chẳng hạn như release, environment, tier, partition và release track.

  Ví dụ:

  ```conf
  release=1.0
  tier=frontend
  environment=pod
  track=stable
  ```

- **Namespace**: Kubernetes hỗ trợ nhiều cluster ảo cùng chạy trên một cluster vật lý.
  Các cluster ảo này được gọi là
  [namespace](242-namespaces-tasks-vi.md).
  Chúng cho phép bạn phân chia tài nguyên thành các nhóm được đặt tên một cách logic.

  Dashboard đưa ra tất cả các namespace hiện có trong một danh sách thả xuống (dropdown), và cho phép
  bạn tạo một namespace mới.
  Tên namespace có thể chứa tối đa 63 ký tự chữ và số cùng dấu gạch ngang (-) nhưng không được chứa
  chữ in hoa.
  Tên namespace không nên chỉ gồm toàn chữ số.
  Nếu tên được đặt là một con số, chẳng hạn 10, pod sẽ được đặt vào namespace `default`.

  Trong trường hợp tạo namespace thành công, nó sẽ được chọn mặc định.
  Nếu việc tạo thất bại, namespace đầu tiên sẽ được chọn.

- **Image Pull Secret**:
  Trong trường hợp container image Docker được chỉ định là riêng tư, nó có thể yêu cầu thông tin
  đăng nhập dạng [pull secret](109-secret-vi.md).

  Dashboard đưa ra tất cả các secret hiện có trong một danh sách thả xuống, và cho phép bạn tạo một
  secret mới.
  Tên secret phải tuân theo cú pháp tên miền DNS, ví dụ `new.image-pull.secret`.
  Nội dung của một secret phải được mã hóa base64 và được chỉ định trong một file
  [`.dockercfg`](40-images-vi.md#specifying-imagepullsecrets-on-a-pod).
  Tên secret có thể gồm tối đa 253 ký tự.

  Trong trường hợp tạo image pull secret thành công, nó sẽ được chọn mặc định. Nếu việc tạo thất bại,
  không có secret nào được áp dụng.

- **CPU requirement (cores)** và **Memory requirement (MiB)**:
  Bạn có thể chỉ định
  [giới hạn tài nguyên](232-memory-default-namespace-vi.md)
  tối thiểu cho container. Theo mặc định, các Pod chạy với giới hạn CPU và bộ nhớ không bị ràng buộc.

- **Run command** và **Run command arguments**:
  Theo mặc định, các container của bạn chạy
  [lệnh entrypoint](330-define-command-argument-vi.md)
  mặc định của image Docker được chỉ định.
  Bạn có thể dùng các tùy chọn command và arguments để ghi đè giá trị mặc định.

- **Run as privileged**: Thiết lập này quyết định liệu các tiến trình trong
  [privileged container](https://kubernetes.io/docs/concepts/workloads/pods/#privileged-mode-for-containers)
  có tương đương với các tiến trình chạy dưới quyền root trên host hay không.
  Privileged container có thể tận dụng các khả năng như thao tác với network stack và truy cập thiết bị.

- **Environment variables**: Kubernetes expose các Service thông qua
  [biến môi trường](336-env-variable-expose-pod-info-vi.md).
  Bạn có thể tạo biến môi trường hoặc truyền tham số cho các lệnh của mình bằng giá trị của các biến
  môi trường.
  Chúng có thể được dùng trong ứng dụng để tìm ra một Service.
  Các giá trị có thể tham chiếu tới biến khác bằng cú pháp `$(VAR_NAME)`.

### Tải lên file YAML hoặc JSON (Uploading a YAML or JSON file)

Kubernetes hỗ trợ cấu hình khai báo (declarative configuration).
Theo phong cách này, toàn bộ cấu hình được lưu trong các manifest (file cấu hình YAML hoặc JSON).
Các manifest sử dụng schema tài nguyên của
[API](21-kubernetes-api-vi.md) Kubernetes.

Thay vì chỉ định thông tin chi tiết của ứng dụng trong trình hướng dẫn triển khai, bạn có thể định
nghĩa ứng dụng của mình trong một hoặc nhiều manifest, rồi tải các file đó lên bằng Dashboard.

## Sử dụng Dashboard (Using Dashboard)

Các phần sau đây mô tả các màn hình của giao diện Kubernetes Dashboard; chúng cung cấp những gì và
có thể được dùng ra sao.

### Điều hướng (Navigation)

Khi có các object Kubernetes được định nghĩa trong cluster, Dashboard sẽ hiển thị chúng ở màn hình
ban đầu.
Theo mặc định, chỉ các object thuộc namespace _default_ được hiển thị và điều này có thể thay đổi
bằng bộ chọn namespace nằm trong menu điều hướng.

Dashboard hiển thị hầu hết các loại object Kubernetes và nhóm chúng vào một vài hạng mục menu.

#### Tổng quan cho quản trị viên (Admin overview)

Với người quản trị cluster và namespace, Dashboard liệt kê Node, Namespace và PersistentVolume, đồng
thời có màn hình chi tiết cho chúng.
Màn hình danh sách Node chứa các chỉ số (metric) về mức sử dụng CPU và bộ nhớ được tổng hợp trên tất
cả các Node.
Màn hình chi tiết hiển thị các chỉ số của một Node, phần đặc tả (specification), trạng thái, tài
nguyên đã cấp phát, các sự kiện (event) và các pod đang chạy trên node đó.

#### Workloads

Hiển thị tất cả các ứng dụng đang chạy trong namespace đã chọn.
Màn hình này liệt kê các ứng dụng theo loại workload (ví dụ: Deployment, ReplicaSet, StatefulSet).
Mỗi loại workload có thể được xem riêng.
Các danh sách tóm tắt những thông tin có thể hành động được về workload, chẳng hạn số pod đã sẵn sàng
(ready) của một ReplicaSet hoặc mức sử dụng bộ nhớ hiện tại của một Pod.

Màn hình chi tiết của workload hiển thị thông tin trạng thái và đặc tả, đồng thời làm nổi bật mối
quan hệ giữa các object.
Ví dụ, các Pod mà ReplicaSet đang điều khiển, hoặc các ReplicaSet mới và HorizontalPodAutoscaler của
các Deployment.

#### Services

Hiển thị các tài nguyên Kubernetes cho phép expose dịch vụ ra thế giới bên ngoài và khám phá chúng
bên trong cluster.
Vì lý do đó, màn hình Service và Ingress hiển thị các Pod mà chúng nhắm tới, các endpoint nội bộ dùng
cho kết nối trong cluster và các endpoint bên ngoài dành cho người dùng bên ngoài.

#### Lưu trữ (Storage)

Màn hình Storage hiển thị các tài nguyên PersistentVolumeClaim được các ứng dụng dùng để lưu dữ liệu.

#### ConfigMap và Secret (ConfigMaps and Secrets) {#config-maps-and-secrets}

Hiển thị tất cả các tài nguyên Kubernetes được dùng để cấu hình trực tiếp (live configuration) cho
các ứng dụng đang chạy trong cluster.
Màn hình này cho phép chỉnh sửa và quản lý các object cấu hình, và hiển thị các secret vốn bị ẩn theo
mặc định.

#### Trình xem log (Logs viewer)

Các danh sách Pod và trang chi tiết đều có link tới một trình xem log được tích hợp sẵn trong
Dashboard.
Trình xem này cho phép đi sâu vào log từ các container thuộc về một Pod duy nhất.

![Trình xem log](https://kubernetes.io/images/docs/ui-dashboard-logs-view.png)

## Tiếp theo (What's next)

Để biết thêm thông tin, hãy xem
[trang dự án Kubernetes Dashboard](https://github.com/kubernetes/dashboard).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 30:

1. Nêu hai điều trong chính trang này khiến lộ trình xếp Dashboard là "đọc để biết, không cài vào
   cluster lab".
2. **Câu bẫy.** Dashboard hiển thị được Node, PersistentVolume, cả nội dung Secret. Vậy nó có một
   đường vào đặc quyền riêng, đi vòng qua cơ chế phân quyền của cluster, đúng không?
3. Giả sử bạn cài Dashboard rồi chạy đúng lệnh ở mục *Proxy dòng lệnh* trong phiên SSH tới
   `lab-k8s-master` (`192.168.100.221`). Trình duyệt trên máy host của bạn mở
   `https://192.168.100.221:8443` — có vào được không, vì sao?
4. Màn hình *ConfigMap và Secret* làm gì với những secret vốn bị ẩn theo mặc định, và ghép với
   cảnh báo về user mẫu thì hệ quả bảo mật là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Thứ nhất: **dự án đã deprecated, đã archive và không còn được bảo trì**, upstream khuyến nghị
   chuyển sang Headlamp — hai blockquote đầu trang nói thẳng điều đó. Thứ hai: **Dashboard không
   được triển khai mặc định và hiện chỉ cài được bằng Helm**, tức nó là một add-on của bên thứ ba
   phải cài thêm, không phải thành phần của cluster.
2. **Không.** Trang nói Dashboard được triển khai với **cấu hình RBAC tối thiểu theo mặc định** và
   **chỉ đăng nhập bằng Bearer Token** — nó chỉ nhìn thấy đúng những gì token bạn dán vào được
   phép. Lý do trong hướng dẫn nó "thấy mọi thứ" là vì **user mẫu được tạo với quyền quản trị**, và
   chính trang cảnh báo user đó chỉ dành cho mục đích học tập. Đặc quyền đến từ token bạn cấp, không
   phải từ Dashboard.
3. **Không vào được.** Trang nói rõ giao diện *chỉ* có thể được truy cập từ **chính máy nơi lệnh
   được thực thi**. Lệnh chạy trên `lab-k8s-master` thì `port-forward` lắng nghe trên localhost của
   node đó, nên `https://localhost:8443` chỉ mở được từ bên trong phiên trên chính node. Máy host
   không nằm trong đường đó — `port-forward` là công cụ cho **một máy trạm**, không phải cách expose
   dịch vụ cho người khác dùng.
4. Màn hình đó **hiển thị các secret vốn bị ẩn theo mặc định**, và còn cho phép chỉnh sửa, quản lý
   object cấu hình. Ghép với user mẫu **có quyền quản trị**: bất kỳ ai chạm được vào phiên đăng nhập
   đó đều **đọc được nội dung Secret của cluster ngay trên trình duyệt**. Đó là lý do một UI chạy
   trong cluster phải được coi như một đường vào đặc quyền chứ không phải một tiện ích xem cho vui.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

# Triển khai và Truy cập Kubernetes Dashboard (Deploy and Access the Kubernetes Dashboard)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
>
> Triển khai giao diện web (Kubernetes Dashboard) và truy cập nó.

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

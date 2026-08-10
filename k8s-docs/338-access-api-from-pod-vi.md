# Truy cập Kubernetes API từ một Pod (Accessing the Kubernetes API from a Pod)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/>

Hướng dẫn này trình bày cách truy cập Kubernetes API từ bên trong một pod.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Khuyến nghị chạy hướng dẫn này trên một cluster có ít nhất hai node không
đóng vai trò là máy chủ control plane. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Truy cập API từ bên trong một Pod (Accessing the API from within a Pod)

Khi truy cập API từ bên trong một Pod, việc định vị và xác thực (authenticate)
với API server hơi khác so với trường hợp client bên ngoài.

Cách dễ nhất để sử dụng Kubernetes API từ một Pod là dùng một trong các
[thư viện client](https://kubernetes.io/docs/reference/using-api/client-libraries/) chính thức. Các
thư viện này có thể tự động khám phá (discover) API server và xác thực.

### Sử dụng các thư viện client chính thức (Using Official Client Libraries)

Từ bên trong một Pod, các cách được khuyến nghị để kết nối tới Kubernetes API là:

- Với client Go, hãy dùng
  [thư viện client Go](https://github.com/kubernetes/client-go/) chính thức.
  Hàm `rest.InClusterConfig()` tự động xử lý việc khám phá host của API và xác thực.
  Xem [một ví dụ tại đây](https://git.k8s.io/client-go/examples/in-cluster-client-configuration/main.go).

- Với client Python, hãy dùng
  [thư viện client Python](https://github.com/kubernetes-client/python/) chính thức.
  Hàm `config.load_incluster_config()` tự động xử lý việc khám phá host của API và xác thực.
  Xem [một ví dụ tại đây](https://github.com/kubernetes-client/python/blob/master/examples/in_cluster_config.py).

- Còn có nhiều thư viện khác, vui lòng tham khảo trang
  [Client Libraries](https://kubernetes.io/docs/reference/using-api/client-libraries/).

Trong mỗi trường hợp, thông tin xác thực (credential) service account của Pod được dùng để
giao tiếp một cách an toàn với API server.

### Truy cập trực tiếp REST API (Directly accessing the REST API)

Khi đang chạy trong một Pod, container của bạn có thể tạo một URL HTTPS cho Kubernetes API
server bằng cách lấy các biến môi trường `KUBERNETES_SERVICE_HOST` và
`KUBERNETES_SERVICE_PORT_HTTPS`. Địa chỉ trong cluster (in-cluster) của API server cũng được
công bố qua một Service tên là `kubernetes` trong namespace `default`, nhờ đó các pod có thể
tham chiếu `kubernetes.default.svc` như một tên DNS cho API server cục bộ.

> **Ghi chú:**
> Kubernetes không đảm bảo rằng API server có certificate hợp lệ cho
> hostname `kubernetes.default.svc`;
> tuy nhiên, control plane **được** kỳ vọng sẽ xuất trình một certificate hợp lệ cho
> hostname hoặc địa chỉ IP mà `$KUBERNETES_SERVICE_HOST` đại diện.

Cách được khuyến nghị để xác thực với API server là dùng thông tin xác thực của một
[service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/).
Theo mặc định, một Pod được gắn với một service account, và thông tin xác thực (token) của
service account đó được đặt vào cây filesystem của mỗi container trong Pod đó,
tại `/var/run/secrets/kubernetes.io/serviceaccount/token`.

Nếu có, một bundle certificate được đặt vào cây filesystem của mỗi container
tại `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`, và nên được
dùng để xác minh serving certificate của API server.

Cuối cùng, namespace mặc định dùng cho các thao tác API có phạm vi namespace được đặt trong một
file tại `/var/run/secrets/kubernetes.io/serviceaccount/namespace` trong mỗi container.

### Sử dụng kubectl proxy (Using kubectl proxy)

Nếu bạn muốn truy vấn API mà không dùng thư viện client chính thức, bạn có thể chạy `kubectl proxy`
như là [lệnh (command)](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)
của một container sidecar mới trong Pod. Bằng cách này, `kubectl proxy` sẽ xác thực
với API và mở nó ra trên interface `localhost` của Pod, nhờ đó các container khác
trong Pod có thể dùng nó trực tiếp.

### Không sử dụng proxy (Without using a proxy)

Bạn có thể tránh dùng kubectl proxy bằng cách truyền token xác thực
trực tiếp cho API server. Certificate nội bộ bảo đảm an toàn cho kết nối.

```shell
# Trỏ tới hostname nội bộ của API server
APISERVER=https://kubernetes.default.svc

# Đường dẫn tới token của ServiceAccount
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount

# Đọc namespace của Pod này
NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)

# Đọc bearer token của ServiceAccount
TOKEN=$(cat ${SERVICEACCOUNT}/token)

# Tham chiếu certificate authority (CA) nội bộ
CACERT=${SERVICEACCOUNT}/ca.crt

# Khám phá API với TOKEN
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/api
```

Kết quả đầu ra sẽ tương tự như sau:

```json
{
  "kind": "APIVersions",
  "versions": ["v1"],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```

# Truy cập cluster (Accessing Clusters)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/>
>
> Chủ đề này thảo luận nhiều cách khác nhau để tương tác với cluster.

## Truy cập lần đầu với kubectl (Accessing for the first time with kubectl)

Khi truy cập Kubernetes API lần đầu tiên, chúng tôi khuyên bạn dùng công cụ dòng lệnh
của Kubernetes là `kubectl`.

Để truy cập một cluster, bạn cần biết vị trí (location) của cluster và có thông tin
xác thực (credentials) để truy cập nó. Thông thường, những thứ này được thiết lập tự
động khi bạn làm theo một [Hướng dẫn bắt đầu](https://kubernetes.io/docs/setup/),
hoặc do một người khác đã dựng cluster và cung cấp cho bạn thông tin xác thực cùng vị trí.

Kiểm tra vị trí và thông tin xác thực mà kubectl đang biết bằng lệnh sau:

```shell
kubectl config view
```

Nhiều [ví dụ](https://kubernetes.io/docs/reference/kubectl/quick-reference/) cung cấp
phần giới thiệu về cách dùng `kubectl`, và tài liệu đầy đủ nằm trong
[tài liệu tham khảo kubectl](https://kubernetes.io/docs/reference/kubectl/).

## Truy cập trực tiếp REST API (Directly accessing the REST API) {#directly-accessing-the-rest-api}

Kubectl đảm nhận việc định vị (locate) và xác thực (authenticate) với apiserver.
Nếu bạn muốn truy cập trực tiếp REST API bằng một http client như
curl hay wget, hoặc bằng trình duyệt, có một số cách để định vị và xác thực:

- Chạy kubectl ở chế độ proxy.
  - Cách được khuyến nghị.
  - Dùng vị trí apiserver đã được lưu sẵn.
  - Xác minh danh tính của apiserver bằng certificate tự ký (self-signed cert). Không thể xảy ra tấn công xen giữa (MITM).
  - Tự xác thực với apiserver.
  - Trong tương lai, có thể thực hiện cân bằng tải (load balancing) và chuyển đổi dự phòng (failover) thông minh phía client.
- Cung cấp trực tiếp vị trí và thông tin xác thực cho http client.
  - Cách thay thế.
  - Phù hợp với một số loại mã client bị nhầm lẫn khi dùng proxy.
  - Cần import một root certificate vào trình duyệt của bạn để phòng chống tấn công MITM.

### Dùng kubectl proxy (Using kubectl proxy)

Lệnh sau chạy kubectl ở chế độ hoạt động như một reverse proxy. Nó đảm nhận
việc định vị apiserver và xác thực.
Chạy lệnh như sau:

```shell
kubectl proxy --port=8080
```

Xem [kubectl proxy](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#proxy) để biết thêm chi tiết.

Sau đó bạn có thể khám phá API bằng curl, wget, hoặc trình duyệt; với IPv6 hãy thay
localhost bằng [::1], như sau:

```shell
curl http://localhost:8080/api/
```

Kết quả tương tự như sau:

```json
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```

### Không dùng kubectl proxy (Without kubectl proxy)

Dùng `kubectl apply` và `kubectl describe secret...` để tạo một token cho service account mặc định, với grep/cut:

Trước tiên, tạo Secret, yêu cầu một token cho ServiceAccount `default`:

```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: default-token
  annotations:
    kubernetes.io/service-account.name: default
type: kubernetes.io/service-account-token
EOF
```

Tiếp theo, chờ token controller điền token vào Secret:

```shell
while ! kubectl describe secret default-token | grep -E '^token' >/dev/null; do
  echo "waiting for token..." >&2
  sleep 1
done
```

Lấy và sử dụng token đã được sinh ra:

```shell
APISERVER=$(kubectl config view --minify | grep server | cut -f 2- -d ":" | tr -d " ")
TOKEN=$(kubectl describe secret default-token | grep -E '^token' | cut -f2 -d':' | tr -d " ")

curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
```

Kết quả tương tự như sau:

```json
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```

Dùng `jsonpath`:

```shell
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
TOKEN=$(kubectl get secret default-token -o jsonpath='{.data.token}' | base64 --decode)

curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
```

Kết quả tương tự như sau:

```json
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```

Các ví dụ trên dùng cờ `--insecure`. Điều này khiến kết nối dễ bị tấn công MITM.
Khi kubectl truy cập cluster, nó dùng root certificate đã lưu cùng các client
certificate để truy cập server. (Chúng được cài trong thư mục `~/.kube`).
Vì certificate của cluster thường là tự ký, bạn có thể cần cấu hình đặc biệt
để http client của bạn dùng được root certificate.

Trên một số cluster, apiserver không yêu cầu xác thực; nó có thể phục vụ trên
localhost, hoặc được bảo vệ bởi tường lửa (firewall). Không có chuẩn chung nào
cho việc này. [Kiểm soát truy cập tới API](119-controlling-access-vi.md)
mô tả cách quản trị viên cluster có thể cấu hình điều này.

## Truy cập API bằng lập trình (Programmatic access to the API)

Kubernetes chính thức hỗ trợ thư viện client cho [Go](#go-client) và [Python](#python-client).

### Go client

* Để lấy thư viện, chạy lệnh sau: `go get k8s.io/client-go@kubernetes-<kubernetes-version-number>`,
  xem [INSTALL.md](https://github.com/kubernetes/client-go/blob/master/INSTALL.md#for-the-casual-user)
  để có hướng dẫn cài đặt chi tiết. Xem
  [https://github.com/kubernetes/client-go](https://github.com/kubernetes/client-go#compatibility-matrix)
  để biết các phiên bản nào được hỗ trợ.
* Viết ứng dụng dựa trên các client của client-go. Lưu ý rằng client-go định nghĩa các đối tượng API riêng của nó,
  vì vậy nếu cần, hãy import các định nghĩa API từ client-go thay vì từ repository chính,
  ví dụ `import "k8s.io/client-go/kubernetes"` là cách đúng.

Go client có thể dùng cùng [file kubeconfig](111-kubeconfig-vi.md)
như kubectl CLI để định vị và xác thực với apiserver. Xem
[ví dụ](https://git.k8s.io/client-go/examples/out-of-cluster-client-configuration/main.go) này.

Nếu ứng dụng được triển khai dưới dạng một Pod trong cluster, hãy tham khảo [mục tiếp theo](#accessing-the-api-from-a-pod).

### Python client

Để dùng [Python client](https://github.com/kubernetes-client/python), chạy lệnh sau:
`pip install kubernetes`. Xem [trang Python Client Library](https://github.com/kubernetes-client/python)
để biết thêm các tùy chọn cài đặt khác.

Python client có thể dùng cùng [file kubeconfig](111-kubeconfig-vi.md)
như kubectl CLI để định vị và xác thực với apiserver. Xem
[ví dụ](https://github.com/kubernetes-client/python/tree/master/examples) này.

### Các ngôn ngữ khác (Other languages)

Có các [thư viện client](https://kubernetes.io/docs/reference/using-api/client-libraries/) để truy cập API từ các ngôn ngữ khác.
Xem tài liệu của từng thư viện để biết cách chúng xác thực.

## Truy cập API từ một Pod (Accessing the API from a Pod) {#accessing-the-api-from-a-pod}

Khi truy cập API từ một pod, việc định vị và xác thực với API server
có đôi chút khác biệt.

Vui lòng xem [Truy cập API từ bên trong một Pod](338-access-api-from-pod-vi.md)
để biết thêm chi tiết.

## Truy cập các service đang chạy trên cluster (Accessing services running on the cluster)

Mục trước mô tả cách kết nối tới Kubernetes API server.
Để biết thông tin về việc kết nối tới các service khác đang chạy trên một cluster Kubernetes, xem
[Truy cập các Service của cluster](369-access-cluster-services-vi.md).

## Yêu cầu chuyển hướng (Requesting redirects)

Khả năng chuyển hướng (redirect) đã bị loại bỏ sau khi ngưng hỗ trợ (deprecated). Vui lòng dùng proxy (xem bên dưới) thay thế.

## Rất nhiều loại proxy (So many proxies)

Có một số loại proxy khác nhau mà bạn có thể gặp khi dùng Kubernetes:

1. [kubectl proxy](#directly-accessing-the-rest-api):

   - chạy trên máy tính của người dùng hoặc trong một pod
   - proxy từ một địa chỉ localhost tới Kubernetes apiserver
   - client tới proxy dùng HTTP
   - proxy tới apiserver dùng HTTPS
   - định vị apiserver
   - thêm các header xác thực

1. [apiserver proxy](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster-services/#discovering-builtin-services):

   - là một bastion được tích hợp sẵn trong apiserver
   - kết nối người dùng ở bên ngoài cluster tới các cluster IP mà nếu không có nó thì có thể không truy cập được
   - chạy trong các tiến trình của apiserver
   - client tới proxy dùng HTTPS (hoặc http nếu apiserver được cấu hình như vậy)
   - proxy tới đích có thể dùng HTTP hoặc HTTPS tùy proxy lựa chọn dựa trên thông tin sẵn có
   - có thể được dùng để truy cập một Node, Pod, hoặc Service
   - thực hiện cân bằng tải khi được dùng để truy cập một Service

1. [kube proxy](https://kubernetes.io/docs/concepts/services-networking/service#ips-and-vips):

   - chạy trên mỗi node
   - proxy các giao thức UDP và TCP
   - không hiểu HTTP
   - cung cấp cân bằng tải
   - chỉ được dùng để truy cập các service

1. Một Proxy/Load-balancer đặt trước (các) apiserver:

   - sự tồn tại và cách triển khai thay đổi tùy từng cluster (ví dụ nginx)
   - nằm giữa tất cả các client và một hoặc nhiều apiserver
   - đóng vai trò bộ cân bằng tải (load balancer) nếu có nhiều apiserver.

1. Cloud Load Balancer trên các service bên ngoài:

   - được cung cấp bởi một số nhà cung cấp cloud (ví dụ AWS ELB, Google Cloud Load Balancer)
   - được tạo tự động khi Kubernetes service có type `LoadBalancer`
   - chỉ dùng UDP/TCP
   - cách triển khai thay đổi tùy nhà cung cấp cloud.

Người dùng Kubernetes thường sẽ không cần bận tâm tới bất kỳ loại nào ngoài hai loại đầu tiên.
Quản trị viên cluster thường sẽ đảm bảo các loại còn lại được thiết lập đúng.

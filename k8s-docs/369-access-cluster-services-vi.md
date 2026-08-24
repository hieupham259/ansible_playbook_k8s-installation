# Truy cập các Service đang chạy trên cluster (Access Services Running on Clusters)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster-services/>
>
> Trang này hướng dẫn cách kết nối tới các service đang chạy trên cluster Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Truy cập các service đang chạy trên cluster (Accessing services running on the cluster)

Trong Kubernetes, [node](23-nodes-vi.md), [pod](46-pods-vi.md) và
[service](82-service-vi.md) đều có IP riêng của mình. Trong nhiều trường hợp, IP của node, IP của
pod và một số IP của service trên cluster sẽ không định tuyến được (routable), nên chúng không thể
truy cập được từ một máy nằm ngoài cluster, chẳng hạn máy desktop của bạn.

### Các cách kết nối (Ways to connect)

Bạn có vài lựa chọn để kết nối tới node, pod và service từ bên ngoài cluster:

- Truy cập service qua public IP.
  - Dùng một service có type `NodePort` hoặc `LoadBalancer` để service có thể truy cập được từ bên
    ngoài cluster. Xem tài liệu về [service](82-service-vi.md) và
    [kubectl expose](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#expose).
  - Tùy vào môi trường cluster của bạn, cách này có thể chỉ expose service ra mạng nội bộ của công
    ty, hoặc có thể expose nó ra internet. Hãy cân nhắc xem service được expose có an toàn hay
    không. Bản thân nó có tự thực hiện xác thực (authentication) không?
  - Đặt các pod phía sau service. Để truy cập một pod cụ thể trong một tập replica, chẳng hạn để
    gỡ lỗi (debug), hãy gán một label duy nhất cho pod đó rồi tạo một service mới chọn (select)
    label này.
  - Trong hầu hết trường hợp, lập trình viên ứng dụng không cần truy cập trực tiếp vào node thông
    qua nodeIP của chúng.
- Truy cập service, node hoặc pod bằng Proxy Verb.
  - apiserver thực hiện xác thực (authentication) và phân quyền (authorization) trước khi truy cập
    tới service ở xa. Hãy dùng cách này nếu các service chưa đủ an toàn để expose ra internet,
    hoặc khi cần truy cập các port trên IP của node, hoặc để gỡ lỗi.
  - Proxy có thể gây ra vấn đề với một số ứng dụng web.
  - Chỉ hoạt động với HTTP/HTTPS.
  - Được mô tả [ở đây](#manually-constructing-apiserver-proxy-urls).
- Truy cập từ một node hoặc pod bên trong cluster.
  - Chạy một pod, rồi kết nối vào shell bên trong nó bằng
    [kubectl exec](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#exec).
    Từ shell đó, hãy kết nối tới các node, pod và service khác.
  - Một số cluster có thể cho phép bạn ssh vào một node trong cluster. Từ đó bạn có thể truy cập
    được các service của cluster. Đây là phương pháp không chuẩn (non-standard), nó chạy được trên
    một số cluster nhưng không chạy được trên cluster khác. Trình duyệt và các công cụ khác có thể
    có hoặc không được cài đặt sẵn. DNS của cluster có thể không hoạt động.

### Khám phá các service có sẵn (Discovering builtin services)

Thông thường, có vài service được kube-system khởi động trên cluster. Hãy lấy danh sách các service
này bằng lệnh `kubectl cluster-info`:

```shell
kubectl cluster-info
```

Kết quả sẽ tương tự như sau:

```
Kubernetes master is running at https://192.0.2.1
elasticsearch-logging is running at https://192.0.2.1/api/v1/namespaces/kube-system/services/elasticsearch-logging/proxy
kibana-logging is running at https://192.0.2.1/api/v1/namespaces/kube-system/services/kibana-logging/proxy
kube-dns is running at https://192.0.2.1/api/v1/namespaces/kube-system/services/kube-dns/proxy
grafana is running at https://192.0.2.1/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
heapster is running at https://192.0.2.1/api/v1/namespaces/kube-system/services/monitoring-heapster/proxy
```

Kết quả này cho thấy proxy-verb URL để truy cập từng service.
Ví dụ, cluster này đã bật ghi log ở cấp cluster (dùng Elasticsearch), và có thể truy cập nó tại
`https://192.0.2.1/api/v1/namespaces/kube-system/services/elasticsearch-logging/proxy/`
nếu truyền vào thông tin xác thực (credentials) phù hợp, hoặc thông qua một kubectl proxy, ví dụ tại:
`http://localhost:8080/api/v1/namespaces/kube-system/services/elasticsearch-logging/proxy/`.

> **Ghi chú:** Xem [Truy cập cluster bằng Kubernetes API](190-access-cluster-api-vi.md), mục
> "Truy cập Kubernetes API (Accessing the Kubernetes API)", để biết cách truyền thông tin xác thực
> hoặc cách dùng kubectl proxy.

#### Tự tay dựng apiserver proxy URL (Manually constructing apiserver proxy URLs) {#manually-constructing-apiserver-proxy-urls}

Như đã nói ở trên, bạn dùng lệnh `kubectl cluster-info` để lấy proxy URL của service. Để tạo các
proxy URL có kèm endpoint, hậu tố (suffix) và tham số của service, bạn nối thêm chúng vào sau proxy
URL của service:
`http://`*`kubernetes_master_address`*`/api/v1/namespaces/`*`namespace_name`*`/services/`*`[https:]service_name[:port_name]`*`/proxy`

Nếu bạn chưa đặt tên cho port của mình, bạn không cần chỉ định *port_name* trong URL. Bạn cũng có
thể dùng số hiệu port thay cho *port_name*, với cả port có tên lẫn port không tên.

Mặc định, API server proxy tới service của bạn bằng HTTP. Để dùng HTTPS, hãy thêm tiền tố `https:`
vào trước tên service:
`http://<kubernetes_master_address>/api/v1/namespaces/<namespace_name>/services/<service_name>/proxy`

Các định dạng được hỗ trợ cho phần `<service_name>` của URL là:

* `<service_name>` - proxy tới port mặc định hoặc port không tên, dùng http
* `<service_name>:<port_name>` - proxy tới port name hoặc số hiệu port được chỉ định, dùng http
* `https:<service_name>:` - proxy tới port mặc định hoặc port không tên, dùng https (chú ý dấu hai
  chấm ở cuối)
* `https:<service_name>:<port_name>` - proxy tới port name hoặc số hiệu port được chỉ định, dùng
  https

##### Ví dụ (Examples)

* Để truy cập endpoint `_search?q=user:kimchy` của service Elasticsearch, bạn sẽ dùng:

  ```
  http://192.0.2.1/api/v1/namespaces/kube-system/services/elasticsearch-logging/proxy/_search?q=user:kimchy
  ```

* Để truy cập thông tin sức khỏe của cluster Elasticsearch tại `_cluster/health?pretty=true`, bạn
  sẽ dùng:

  ```
  https://192.0.2.1/api/v1/namespaces/kube-system/services/elasticsearch-logging/proxy/_cluster/health?pretty=true
  ```

  Thông tin sức khỏe sẽ tương tự như sau:

  ```json
  {
    "cluster_name" : "kubernetes_logging",
    "status" : "yellow",
    "timed_out" : false,
    "number_of_nodes" : 1,
    "number_of_data_nodes" : 1,
    "active_primary_shards" : 5,
    "active_shards" : 5,
    "relocating_shards" : 0,
    "initializing_shards" : 0,
    "unassigned_shards" : 5
  }
  ```

* Để truy cập thông tin sức khỏe `_cluster/health?pretty=true` của service Elasticsearch qua
  *https*, bạn sẽ dùng:

  ```
  https://192.0.2.1/api/v1/namespaces/kube-system/services/https:elasticsearch-logging:/proxy/_cluster/health?pretty=true
  ```

#### Dùng trình duyệt web để truy cập các service đang chạy trên cluster (Using web browsers to access services running on the cluster)

Bạn có thể đưa một apiserver proxy URL vào thanh địa chỉ của trình duyệt. Tuy nhiên:

- Trình duyệt web thường không thể truyền token, nên bạn có thể phải dùng xác thực cơ bản
  (basic auth) bằng mật khẩu. Apiserver có thể được cấu hình để chấp nhận basic auth, nhưng cluster
  của bạn có thể lại không được cấu hình để chấp nhận basic auth.
- Một số ứng dụng web có thể không hoạt động, đặc biệt là những ứng dụng có javascript phía client
  tự dựng URL theo cách không biết tới tiền tố đường dẫn (path prefix) của proxy.

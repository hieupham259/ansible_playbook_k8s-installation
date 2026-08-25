# Truy cập các Service đang chạy trên cluster (Access Services Running on Clusters)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster-services/>
>
> Trang này hướng dẫn cách kết nối tới các service đang chạy trên cluster Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 30 — Truy cập ứng dụng trong cluster](00-ALO-TRINH-ADMIN.md#giai-đoạn-30--truy-cập-ứng-dụng-trong-cluster),
bài 4/4 — **bài cuối của giai đoạn 30 và cũng là bài cuối của lộ trình** · Phần II không có lab:
thực hành thẳng trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng
**Checkpoint** ở cuối
[mục giai đoạn 30](00-ALO-TRINH-ADMIN.md#giai-đoạn-30--truy-cập-ứng-dụng-trong-cluster). Cơ chế
của bài đã được kiểm chứng ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) phần **B6** —
cụ thể là **B6.4**, nơi bạn gọi một Service bên trong cluster qua `kubectl proxy` rồi qua apiserver
proxy.

Bài nối tiếp [190 — Truy cập cluster bằng Kubernetes API](190-access-cluster-api-vi.md) và là
**đường vào thứ ba** trong ba đường mà Checkpoint giai đoạn 30 bắt phân biệt. Đọc xong bài này bạn
phải xếp được cả ba cạnh nhau: **Service** (bài [370](370-service-access-application-cluster-vi.md))
là lối vào thường trực cho người dùng thật; **`kubectl port-forward`** (bài
[366](366-port-forward-vi.md)) là tiến trình gỡ lỗi phục vụ đúng máy đang chạy lệnh; còn
**apiserver proxy** ở bài này là một URL trên chính API server, đi qua xác thực và phân quyền của
cluster.

**Phải hiểu ở lần đọc này:**

- Đoạn mở đầu mục *Truy cập các service đang chạy trên cluster*: IP của node, của Pod và một số IP
  của Service **không định tuyến được** từ máy ngoài cluster. Toàn bộ phần còn lại của bài tồn tại
  chỉ để vòng qua sự thật đó.
- Mục *Các cách kết nối* chia làm ba nhóm: qua **public IP** bằng Service `NodePort`/`LoadBalancer`;
  qua **Proxy Verb** của apiserver — apiserver **xác thực và phân quyền trước** khi chạm tới service
  ở xa, nhưng **chỉ hoạt động với HTTP/HTTPS** và có thể gây vấn đề với một số ứng dụng web; và
  **từ bên trong cluster** bằng `kubectl exec` vào một Pod, hoặc ssh vào node — cách mà bài gọi
  thẳng là **không chuẩn**. Cũng ở mục này có mẹo gỡ lỗi đáng nhớ: muốn chạm đúng **một** Pod trong
  tập replica thì gán cho nó một label duy nhất rồi tạo Service chọn label đó.
- Mục *Khám phá các service có sẵn*: `kubectl cluster-info` in ra **proxy-verb URL** của từng
  service; các URL đó chỉ dùng được khi truyền credentials phù hợp, hoặc khi đi qua `kubectl proxy`.
- Mục *Tự tay dựng apiserver proxy URL*: khuôn
  `/api/v1/namespaces/<namespace>/services/[https:]<service>[:port_name]/proxy`, nối endpoint và
  tham số của service vào **sau** phần `/proxy`. Bốn dạng của phần `<service_name>`, ý nghĩa của
  tiền tố `https:` và của dấu hai chấm ở cuối, cùng mặc định: apiserver proxy tới service bằng
  **HTTP** nếu không khai gì.
- Mục *Dùng trình duyệt web để truy cập các service*: hai giới hạn thật — trình duyệt thường
  **không truyền được token**, và ứng dụng web có JavaScript tự dựng URL mà **không biết tới path
  prefix** của proxy sẽ hỏng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Danh sách service trong output ví dụ `kubectl cluster-info` — elasticsearch-logging, kibana-logging, grafana, heapster — và cách gọi "Kubernetes master" | đó là addon của cluster viết tài liệu, cluster lab do kubeadm dựng không có chúng | **không áp dụng cho cluster lab**: [Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) phần B0.3 đã xác nhận addon kiểu này trên cluster của bạn là CoreDNS với Service `kube-dns`; addon quan sát thì cài ở [Lab 11a](labs/LAB-11A-OBSERVABILITY.md) |
| Mọi ví dụ URL Elasticsearch (`_search?q=user:kimchy`, `_cluster/health?pretty=true`) và khối JSON kết quả | Elasticsearch chỉ là ứng dụng làm ví dụ, không phải nội dung của proxy URL | thứ cần nhớ là **khuôn URL**, đã kiểm chứng bằng Service HTTP của bạn ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) phần B6.4 |
| Cách truyền credentials vào các proxy URL — chính bài trỏ sang chỗ khác | đây là nội dung xác thực với API server, không phải nội dung của bài này | bài [190](190-access-cluster-api-vi.md), mục *Truy cập Kubernetes API*, đã đọc ở [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) |
| Nhánh xác thực cơ bản (basic auth) cho trình duyệt | bài đã nói ngay là cluster của bạn có thể không được cấu hình chấp nhận basic auth | **không áp dụng cho cluster lab**: đường xác thực của bạn là kubeconfig và token, đã học ở bài [119](119-controlling-access-vi.md) và thực hành ở [Lab 9a](labs/LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md) |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 30 — và đây là lần tự
kiểm tra cuối cùng của lộ trình:

1. Điều gì về IP của node, Pod và Service khiến máy desktop của bạn không gọi thẳng vào được, và ba
   nhóm cách kết nối mà bài đưa ra lần lượt vòng qua nó bằng cách nào?
2. **Câu bẫy.** Cả `kubectl port-forward` (bài [366](366-port-forward-vi.md)) lẫn apiserver proxy
   đều cho bạn chạm tới workload mà không phải mở thêm port nào ra ngoài. Vậy hai đường đó có phải
   là hai cách gõ của cùng một cơ chế không? Ai kiểm soát truy cập, và mỗi đường bị giới hạn ở
   giao thức nào?
3. Ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) phần B6.4 bạn đã gọi Service `web`
   (port tên `http`, namespace `lab-5b`) qua URL
   `/api/v1/namespaces/lab-5b/services/web:http/proxy/`. Dựa theo khuôn của bài, hãy sửa URL đó cho
   trường hợp Service nghe **HTTPS** trên port mặc định không đặt tên. Và vì sao khi gọi qua
   `kubectl proxy` trên `lab-k8s-master` thì `curl` không phải kèm token nào?
4. Một ứng dụng web bình thường đặt sau apiserver proxy có thể hỏng vì hai lý do bài nêu. Hai lý do
   đó là gì?
5. Checkpoint giai đoạn 30 bắt giải thích ba đường vào cluster. Với mỗi việc sau, chọn đúng một
   đường và nói vì sao: (a) mở ứng dụng cho người dùng thật; (b) gọi thử một Pod chưa có Service để
   xem nó có phản hồi không; (c) từ máy trạm gọi API HTTP của một Service nội bộ mà không muốn mở
   bất kỳ port nào ra ngoài.

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Trong nhiều trường hợp các IP đó không định tuyến được** từ máy ngoài cluster. Ba nhóm cách
   vòng qua: **public IP** — dùng Service `NodePort` hoặc `LoadBalancer` để chính service có địa chỉ
   gọi được từ ngoài; **Proxy Verb** — mượn API server làm điểm vào, apiserver xác thực và phân
   quyền rồi mới chuyển tiếp; **từ bên trong** — `kubectl exec` vào một Pod rồi gọi từ đó, hoặc ssh
   vào node, cách mà bài nói rõ là **không chuẩn** và có thể không chạy trên cluster khác.
2. **Không phải cùng một cơ chế.** `kubectl port-forward` là một **tiến trình đang chạy trên máy
   bạn**, mở một port cục bộ và chuyển tiếp vào **một Pod**; nó **chỉ hiện thực cho TCP**, và quyền
   được API server thực thi trên subresource `pods/portforward`. Apiserver proxy là một **URL trên
   chính API server**, nhắm tới **Service, node hoặc Pod**; apiserver xác thực và phân quyền trước
   khi chạm tới đích, nhưng nó **chỉ hoạt động với HTTP/HTTPS** — thứ gì không phải HTTP thì đường
   này chịu. Điểm chung duy nhất: cả hai đều đi qua API server nên đều đòi credentials, và cả hai
   đều không mở thêm cửa nào trên node.
3. URL thành **`/api/v1/namespaces/lab-5b/services/https:web:/proxy/`** — thêm tiền tố `https:`
   trước tên service, và **giữ dấu hai chấm ở cuối** vì đây là port mặc định/không tên. (Không khai
   gì thì apiserver proxy tới service bằng HTTP, nên phải nói rõ mới thành HTTPS.) Còn `curl` không
   cần token vì **`kubectl proxy` mới là thứ nói chuyện với apiserver**: nó nhận HTTP thuần từ
   localhost rồi tự gắn thông tin xác thực từ kubeconfig của bạn — đúng điều bạn đã đo ở
   [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) phần B6.4. Hệ quả: ai chạm được cổng đó
   đang phát request với danh nghĩa của bạn.
4. Thứ nhất, **trình duyệt thường không truyền được token**, nên có thể phải dùng basic auth mà
   cluster lại không được cấu hình để chấp nhận. Thứ hai, **JavaScript phía client tự dựng URL mà
   không biết tới path prefix của proxy** — ứng dụng sẽ gọi vào đường dẫn sai và hỏng.
5. (a) **Service** — bài [370](370-service-access-application-cluster-vi.md): lối vào thường trực,
   không đòi kubeconfig, ai tới được địa chỉ node là dùng được. (b) **`kubectl port-forward`** — bài
   [366](366-port-forward-vi.md): nhắm thẳng vào một Pod, không cần tạo object nào, đúng bản chất
   công cụ gỡ lỗi. Cách còn lại mà chính bài này gợi ý là gán label duy nhất cho Pod đó rồi tạo
   Service chọn label ấy. (c) **apiserver proxy** — chỉ là một URL trên API server đã có sẵn, hợp
   với HTTP, và mọi request đều qua xác thực cùng phân quyền của cluster.

</details>

Đây là **chặng cuối**. Câu nào chưa trả lời được thì quay lại đúng mục tương ứng, rồi làm
**Checkpoint** ở cuối
[mục giai đoạn 30](00-ALO-TRINH-ADMIN.md#giai-đoạn-30--truy-cập-ứng-dụng-trong-cluster): expose một
Deployment bằng NodePort và gọi từ máy host, `kubectl port-forward` tới một Pod không có Service, và
giải thích ba đường vào cluster khác nhau ở đâu.

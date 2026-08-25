# Thiết lập dịch vụ Konnectivity (Set up Konnectivity service)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/extend-kubernetes/setup-konnectivity/>
>
> Dịch vụ Konnectivity cung cấp một proxy ở tầng TCP cho giao tiếp từ control plane tới
> cluster.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 28 — Mở rộng Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes), bài 10/11 ·
Phần II không có lab riêng: bạn thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) rồi tự chấm bằng **Checkpoint** của
[giai đoạn 28](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes).
Bài nối tiếp [24](24-control-plane-node-communication-vi.md) ở
[giai đoạn 1](00-ALO-TRINH-ADMIN.md#giai-đoạn-1--mô-hình-kubernetes) — bài đã dạy chiều giao tiếp
nào đi qua API server và chiều nào không.

**Bài cuối của cụm ba bài về trung gian, và là bài dễ xếp nhầm chỗ nhất.** Hai bài trước —
[379](379-http-proxy-access-api-vi.md) và [382](382-socks5-proxy-access-api-vi.md) — là công cụ
**của client**, trung gian nằm trên máy bạn, chiều đi là *client → API server*. Bài này ngược
lại hoàn toàn: câu đầu tiên nói rõ Konnectivity là proxy tầng TCP cho giao tiếp **từ control
plane tới cluster**. `kubectl` của bạn không biết nó tồn tại và không cấu hình gì cho nó. Đọc
xong hãy gọi tên được ba đường một cách tách bạch, vì Checkpoint của giai đoạn không cho bạn nhìn
tài liệu.

**Phải hiểu ở lần đọc này:**

- Chiều đi và tầng làm việc, ngay câu mở đầu: một **proxy ở tầng TCP** cho giao tiếp **từ control
  plane tới cluster**. Không phải đường cho client vào API server.
- Kiến trúc hai nửa: **Konnectivity server** triển khai trên control plane node — manifest mẫu là
  một Pod `hostNetwork: true` chạy dạng static Pod — và **Konnectivity agent** chạy trong
  cluster. Bài ghi rõ trong comment của manifest agent: có thể triển khai agent dạng Deployment,
  và **không nhất thiết mỗi node một agent**.
- `EgressSelectorConfiguration` là chỗ khai báo lưu lượng nào đi qua đường hầm: trường
  `egressSelections[].name` nhận `cluster`, `etcd` hoặc `controlplane`; bài chọn `cluster` vì
  muốn kiểm soát lưu lượng hướng tới cluster. API Server được trỏ vào file đó bằng flag
  `--egress-selector-config-file`.
- Ba chỗ **hai phía phải khai trùng nhau**, chính bài đánh dấu bằng comment "giá trị này phải
  khớp": `proxyProtocol: GRPC` ứng với `--mode=grpc` của server; `uds.udsName` ứng với
  `--uds-name`; và nếu đi bằng UDS thì kube-apiserver còn phải mount đúng thư mục chứa socket
  bằng `hostPath`. Lệch một chỗ là hai nửa không nối được.
- Danh tính của từng nửa: server dùng một certificate X.509 `CN=system:konnectivity-server` ký
  bằng CA của cluster, cộng kubeconfig riêng và ClusterRoleBinding tới `system:auth-delegator`;
  agent dùng ServiceAccount `konnectivity-agent` với **projected token mang
  `audience: system:konnectivity-server`** — đó là lý do bước 1 đòi bật
  [Service Account Token Volume Projection](279-configure-service-account-vi.md#serviceaccount-token-volume-projection).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nhánh `transport` dùng `tcp` thay cho `uds`, và cấu hình TLS mà nó kéo theo | bài chỉ nhắc đúng một câu là bạn sẽ phải thiết lập TLS để bảo vệ transport TCP, rồi không hướng dẫn gì thêm; manifest mẫu của chính bài đặt server cùng máy với API Server nên đi bằng UDS | phát hành và quản lý certificate của cluster là [giai đoạn 18](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ), bài [219](219-kubeadm-certs-vi.md) |
| Từng cờ vận hành trong hai manifest — `--admin-port`, `--health-port`, `--server-port`, `livenessProbe`, danh sách `volumeMounts` | đó là chi tiết của bản hiện thực tham chiếu `apiserver-network-proxy`, và image của nó nằm ngoài [baseline Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) | Checkpoint của [giai đoạn 28](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes) không hỏi tới đây; ở lần đọc này chỉ cần đọc ra **ai nối với ai qua port nào** — agent tìm server ở `--proxy-server-port=8132`, ứng với `--agent-port=8132` phía server |
| Việc thật sự gắn Konnectivity vào cluster của bạn — thêm flag `--egress-selector-config-file` và khối volume vào kube-apiserver | đó là sửa static pod manifest của control plane đang chạy; hiểu được cờ đó làm gì là đủ cho lần đọc này, còn tự tay đổi nó là chuyện khác | [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |

---

Dịch vụ Konnectivity cung cấp một proxy ở tầng TCP cho giao tiếp từ control plane tới
cluster.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/).

## Cấu hình dịch vụ Konnectivity (Configure the Konnectivity service)

Các bước sau đây cần một cấu hình egress, ví dụ như sau:

```yaml
apiVersion: apiserver.k8s.io/v1beta1
kind: EgressSelectorConfiguration
egressSelections:
# Vì chúng ta muốn kiểm soát lưu lượng đi ra (egress traffic) hướng tới cluster, nên ta
# dùng "cluster" làm tên. Các giá trị khác được hỗ trợ là "etcd" và "controlplane".
- name: cluster
  connection:
    # Mục này điều khiển giao thức giữa API Server và Konnectivity server.
    # Các giá trị được hỗ trợ là "GRPC" và "HTTPConnect". Không có khác biệt nào mà
    # người dùng cuối nhìn thấy được giữa hai chế độ này. Bạn cần đặt Konnectivity
    # server chạy ở cùng chế độ.
    proxyProtocol: GRPC
    transport:
      # Mục này điều khiển transport mà API Server dùng để giao tiếp với Konnectivity
      # server. Nên dùng UDS nếu Konnectivity server nằm trên cùng một máy với API
      # Server. Bạn cần cấu hình Konnectivity server lắng nghe trên đúng UDS socket đó.
      # Transport còn lại được hỗ trợ là "tcp". Khi dùng nó, bạn sẽ phải thiết lập cấu
      # hình TLS để bảo vệ transport TCP.
      uds:
        udsName: /etc/kubernetes/konnectivity-server/konnectivity-server.socket
```

Bạn cần cấu hình API Server để nó sử dụng dịch vụ Konnectivity và điều hướng lưu lượng mạng
tới các node của cluster:

1. Hãy chắc chắn rằng tính năng
   [Chiếu token của ServiceAccount vào volume (Service Account Token Volume Projection)](279-configure-service-account-vi.md#serviceaccount-token-volume-projection)
   đã được bật trong cluster của bạn. Tính năng này được bật mặc định kể từ Kubernetes v1.20.
2. Tạo một file cấu hình egress, chẳng hạn
   `admin/konnectivity/egress-selector-configuration.yaml`.
3. Đặt flag `--egress-selector-config-file` của API Server trỏ tới đường dẫn của file cấu
   hình egress dành cho API Server.
4. Nếu bạn dùng kết nối UDS, hãy thêm phần cấu hình volume vào kube-apiserver:

   ```yaml
   spec:
     containers:
       volumeMounts:
       - name: konnectivity-uds
         mountPath: /etc/kubernetes/konnectivity-server
         readOnly: false
     volumes:
     - name: konnectivity-uds
       hostPath:
         path: /etc/kubernetes/konnectivity-server
         type: DirectoryOrCreate
   ```

Hãy tạo hoặc xin cấp một certificate và một kubeconfig cho konnectivity-server. Ví dụ, bạn có
thể dùng công cụ dòng lệnh OpenSSL để phát hành một certificate X.509, sử dụng certificate CA
của cluster là `/etc/kubernetes/pki/ca.crt` lấy từ một control-plane host.

```bash
openssl req -subj "/CN=system:konnectivity-server" -new -newkey rsa:2048 -noenc -out konnectivity.csr -keyout konnectivity.key
openssl x509 -req -in konnectivity.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out konnectivity.crt -days 375 -sha256
SERVER=$(kubectl config view -o jsonpath='{.clusters..server}')
kubectl --kubeconfig /etc/kubernetes/konnectivity-server.conf config set-credentials system:konnectivity-server --client-certificate konnectivity.crt --client-key konnectivity.key --embed-certs=true
kubectl --kubeconfig /etc/kubernetes/konnectivity-server.conf config set-cluster kubernetes --server "$SERVER" --certificate-authority /etc/kubernetes/pki/ca.crt --embed-certs=true
kubectl --kubeconfig /etc/kubernetes/konnectivity-server.conf config set-context system:konnectivity-server@kubernetes --cluster kubernetes --user system:konnectivity-server
kubectl --kubeconfig /etc/kubernetes/konnectivity-server.conf config use-context system:konnectivity-server@kubernetes
rm -f konnectivity.crt konnectivity.key konnectivity.csr
```

Tiếp theo, bạn cần triển khai Konnectivity server và các Konnectivity agent.
[kubernetes-sigs/apiserver-network-proxy](https://github.com/kubernetes-sigs/apiserver-network-proxy)
là một bản hiện thực tham chiếu (reference implementation).

Hãy triển khai Konnectivity server trên control plane node của bạn. Manifest
`konnectivity-server.yaml` được cung cấp dưới đây giả định rằng các thành phần Kubernetes
được triển khai dưới dạng static Pod trong cluster của bạn. Nếu không phải như vậy, bạn có
thể triển khai Konnectivity server dưới dạng một DaemonSet.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: konnectivity-server
  namespace: kube-system
spec:
  priorityClassName: system-cluster-critical
  hostNetwork: true
  containers:
  - name: konnectivity-server-container
    image: registry.k8s.io/kas-network-proxy/proxy-server:v0.0.37
    command: ["/proxy-server"]
    args: [
            "--logtostderr=true",
            # Giá trị này phải khớp với giá trị đã đặt trong egressSelectorConfiguration.
            "--uds-name=/etc/kubernetes/konnectivity-server/konnectivity-server.socket",
            "--delete-existing-uds-file",
            # Hai dòng dưới đây giả định rằng Konnectivity server được triển khai trên
            # cùng một máy với apiserver, và certificate cùng key của API Server nằm
            # đúng ở vị trí được chỉ định.
            "--cluster-cert=/etc/kubernetes/pki/apiserver.crt",
            "--cluster-key=/etc/kubernetes/pki/apiserver.key",
            # Giá trị này phải khớp với giá trị đã đặt trong egressSelectorConfiguration.
            "--mode=grpc",
            "--server-port=0",
            "--agent-port=8132",
            "--admin-port=8133",
            "--health-port=8134",
            "--agent-namespace=kube-system",
            "--agent-service-account=konnectivity-agent",
            "--kubeconfig=/etc/kubernetes/konnectivity-server.conf",
            "--authentication-audience=system:konnectivity-server"
            ]
    livenessProbe:
      httpGet:
        scheme: HTTP
        host: 127.0.0.1
        port: 8134
        path: /healthz
      initialDelaySeconds: 30
      timeoutSeconds: 60
    ports:
    - name: agentport
      containerPort: 8132
      hostPort: 8132
    - name: adminport
      containerPort: 8133
      hostPort: 8133
    - name: healthport
      containerPort: 8134
      hostPort: 8134
    volumeMounts:
    - name: k8s-certs
      mountPath: /etc/kubernetes/pki
      readOnly: true
    - name: kubeconfig
      mountPath: /etc/kubernetes/konnectivity-server.conf
      readOnly: true
    - name: konnectivity-uds
      mountPath: /etc/kubernetes/konnectivity-server
      readOnly: false
  volumes:
  - name: k8s-certs
    hostPath:
      path: /etc/kubernetes/pki
  - name: kubeconfig
    hostPath:
      path: /etc/kubernetes/konnectivity-server.conf
      type: FileOrCreate
  - name: konnectivity-uds
    hostPath:
      path: /etc/kubernetes/konnectivity-server
      type: DirectoryOrCreate
```

Sau đó, hãy triển khai các Konnectivity agent trong cluster của bạn:

```yaml
apiVersion: apps/v1
# Ngoài ra, bạn cũng có thể triển khai các agent dưới dạng Deployment. Không nhất thiết
# phải có một agent trên mỗi node.
kind: DaemonSet
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    k8s-app: konnectivity-agent
  namespace: kube-system
  name: konnectivity-agent
spec:
  selector:
    matchLabels:
      k8s-app: konnectivity-agent
  template:
    metadata:
      labels:
        k8s-app: konnectivity-agent
    spec:
      priorityClassName: system-cluster-critical
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      containers:
        - image: us.gcr.io/k8s-artifacts-prod/kas-network-proxy/proxy-agent:v0.0.37
          name: konnectivity-agent
          command: ["/proxy-agent"]
          args: [
                  "--logtostderr=true",
                  "--ca-cert=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt",
                  # Vì konnectivity server chạy với hostNetwork=true, đây chính là
                  # địa chỉ IP của máy master.
                  "--proxy-server-host=35.225.206.7",
                  "--proxy-server-port=8132",
                  "--admin-server-port=8133",
                  "--health-server-port=8134",
                  "--service-account-token-path=/var/run/secrets/tokens/konnectivity-agent-token"
                  ]
          volumeMounts:
            - mountPath: /var/run/secrets/tokens
              name: konnectivity-agent-token
          livenessProbe:
            httpGet:
              port: 8134
              path: /healthz
            initialDelaySeconds: 15
            timeoutSeconds: 15
      serviceAccountName: konnectivity-agent
      volumes:
        - name: konnectivity-agent-token
          projected:
            sources:
              - serviceAccountToken:
                  path: konnectivity-agent-token
                  audience: system:konnectivity-server
```

Cuối cùng, nếu RBAC được bật trong cluster của bạn, hãy tạo các quy tắc RBAC tương ứng:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:konnectivity-server
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: system:konnectivity-server
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: konnectivity-agent
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
```

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 28:

1. **Câu bẫy.** Ba bài liên tiếp của giai đoạn này đều nói về việc đi qua trung gian. Konnectivity
   phục vụ chiều nào, trung gian của nó đặt ở đâu, và `kubectl` trên máy bạn phải cấu hình gì để
   dùng nó? So sánh với `kubectl proxy` của bài [379](379-http-proxy-access-api-vi.md) và đường
   hầm SOCKS5 của bài [382](382-socks5-proxy-access-api-vi.md).
2. Konnectivity gồm hai nửa nào, mỗi nửa chạy ở đâu, và có bắt buộc mỗi node phải có một agent
   không?
3. Bạn khai `proxyProtocol: GRPC` và `uds.udsName: /etc/kubernetes/konnectivity-server/konnectivity-server.socket`
   trong `EgressSelectorConfiguration`. Phía Konnectivity server phải khai gì cho khớp, và ngoài
   hai giá trị đó thì kube-apiserver còn phải được sửa thêm gì khi đi bằng UDS?
4. Trên cluster lab, control plane là `lab-k8s-master` còn workload chạy ở `lab-k8s-worker1` và
   `lab-k8s-worker2`. Nếu dựng Konnectivity ở đây thì server nằm trên VM nào, agent chạy dạng gì,
   và agent lấy danh tính ở đâu để server chấp nhận nó? Nếu Service Account Token Volume
   Projection chưa bật thì hỏng ở chỗ nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Konnectivity phục vụ chiều **từ control plane tới cluster**, ở **tầng TCP**, và trung gian của
   nó nằm **bên trong cluster**: server trên control plane node, agent trong cluster. `kubectl`
   của bạn **không cấu hình gì cả** — nó thậm chí không biết Konnectivity có tồn tại. Đây đúng
   là chỗ dễ nhầm: hai bài kia là công cụ **của client** cho chiều *client → API server*, với
   trung gian nằm **trên máy bạn** — `kubectl proxy` là một tiến trình cục bộ phơi ra endpoint
   HTTP, còn SOCKS5 là đường hầm SSH mà bạn trỏ `kubectl` vào bằng `HTTPS_PROXY` hoặc
   `proxy-url`. **Ba đường không thay thế nhau và không cùng chiều.**
2. **Konnectivity server** triển khai trên control plane node — manifest mẫu chạy nó dạng static
   Pod với `hostNetwork: true` — và **Konnectivity agent** chạy trong cluster, manifest mẫu dùng
   DaemonSet. Trả lời vế sau: **không bắt buộc.** Chính comment trong manifest agent nói rằng
   bạn cũng có thể triển khai agent dưới dạng Deployment, và không nhất thiết phải có một agent
   trên mỗi node.
3. Phía server phải khai **`--mode=grpc`** cho khớp với `proxyProtocol: GRPC`, và **`--uds-name`
   đúng bằng đường dẫn socket** đã ghi trong `udsName` — hai chỗ này chính bài đánh dấu bằng
   comment "giá trị này phải khớp với giá trị đã đặt trong egressSelectorConfiguration". Ngoài
   ra, kube-apiserver phải được trỏ vào file cấu hình bằng flag
   **`--egress-selector-config-file`**, và khi dùng UDS thì phải **thêm khối volume**: một
   `hostPath` tới `/etc/kubernetes/konnectivity-server` kèm `volumeMounts` tương ứng, để API
   Server nhìn thấy được socket.
4. Server nằm trên **`lab-k8s-master`** — đó là control plane node duy nhất, và manifest mẫu giả
   định các thành phần Kubernetes chạy dạng static Pod, đúng như cluster kubeadm của bạn. Agent
   chạy dạng **DaemonSet trong `kube-system`** theo manifest mẫu (hoặc Deployment, vì không bắt
   buộc mỗi node một agent). Agent lấy danh tính từ **ServiceAccount `konnectivity-agent`**, qua
   một **projected `serviceAccountToken` có `audience: system:konnectivity-server`** — khớp với
   `--authentication-audience=system:konnectivity-server` mà server khai. Nếu Service Account
   Token Volume Projection chưa bật thì **volume `konnectivity-agent-token` không sinh ra được
   token đúng audience**, nên agent không chứng minh được danh tính với server; đó là lý do bài
   đặt việc kiểm tính năng này làm **bước 1**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

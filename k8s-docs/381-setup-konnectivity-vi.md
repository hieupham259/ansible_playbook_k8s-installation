# Thiết lập dịch vụ Konnectivity (Set up Konnectivity service)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/extend-kubernetes/setup-konnectivity/>
>
> Dịch vụ Konnectivity cung cấp một proxy ở tầng TCP cho giao tiếp từ control plane tới
> cluster.

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

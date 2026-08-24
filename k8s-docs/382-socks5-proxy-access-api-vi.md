# Dùng SOCKS5 Proxy để truy cập Kubernetes API (Use a SOCKS5 Proxy to Access the Kubernetes API)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/extend-kubernetes/socks5-proxy-access-api/>
>
> Trang này hướng dẫn cách dùng một SOCKS5 proxy để truy cập API của một cluster Kubernetes
> ở xa.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Trang này hướng dẫn cách dùng một SOCKS5 proxy để truy cập API của một cluster Kubernetes ở xa.
Cách làm này hữu ích khi cluster mà bạn muốn truy cập không expose API của nó trực tiếp ra
internet công cộng.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn v1.24. Để kiểm tra phiên bản, nhập
`kubectl version`.

Bạn cần có phần mềm SSH client (công cụ `ssh`), và một dịch vụ SSH đang chạy trên server ở xa.
Bạn phải đăng nhập được vào dịch vụ SSH trên server ở xa đó.

## Bối cảnh của bài thực hành (Task context)

> **Ghi chú:**
> Ví dụ này tạo đường hầm (tunnel) cho lưu lượng bằng SSH, trong đó SSH client và SSH server
> đóng vai trò một SOCKS proxy. Bạn cũng có thể dùng bất kỳ loại proxy
> [SOCKS5](https://en.wikipedia.org/wiki/SOCKS#SOCKS5) nào khác.

Hình 1 mô tả những gì bạn sẽ đạt được trong bài thực hành này.

* Bạn có một máy client, được gọi là *local* trong các bước phía sau, đây là nơi bạn sẽ tạo
  các request để nói chuyện với Kubernetes API.
* Kubernetes server/API được đặt trên một server ở xa.
* Bạn sẽ dùng phần mềm SSH client và SSH server để tạo một đường hầm SOCKS5 an toàn giữa máy
  local và server ở xa. Lưu lượng HTTPS giữa client và Kubernetes API sẽ đi qua đường hầm
  SOCKS5 này, và bản thân đường hầm SOCKS5 lại được truyền qua SSH.

```mermaid
graph LR;

  subgraph local[Local client machine]
  client([client])-. local <br> traffic .->  local_ssh[Local SSH <br> SOCKS5 proxy];
  end
  local_ssh[SSH <br>SOCKS5 <br> proxy]-- SSH Tunnel -->sshd
  
  subgraph remote[Remote server]
  sshd[SSH <br> server]-- local traffic -->service1;
  end
  client([client])-. proxied HTTPs traffic <br> going through the proxy .->service1[Kubernetes API];

  classDef plain fill:#ddd,stroke:#fff,stroke-width:4px,color:#000;
  classDef k8s fill:#326ce5,stroke:#fff,stroke-width:4px,color:#fff;
  classDef cluster fill:#fff,stroke:#bbb,stroke-width:2px,color:#326ce5;
  class ingress,service1,service2,pod1,pod2,pod3,pod4 k8s;
  class client plain;
  class cluster cluster;
```

*Hình 1. Các thành phần trong hướng dẫn SOCKS5.*

## Dùng ssh để tạo một SOCKS5 proxy (Using ssh to create a SOCKS5 proxy)

Lệnh sau khởi động một SOCKS5 proxy giữa máy client của bạn và SOCKS server ở xa:

```shell
# Đường hầm SSH tiếp tục chạy ở foreground sau khi bạn chạy lệnh này
ssh -D 1080 -q -N username@kubernetes-remote-server.example
```

SOCKS5 proxy cho phép bạn kết nối tới API server của cluster dựa trên cấu hình sau:
* `-D 1080`: mở một SOCKS proxy trên port local :1080.
* `-q`: chế độ im lặng (quiet mode). Làm cho phần lớn các thông báo cảnh báo và chẩn đoán bị
  chặn lại, không hiển thị.
* `-N`: Không thực thi lệnh nào trên máy ở xa. Hữu ích khi chỉ cần forward port.
* `username@kubernetes-remote-server.example`: SSH server ở xa mà đằng sau nó cluster
  Kubernetes đang chạy (ví dụ: một bastion host).

## Cấu hình phía client (Client configuration)

Để truy cập Kubernetes API server qua proxy, bạn phải hướng dẫn `kubectl` gửi các truy vấn
qua `SOCKS` proxy mà chúng ta đã tạo ở trên. Hãy làm điều này bằng cách đặt biến môi trường
phù hợp, hoặc qua thuộc tính `proxy-url` trong file kubeconfig. Dùng biến môi trường:

```shell
export HTTPS_PROXY=socks5://localhost:1080
```

Để luôn dùng thiết lập này cho một context `kubectl` cụ thể, hãy khai báo thuộc tính
`proxy-url` trong mục `cluster` tương ứng bên trong file `~/.kube/config`. Ví dụ:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LRMEMMW2 # được rút gọn cho dễ đọc
    server: https://<API_SERVER_IP_ADDRESS>:6443  # server "Kubernetes API", nói cách khác là địa chỉ IP của kubernetes-remote-server.example
    proxy-url: socks5://localhost:1080   # "SSH SOCKS5 proxy" trong sơ đồ ở trên
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: LS0tLS1CR== # được rút gọn cho dễ đọc
    client-key-data: LS0tLS1CRUdJT=      # được rút gọn cho dễ đọc
```

Sau khi đã tạo đường hầm bằng lệnh ssh nêu ở trên, và đã định nghĩa biến môi trường hoặc
thuộc tính `proxy-url`, bạn có thể tương tác với cluster của mình qua proxy đó. Ví dụ:

```shell
kubectl get pods
```

```console
NAMESPACE     NAME                                     READY   STATUS      RESTARTS   AGE
kube-system   coredns-85cb69466-klwq8                  1/1     Running     0          5m46s
```

> **Ghi chú:**
> - Trước `kubectl` 1.24, hầu hết các lệnh `kubectl` đều hoạt động được khi dùng socks proxy,
>   ngoại trừ `kubectl exec`.
> - `kubectl` hỗ trợ cả biến môi trường `HTTPS_PROXY` lẫn `https_proxy`. Các biến này cũng
>   được những chương trình khác có hỗ trợ SOCKS sử dụng, chẳng hạn `curl`. Vì vậy trong một
>   số trường hợp, sẽ tốt hơn nếu bạn định nghĩa biến môi trường ngay trên dòng lệnh:
>   ```shell
>   HTTPS_PROXY=socks5://localhost:1080 kubectl get pods
>   ```
> - Khi dùng `proxy-url`, proxy chỉ được áp dụng cho context `kubectl` tương ứng, trong khi
>   biến môi trường sẽ ảnh hưởng tới mọi context.
> - Hostname của k8s API server có thể được bảo vệ thêm khỏi rò rỉ DNS (DNS leakage) bằng
>   cách dùng tên giao thức `socks5h` thay cho `socks5` phổ biến hơn được nêu ở trên. Trong
>   trường hợp đó, `kubectl` sẽ yêu cầu proxy server (chẳng hạn một ssh bastion) phân giải tên
>   miền của k8s API server, thay vì phân giải nó trên chính hệ thống đang chạy `kubectl`.
>   Cũng lưu ý rằng với `socks5h`, một URL k8s API server như `https://localhost:6443/api`
>   không trỏ tới máy client local của bạn. Thay vào đó, nó trỏ tới `localhost` theo cách hiểu
>   của proxy server (ví dụ ssh bastion).

## Dọn dẹp (Clean up)

Dừng tiến trình port-forwarding của ssh bằng cách nhấn `CTRL+C` trên terminal nơi nó đang chạy.

Gõ `unset https_proxy` trong một terminal để ngừng chuyển tiếp lưu lượng http qua proxy.

## Đọc thêm (Further reading)

* [OpenSSH remote login client](https://man.openbsd.org/ssh)

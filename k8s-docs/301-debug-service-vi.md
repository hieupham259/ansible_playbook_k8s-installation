# Gỡ lỗi Service (Debug Services)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối, CP9 — Xử lý sự cố](00-ALO-TRINH-ADMIN.md#cp9--xử-lý-sự-cố),
bài 6/10 · Các trang CP không có lab riêng: thực hành trực tiếp trên cluster lab ở mốc
`04-metrics-ready` (xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là bài xương sống của CP9: một quy trình **loại trừ lần từ Service về Pod**. Giá trị của
bài không nằm ở từng lệnh — phần lớn bạn đã biết từ giai đoạn 5 — mà ở **thứ tự** kiểm tra:
mỗi mục `##` khẳng định xong một tầng rồi mới được phép nghi ngờ tầng kế tiếp. Cách đọc tốt
nhất là vừa đọc vừa chạy trên cluster lab với chính Deployment `hostnames` của bài.

**Phải hiểu ở lần đọc này:**

- **Trình tự loại trừ** của quy trình: Pod có phục vụ thật không (gọi thẳng Pod IP) → Service
  có tồn tại → DNS phân giải được → gọi qua Service IP → định nghĩa Service đúng chưa →
  EndpointSlice có endpoint → gọi lại thẳng Pod một lần nữa → cuối cùng mới nghi ngờ
  kube-proxy. Đi hết chuỗi mà vẫn hỏng thì mới tới bước "tìm trợ giúp".
- Mọi phép thử phải đứng **từ trong một Pod** (busybox tạm bằng `kubectl run -it --rm`), vì
  tên DNS của Service chỉ phân giải được nhờ `/etc/resolv.conf` mà cluster bơm vào Pod; đứng
  từ node thì phải chỉ định tên đầy đủ kèm IP của DNS Service.
- Cách **khoanh vùng lỗi DNS bằng ba mức tên**: `hostnames` → `hostnames.default` →
  `hostnames.default.svc.cluster.local`, và ba dòng `nameserver` / `search` / `options ndots:5`
  trong `/etc/resolv.conf` quyết định mức nào phân giải được. So sánh thêm với
  `nslookup kubernetes.default` để biết lỗi thuộc riêng Service của bạn hay của cả hệ DNS.
- **Checklist đọc lại định nghĩa Service**: port cần truy cập có trong `spec.ports[]`,
  `targetPort` đúng port Pod đang nghe, số 9376 hay chuỗi "9376", named port có tồn tại phía
  Pod, và `protocol` có đúng.
- **EndpointSlice là bằng chứng selector khớp label**: nếu
  `kubectl get endpointslices -l kubernetes.io/service-name=<tên>` cho cột `ENDPOINTS` là
  `<none>` thì so `spec.selector` của Service với `metadata.labels` của Pod — lỗi này không
  liên quan gì tới DNS hay kube-proxy.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Từng dòng rule trong `iptables-save` và `ipvsadm -ln` | ở lần đọc này chỉ cần nhận diện có đủ bộ chuỗi `KUBE-SERVICES` → `KUBE-SVC-<hash>` → `KUBE-SEP-<hash>` (hoặc virtual server có đủ real server); đọc hiểu từng flag là kỹ năng Linux, không phải mục tiêu của bài | không cần — tra cứu khi gặp sự cố thật |
| Trường hợp hiếm hairpin mode (`hairpin-veth`, `promiscuous-bridge`, bridge `cbr0`) | mô tả cấu hình bridge đời cũ; chỉ mở khi gặp đúng triệu chứng Pod không tự gọi được chính nó qua Service IP | không cần — tra cứu khi gặp đúng triệu chứng |

---

Một vấn đề khá thường gặp với các bản cài đặt Kubernetes mới là Service không hoạt động đúng.
Bạn đã chạy các Pod thông qua một Deployment (hoặc một workload controller khác) và tạo một
Service, nhưng lại không nhận được phản hồi nào khi cố truy cập nó. Tài liệu này hy vọng sẽ
giúp bạn tìm ra điều gì đang trục trặc.

## Chạy lệnh trong một Pod (Running commands in a Pod)

Với nhiều bước trong bài này, bạn sẽ muốn nhìn thấy những gì một Pod đang chạy trong cluster
nhìn thấy. Cách đơn giản nhất để làm điều đó là chạy một Pod busybox tương tác:

```none
kubectl run -it --rm --restart=Never busybox --image=registry.k8s.io/busybox:1.27.2 sh
```

> **Ghi chú:** Nếu bạn không thấy dấu nhắc lệnh, hãy thử nhấn enter.

Nếu bạn đã có sẵn một Pod đang chạy mà bạn muốn dùng, bạn có thể chạy một lệnh trong Pod đó
bằng:

```shell
kubectl exec <POD-NAME> -c <CONTAINER-NAME> -- <COMMAND>
```

## Chuẩn bị (Setup)

Cho mục đích của bài hướng dẫn này, hãy chạy vài Pod. Vì có lẽ bạn đang gỡ lỗi Service của
chính mình, bạn có thể thay bằng các thông tin của riêng bạn, hoặc làm theo bài để có thêm
một điểm dữ liệu đối chiếu.

```shell
kubectl create deployment hostnames --image=registry.k8s.io/serve_hostname
```
```none
deployment.apps/hostnames created
```

Các lệnh `kubectl` sẽ in ra loại và tên của tài nguyên vừa được tạo hoặc thay đổi, và bạn có
thể dùng chúng trong các lệnh tiếp theo.

Hãy scale Deployment lên 3 replica.
```shell
kubectl scale deployment hostnames --replicas=3
```
```none
deployment.apps/hostnames scaled
```

Lưu ý rằng điều này giống hệt như khi bạn khởi động Deployment bằng YAML sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hostnames
  name: hostnames
spec:
  selector:
    matchLabels:
      app: hostnames
  replicas: 3
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: registry.k8s.io/serve_hostname
```

Label "app" được `kubectl create deployment` tự động đặt theo tên của Deployment.

Bạn có thể xác nhận các Pod của mình đang chạy:

```shell
kubectl get pods -l app=hostnames
```
```none
NAME                        READY     STATUS    RESTARTS   AGE
hostnames-632524106-bbpiw   1/1       Running   0          2m
hostnames-632524106-ly40y   1/1       Running   0          2m
hostnames-632524106-tlaok   1/1       Running   0          2m
```

Bạn cũng có thể xác nhận rằng các Pod đang phục vụ. Bạn có thể lấy danh sách địa chỉ IP của
các Pod và kiểm tra chúng trực tiếp.

```shell
kubectl get pods -l app=hostnames \
    -o go-template='{{range .items}}{{.status.podIP}}{{"\n"}}{{end}}'
```
```none
10.244.0.5
10.244.0.6
10.244.0.7
```

Container ví dụ dùng trong bài này phục vụ hostname của chính nó qua HTTP trên port 9376,
nhưng nếu bạn đang gỡ lỗi ứng dụng của riêng mình, bạn sẽ dùng số port mà các Pod của bạn
đang lắng nghe.

Từ bên trong một Pod:

```shell
for ep in 10.244.0.5:9376 10.244.0.6:9376 10.244.0.7:9376; do
    wget -qO- $ep
done
```

Kết quả sẽ trông giống như:

```
hostnames-632524106-bbpiw
hostnames-632524106-ly40y
hostnames-632524106-tlaok
```

Nếu tới bước này bạn không nhận được các phản hồi như mong đợi, các Pod của bạn có thể không
khỏe mạnh, hoặc không lắng nghe trên port mà bạn nghĩ. Bạn có thể thấy `kubectl logs` hữu ích
để xem chuyện gì đang xảy ra, hoặc có thể bạn cần `kubectl exec` trực tiếp vào Pod và gỡ lỗi
từ đó.

Giả sử tới giờ mọi thứ đều theo đúng kế hoạch, bạn có thể bắt đầu điều tra vì sao Service của
bạn không hoạt động.

## Service có tồn tại không? (Does the Service exist?)

Bạn đọc tinh ý sẽ nhận ra rằng bạn thực ra chưa hề tạo Service — điều đó là cố ý. Đây là bước
đôi khi bị bỏ quên, và là thứ đầu tiên cần kiểm tra.

Điều gì sẽ xảy ra nếu bạn cố truy cập một Service không tồn tại? Nếu bạn có một Pod khác tiêu
thụ Service này theo tên, bạn sẽ nhận được kết quả kiểu như:

```shell
wget -O- hostnames
```
```none
Resolving hostnames (hostnames)... failed: Name or service not known.
wget: unable to resolve host address 'hostnames'
```

Điều đầu tiên cần kiểm tra là Service đó có thực sự tồn tại hay không:

```shell
kubectl get svc hostnames
```
```none
No resources found.
Error from server (NotFound): services "hostnames" not found
```

Hãy tạo Service. Như trước, đây là cho bài hướng dẫn — bạn có thể dùng thông tin Service của
riêng bạn ở đây.

```shell
kubectl expose deployment hostnames --port=80 --target-port=9376
```
```none
service/hostnames exposed
```

Và đọc lại nó:

```shell
kubectl get svc hostnames
```
```none
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
hostnames   ClusterIP   10.0.1.175   <none>        80/TCP    5s
```

Giờ bạn biết rằng Service tồn tại.

Như trước, điều này giống hệt như khi bạn khởi động Service bằng YAML:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hostnames
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
```

Để làm nổi bật đầy đủ dải cấu hình, Service bạn tạo ở đây dùng số port khác với các Pod. Với
nhiều Service thực tế, các giá trị này có thể giống nhau.

## Có quy tắc Network Policy Ingress nào ảnh hưởng tới các Pod đích không? (Any Network Policy Ingress rules affecting the target Pods?)

Nếu bạn đã triển khai bất kỳ quy tắc Network Policy Ingress nào có thể ảnh hưởng tới lưu lượng
đi vào các Pod `hostnames-*`, chúng cần được rà soát lại.

Vui lòng tham khảo [Network Policies](84-network-policies-vi.md) để biết thêm chi tiết.

## Service có hoạt động qua tên DNS không? (Does the Service work by DNS name?)

Một trong những cách phổ biến nhất mà client tiêu thụ một Service là thông qua tên DNS.

Từ một Pod trong cùng namespace:

```shell
nslookup hostnames
```
```none
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      hostnames
Address 1: 10.0.1.175 hostnames.default.svc.cluster.local
```

Nếu lệnh này thất bại, có thể Pod và Service của bạn đang nằm ở hai namespace khác nhau, hãy
thử tên có kèm namespace (một lần nữa, từ trong Pod):

```shell
nslookup hostnames.default
```
```none
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      hostnames.default
Address 1: 10.0.1.175 hostnames.default.svc.cluster.local
```

Nếu cách này chạy được, bạn sẽ cần điều chỉnh ứng dụng để dùng tên liên-namespace
(cross-namespace), hoặc chạy ứng dụng và Service trong cùng một namespace. Nếu vẫn thất bại,
hãy thử tên đầy đủ (fully-qualified name):

```shell
nslookup hostnames.default.svc.cluster.local
```
```none
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      hostnames.default.svc.cluster.local
Address 1: 10.0.1.175 hostnames.default.svc.cluster.local
```

Hãy chú ý hậu tố ở đây: "default.svc.cluster.local". "default" là namespace bạn đang thao
tác. "svc" cho biết đây là một Service. "cluster.local" là domain của cluster bạn, và CÓ THỂ
khác trong cluster của riêng bạn.

Bạn cũng có thể thử điều này từ một node trong cluster:

> **Ghi chú:** 10.0.0.10 là IP của DNS Service trong cluster này; IP của bạn có thể khác.

```shell
nslookup hostnames.default.svc.cluster.local 10.0.0.10
```
```none
Server:         10.0.0.10
Address:        10.0.0.10#53

Name:   hostnames.default.svc.cluster.local
Address: 10.0.1.175
```

Nếu bạn phân giải được tên đầy đủ nhưng không phân giải được tên tương đối, bạn cần kiểm tra
file `/etc/resolv.conf` trong Pod của bạn có đúng không. Từ trong một Pod:

```shell
cat /etc/resolv.conf
```

Bạn sẽ thấy nội dung kiểu như:

```
nameserver 10.0.0.10
search default.svc.cluster.local svc.cluster.local cluster.local example.com
options ndots:5
```

Dòng `nameserver` phải trỏ tới DNS Service của cluster bạn. Giá trị này được truyền vào
`kubelet` qua flag `--cluster-dns`.

Dòng `search` phải chứa một hậu tố phù hợp để bạn tìm được tên Service. Trong trường hợp này,
nó tìm Service trong namespace cục bộ ("default.svc.cluster.local"), Service trong mọi
namespace ("svc.cluster.local"), và cuối cùng là các tên trong cluster ("cluster.local"). Tùy
bản cài đặt của bạn, có thể có thêm các bản ghi sau đó (tổng cộng tối đa 6). Hậu tố cluster
được truyền vào `kubelet` qua flag `--cluster-domain`. Trong suốt tài liệu này, hậu tố cluster
được giả định là "cluster.local". Cluster của riêng bạn có thể được cấu hình khác, khi đó bạn
nên thay đổi tương ứng trong tất cả các lệnh phía trước.

Dòng `options` phải đặt `ndots` đủ cao để thư viện DNS client của bạn xem xét các đường dẫn
tìm kiếm (search path). Kubernetes mặc định đặt giá trị này là 5, đủ cao để bao phủ tất cả
các tên DNS mà nó sinh ra.

### Có Service nào hoạt động qua tên DNS không? (Does any Service work by DNS name?) {#does-any-service-exist-in-dns}

Nếu các bước trên vẫn thất bại, thì việc tra cứu DNS đang không hoạt động cho Service của
bạn. Bạn có thể lùi lại một bước và xem còn thứ gì khác đang không hoạt động. Service master
của Kubernetes lẽ ra phải luôn hoạt động. Từ trong một Pod:

```shell
nslookup kubernetes.default
```
```none
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local
```

Nếu lệnh này thất bại, vui lòng xem mục [kube-proxy](#is-the-kube-proxy-working) của tài liệu
này, hoặc thậm chí quay lại đầu tài liệu và bắt đầu lại, nhưng thay vì gỡ lỗi Service của
riêng bạn, hãy gỡ lỗi DNS Service.

## Service có hoạt động qua IP không? (Does the Service work by IP?)

Giả sử bạn đã xác nhận rằng DNS hoạt động, bước tiếp theo là kiểm tra xem Service của bạn có
hoạt động qua địa chỉ IP hay không. Từ một Pod trong cluster của bạn, truy cập IP của Service
(lấy từ `kubectl get` ở trên).

```shell
for i in $(seq 1 3); do 
    wget -qO- 10.0.1.175:80
done
```

Kết quả sẽ trông giống như:

```
hostnames-632524106-bbpiw
hostnames-632524106-ly40y
hostnames-632524106-tlaok
```

Nếu Service của bạn đang hoạt động, bạn sẽ nhận được các phản hồi đúng. Nếu không, có khá
nhiều thứ có thể đang trục trặc. Hãy đọc tiếp.

## Service có được định nghĩa đúng không? (Is the Service defined correctly?)

Nghe có vẻ ngớ ngẩn, nhưng bạn thực sự nên kiểm tra đi kiểm tra lại nhiều lần rằng Service
của bạn đúng và khớp với port của Pod. Đọc lại Service của bạn và xác minh nó:

```shell
kubectl get service hostnames -o json
```
```json
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "hostnames",
        "namespace": "default",
        "uid": "428c8b6c-24bc-11e5-936d-42010af0a9bc",
        "resourceVersion": "347189",
        "creationTimestamp": "2015-07-07T15:24:29Z",
        "labels": {
            "app": "hostnames"
        }
    },
    "spec": {
        "ports": [
            {
                "name": "default",
                "protocol": "TCP",
                "port": 80,
                "targetPort": 9376,
                "nodePort": 0
            }
        ],
        "selector": {
            "app": "hostnames"
        },
        "clusterIP": "10.0.1.175",
        "type": "ClusterIP",
        "sessionAffinity": "None"
    },
    "status": {
        "loadBalancer": {}
    }
}
```

* Port của Service mà bạn đang cố truy cập có được liệt kê trong `spec.ports[]` không?
* `targetPort` có đúng với các Pod của bạn không (một số Pod dùng port khác với Service)?
* Nếu bạn định dùng port dạng số, nó là số (9376) hay chuỗi "9376"?
* Nếu bạn định dùng port có tên (named port), các Pod của bạn có expose một port với đúng tên
  đó không?
* `protocol` của port có đúng với các Pod của bạn không?

## Service có EndpointSlice nào không? (Does the Service have any EndpointSlices?)

Nếu bạn đã tới được đây, bạn đã xác nhận rằng Service của bạn được định nghĩa đúng và được
DNS phân giải. Bây giờ hãy kiểm tra rằng các Pod bạn chạy có thực sự được Service chọn hay
không.

Trước đó bạn đã thấy rằng các Pod đang chạy. Bạn có thể kiểm tra lại điều đó:

```shell
kubectl get pods -l app=hostnames
```
```none
NAME                        READY     STATUS    RESTARTS   AGE
hostnames-632524106-bbpiw   1/1       Running   0          1h
hostnames-632524106-ly40y   1/1       Running   0          1h
hostnames-632524106-tlaok   1/1       Running   0          1h
```

Tham số `-l app=hostnames` là một label selector được cấu hình trên Service.

Cột "AGE" cho biết các Pod này đã chạy được khoảng một giờ, ngụ ý rằng chúng đang chạy ổn và
không bị crash.

Cột "RESTARTS" cho biết các Pod này không bị crash thường xuyên hay bị khởi động lại. Việc
khởi động lại thường xuyên có thể dẫn tới các sự cố kết nối chập chờn. Nếu số lần khởi động
lại cao, hãy đọc thêm về cách
[gỡ lỗi Pod](299-debug-pods-vi.md).

Bên trong hệ thống Kubernetes có một vòng lặp điều khiển (control loop) đánh giá selector của
mọi Service và lưu kết quả vào một hoặc nhiều object EndpointSlice.

```shell
kubectl get endpointslices -l kubernetes.io/service-name=hostnames

NAME              ADDRESSTYPE   PORTS   ENDPOINTS
hostnames-ytpni   IPv4          9376    10.244.0.5,10.244.0.6,10.244.0.7
```

Điều này xác nhận rằng EndpointSlice controller đã tìm thấy đúng các Pod cho Service của bạn.
Nếu cột `ENDPOINTS` là `<none>`, bạn nên kiểm tra rằng trường `spec.selector` của Service
thực sự chọn đúng các giá trị `metadata.labels` trên các Pod của bạn. Một lỗi phổ biến là gõ
nhầm hoặc sai sót khác, chẳng hạn Service chọn `app=hostnames` nhưng Deployment lại khai
`run=hostnames`, như ở các phiên bản trước 1.18, khi lệnh `kubectl run` cũng có thể được dùng
để tạo một Deployment.

## Các Pod có hoạt động không? (Are the Pods working?)

Tới đây, bạn biết rằng Service của bạn tồn tại và đã chọn các Pod của bạn. Ở đầu bài, bạn đã
xác minh chính các Pod. Hãy kiểm tra lại một lần nữa rằng các Pod thực sự đang hoạt động —
bạn có thể bỏ qua cơ chế Service và đi thẳng tới các Pod, như được liệt kê bởi Endpoints ở
trên.

> **Ghi chú:** Các lệnh này dùng port của Pod (9376) chứ không phải port của Service (80).

Từ trong một Pod:

```shell
for ep in 10.244.0.5:9376 10.244.0.6:9376 10.244.0.7:9376; do
    wget -qO- $ep
done
```

Kết quả sẽ trông giống như:

```
hostnames-632524106-bbpiw
hostnames-632524106-ly40y
hostnames-632524106-tlaok
```

Bạn kỳ vọng mỗi Pod trong danh sách endpoint trả về hostname của chính nó. Nếu điều đó không
xảy ra (hoặc không đúng với hành vi chuẩn của chính các Pod của bạn), bạn nên điều tra xem
chuyện gì đang xảy ra ở đó.

## kube-proxy có hoạt động không? (Is the kube-proxy working?) {#is-the-kube-proxy-working}

Nếu bạn tới được đây, Service của bạn đang chạy, có EndpointSlice, và các Pod của bạn thực sự
đang phục vụ. Tới lúc này, toàn bộ cơ chế proxy của Service trở thành đối tượng khả nghi. Hãy
xác nhận nó, từng phần một.

Cài đặt (implementation) mặc định của Service, và cũng là thứ được dùng trên hầu hết các
cluster, là kube-proxy. Đây là một chương trình chạy trên mọi node và cấu hình một trong số
ít cơ chế để cung cấp lớp trừu tượng Service. Nếu cluster của bạn không dùng kube-proxy, các
mục sau sẽ không áp dụng, và bạn sẽ phải tự điều tra cài đặt Service mà bạn đang dùng.

### kube-proxy có đang chạy không? (Is kube-proxy running?)

Xác nhận rằng `kube-proxy` đang chạy trên các node của bạn. Chạy trực tiếp trên một node, bạn
sẽ nhận được kết quả như bên dưới:

```shell
ps auxw | grep kube-proxy
```
```none
root  4194  0.4  0.1 101864 17696 ?    Sl Jul04  25:43 /usr/local/bin/kube-proxy --master=https://kubernetes-master --kubeconfig=/var/lib/kube-proxy/kubeconfig --v=2
```

Tiếp theo, xác nhận rằng nó không thất bại vì một điều gì đó hiển nhiên, như không liên lạc
được với master. Để làm điều này, bạn sẽ phải xem log. Cách truy cập log phụ thuộc vào hệ
điều hành của node. Trên một số hệ điều hành, đó là một file như /var/log/kube-proxy.log,
trong khi các hệ điều hành khác dùng `journalctl` để truy cập log. Bạn sẽ thấy nội dung kiểu
như:

```none
I1027 22:14:53.995134    5063 server.go:200] Running in resource-only container "/kube-proxy"
I1027 22:14:53.998163    5063 server.go:247] Using iptables Proxier.
I1027 22:14:54.038140    5063 proxier.go:352] Setting endpoints for "kube-system/kube-dns:dns-tcp" to [10.244.1.3:53]
I1027 22:14:54.038164    5063 proxier.go:352] Setting endpoints for "kube-system/kube-dns:dns" to [10.244.1.3:53]
I1027 22:14:54.038209    5063 proxier.go:352] Setting endpoints for "default/kubernetes:https" to [10.240.0.2:443]
I1027 22:14:54.038238    5063 proxier.go:429] Not syncing iptables until Services and Endpoints have been received from master
I1027 22:14:54.040048    5063 proxier.go:294] Adding new service "default/kubernetes:https" at 10.0.0.1:443/TCP
I1027 22:14:54.040154    5063 proxier.go:294] Adding new service "kube-system/kube-dns:dns" at 10.0.0.10:53/UDP
I1027 22:14:54.040223    5063 proxier.go:294] Adding new service "kube-system/kube-dns:dns-tcp" at 10.0.0.10:53/TCP
```

Nếu bạn thấy các thông báo lỗi về việc không thể liên lạc được với master, bạn nên kiểm tra
lại cấu hình node và các bước cài đặt.

Kube-proxy có thể chạy ở một trong vài chế độ. Trong log liệt kê ở trên, dòng
`Using iptables Proxier` cho biết kube-proxy đang chạy ở chế độ "iptables". Chế độ phổ biến
khác là "ipvs".

#### Chế độ iptables (Iptables mode)

Ở chế độ "iptables", bạn sẽ thấy nội dung như sau trên một node:

```shell
iptables-save | grep hostnames
```
```none
-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376
-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376
-A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR
```

Với mỗi port của mỗi Service, sẽ có 1 rule trong `KUBE-SERVICES` và một chuỗi (chain)
`KUBE-SVC-<hash>`. Với mỗi Pod endpoint, sẽ có một số ít rule trong chuỗi `KUBE-SVC-<hash>`
đó và một chuỗi `KUBE-SEP-<hash>` với một số ít rule bên trong. Các rule chính xác sẽ khác
nhau tùy cấu hình cụ thể của bạn (bao gồm cả node-port và load-balancer).

#### Chế độ IPVS (IPVS mode)

Ở chế độ "ipvs", bạn sẽ thấy nội dung như sau trên một node:

```shell
ipvsadm -ln
```
```none
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
...
TCP  10.0.1.175:80 rr
  -> 10.244.0.5:9376               Masq    1      0          0
  -> 10.244.0.6:9376               Masq    1      0          0
  -> 10.244.0.7:9376               Masq    1      0          0
...
```

Với mỗi port của mỗi Service, cộng thêm mọi NodePort, external IP và IP của load-balancer,
kube-proxy sẽ tạo một virtual server. Với mỗi Pod endpoint, nó sẽ tạo các real server tương
ứng. Trong ví dụ này, Service hostnames (`10.0.1.175:80`) có 3 endpoint (`10.244.0.5:9376`,
`10.244.0.6:9376`, `10.244.0.7:9376`).

### kube-proxy có đang proxy không? (Is kube-proxy proxying?)

Giả sử bạn thấy một trong các trường hợp ở trên, hãy thử lại việc truy cập Service của bạn
qua IP từ một trong các node:

```shell
curl 10.0.1.175:80
```
```none
hostnames-632524106-bbpiw
```

Nếu vẫn thất bại, hãy xem log của `kube-proxy` để tìm các dòng cụ thể như:

```none
Setting endpoints for default/hostnames:default to [10.244.0.5:9376 10.244.0.6:9376 10.244.0.7:9376]
```

Nếu bạn không thấy các dòng đó, hãy thử khởi động lại `kube-proxy` với flag `-v` đặt là 4,
rồi xem log lại lần nữa.

### Trường hợp hiếm: Pod không truy cập được chính nó qua Service IP (Edge case: A Pod fails to reach itself via the Service IP) {#a-pod-fails-to-reach-itself-via-the-service-ip}

Điều này nghe có vẻ khó xảy ra, nhưng nó có xảy ra thật, và lẽ ra nó phải hoạt động.

Vấn đề này có thể xảy ra khi mạng không được cấu hình đúng cho lưu lượng "hairpin", thường là
khi `kube-proxy` chạy ở chế độ `iptables` và các Pod được kết nối bằng mạng bridge. `Kubelet`
cung cấp một [flag](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
`hairpin-mode` cho phép các endpoint của một Service cân bằng tải ngược về chính chúng nếu
chúng cố truy cập VIP của chính Service của mình. Flag `hairpin-mode` phải được đặt là
`hairpin-veth` hoặc `promiscuous-bridge`.

Các bước phổ biến để xử lý sự cố này như sau:

* Xác nhận `hairpin-mode` được đặt là `hairpin-veth` hoặc `promiscuous-bridge`. Bạn sẽ thấy
nội dung như bên dưới. Trong ví dụ sau, `hairpin-mode` được đặt là `promiscuous-bridge`.

```shell
ps auxw | grep kubelet
```
```none
root      3392  1.1  0.8 186804 65208 ?        Sl   00:51  11:11 /usr/local/bin/kubelet --enable-debugging-handlers=true --config=/etc/kubernetes/manifests --allow-privileged=True --v=4 --cluster-dns=10.0.0.10 --cluster-domain=cluster.local --configure-cbr0=true --cgroup-root=/ --system-cgroups=/system --hairpin-mode=promiscuous-bridge --runtime-cgroups=/docker-daemon --kubelet-cgroups=/kubelet --babysit-daemons=true --max-pods=110 --serialize-image-pulls=false --outofdisk-transition-frequency=0
```

* Xác nhận `hairpin-mode` hiệu lực (effective). Để làm điều này, bạn sẽ phải xem log của
kubelet. Cách truy cập log phụ thuộc vào hệ điều hành của node. Trên một số hệ điều hành, đó
là một file như /var/log/kubelet.log, trong khi các hệ điều hành khác dùng `journalctl` để
truy cập log. Xin lưu ý rằng hairpin mode hiệu lực có thể không khớp với flag `--hairpin-mode`
vì lý do tương thích. Hãy kiểm tra xem trong kubelet.log có dòng log nào chứa từ khóa
`hairpin` không. Sẽ có các dòng log cho biết hairpin mode hiệu lực, kiểu như bên dưới.

```none
I0629 00:51:43.648698    3252 kubelet.go:380] Hairpin mode set to "promiscuous-bridge"
```

* Nếu hairpin mode hiệu lực là `hairpin-veth`, hãy bảo đảm `Kubelet` có quyền thao tác trong
`/sys` trên node. Nếu mọi thứ hoạt động đúng, bạn sẽ thấy nội dung như:

```shell
for intf in /sys/devices/virtual/net/cbr0/brif/*; do cat $intf/hairpin_mode; done
```
```none
1
1
1
1
```

* Nếu hairpin mode hiệu lực là `promiscuous-bridge`, hãy bảo đảm `Kubelet` có quyền thao tác
linux bridge trên node. Nếu bridge `cbr0` được dùng và cấu hình đúng, bạn sẽ thấy:

```shell
ifconfig cbr0 |grep PROMISC
```
```none
UP BROADCAST RUNNING PROMISC MULTICAST  MTU:1460  Metric:1
```

* Tìm trợ giúp nếu không cách nào ở trên hiệu quả.

## Tìm trợ giúp (Seek help)

Nếu bạn tới được đây, một điều gì đó rất kỳ lạ đang xảy ra. Service của bạn đang chạy, có
EndpointSlice, và các Pod của bạn thực sự đang phục vụ. DNS của bạn hoạt động, và `kube-proxy`
có vẻ không hành xử sai. Vậy mà Service của bạn vẫn không hoạt động. Hãy cho chúng tôi biết
chuyện gì đang xảy ra, để chúng tôi có thể giúp điều tra!

Liên hệ với chúng tôi trên [Slack](https://slack.k8s.io/) hoặc
[Forum](https://discuss.kubernetes.io) hoặc
[GitHub](https://github.com/kubernetes/kubernetes).

## Tiếp theo (What's next)

Xem [tài liệu tổng quan về xử lý sự cố](296-debug-vi.md) để biết thêm
thông tin.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở checkpoint CP9:

1. Trên cluster lab của bạn, từ một Pod busybox: `wget -qO- <Service-IP>:80` thất bại, nhưng
   gọi thẳng từng `<Pod-IP>:9376` đều trả về hostname, và `nslookup hostnames` phân giải
   đúng. Theo trình tự loại trừ của bài, còn lại những tầng nào để nghi ngờ, và bạn sẽ chạy
   gì trên `k8s-worker1` / `k8s-worker2` ở tầng cuối cùng?
2. `kubectl get endpointslices -l kubernetes.io/service-name=hostnames` cho cột `ENDPOINTS`
   là `<none>` trong khi cả ba Pod đều `Running 1/1`. Có phải kube-proxy hoặc DNS đang hỏng
   không? Lỗi nằm ở đâu?
3. Từ `k8s-master`, bạn gõ `nslookup hostnames` và thất bại, nhưng cùng lệnh đó trong Pod
   busybox lại chạy được. Vì sao bài vẫn coi đây là chuyện bình thường, và muốn thử DNS từ
   node thì phải gõ lệnh như thế nào?
4. Trong Pod, `nslookup hostnames` thất bại nhưng `nslookup hostnames.default` thành công.
   Kết luận được gì, và khác gì với trường hợp chỉ tên đầy đủ
   `hostnames.default.svc.cluster.local` mới phân giải được?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **DNS, Pod và Service IP trực tiếp đã được khẳng định, nên còn ba tầng: định nghĩa
   Service, EndpointSlice, và cuối cùng là kube-proxy.** Trước hết đọc lại Service bằng
   `kubectl get service hostnames -o json` theo checklist 5 câu (port trong `spec.ports[]`,
   `targetPort`, số vs chuỗi, named port, `protocol`), rồi kiểm tra EndpointSlice có đủ
   endpoint. Nếu cả hai đều đúng thì mới sang kube-proxy: trên từng node chạy
   `ps auxw | grep kube-proxy` (hoặc xem log qua `journalctl`), rồi
   `iptables-save | grep hostnames` (chế độ iptables) hoặc `ipvsadm -ln` (chế độ ipvs), và
   thử `curl <Service-IP>:80` từ chính node đó.
2. **Không — kube-proxy và DNS chưa liên quan gì ở đây.** `ENDPOINTS` là `<none>` nghĩa là
   vòng lặp điều khiển đánh giá selector của Service **không chọn được Pod nào**: phải so
   `spec.selector` của Service với `metadata.labels` trên Pod. Lỗi phổ biến bài nêu là gõ
   nhầm kiểu Service chọn `app=hostnames` còn workload lại khai `run=hostnames`. kube-proxy
   chỉ tiêu thụ kết quả của EndpointSlice; khi slice rỗng thì nó không có gì để proxy — nghi
   ngờ nó lúc này là nhảy cóc trình tự.
3. **Vì tên tương đối của Service chỉ phân giải được nhờ `/etc/resolv.conf` mà cluster cấu
   hình bên trong Pod** (dòng `nameserver` trỏ DNS Service, dòng `search` chứa các hậu tố như
   `default.svc.cluster.local`, và `options ndots:5`). Node không có cấu hình đó, nên bài
   yêu cầu mọi phép thử đứng từ trong Pod. Muốn thử từ node thì phải chỉ định cả tên đầy đủ
   lẫn IP của DNS Service: `nslookup hostnames.default.svc.cluster.local <DNS-Service-IP>`.
4. **Pod và Service đang ở hai namespace khác nhau.** Sửa bằng một trong hai cách: cho ứng
   dụng dùng tên liên-namespace (`hostnames.default`), hoặc chạy ứng dụng và Service trong
   cùng namespace. Khác với trường hợp **chỉ tên đầy đủ mới chạy**: khi đó vấn đề không nằm
   ở namespace nữa mà ở chính file `/etc/resolv.conf` của Pod — phải kiểm tra dòng
   `nameserver`, danh sách `search` và giá trị `ndots`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

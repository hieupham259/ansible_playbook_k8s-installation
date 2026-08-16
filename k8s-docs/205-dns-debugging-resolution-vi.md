# Gỡ lỗi phân giải DNS (Debugging DNS Resolution)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/>
>
> Trang này cung cấp các gợi ý để chẩn đoán sự cố DNS.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối — CP6 DNS, CNI và kube-proxy](LO-TRINH-ADMIN.md#cp6--dns-cni-và-kube-proxy),
bài 2/7 · thực hành trực tiếp trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

Đây là bài runbook: giá trị của nó nằm ở **thứ tự các bước loại trừ**, không phải ở từng lệnh
riêng lẻ. Đọc xong bạn phải tự vẽ lại được chuỗi chẩn đoán từ trong Pod ra tới RBAC của
CoreDNS.

**Phải hiểu ở lần đọc này:**

- Quy trình loại trừ theo tầng: tạo Pod `dnsutils` → `nslookup kubernetes.default` → xem
  `/etc/resolv.conf` trong Pod (search path, `nameserver`, `options ndots:5`) → Pod CoreDNS
  có chạy không → log CoreDNS → Service `kube-dns` có tồn tại không → EndpointSlice có
  endpoint không → bật plugin `log` để xác nhận truy vấn có tới CoreDNS.
- Hai điểm dễ nhầm khi kiểm tra: label để lọc Pod DNS là `k8s-app=kube-dns` và tên Service là
  `kube-dns` **cho cả CoreDNS lẫn kube-dns**.
- CoreDNS cần quyền `list`/`watch` trên các tài nguyên service và endpointslice (ClusterRole
  `system:coredns`); thiếu quyền thì truy vấn trả về `SERVFAIL`.
- Truy vấn không kèm namespace chỉ tra trong namespace của Pod đang truy vấn; khác namespace
  thì phải dùng `<service-name>.<namespace>`.
- Ba vấn đề đã biết: stub file của `systemd-resolved` gây vòng lặp chuyển tiếp (kubeadm tự
  phát hiện và chỉnh `--resolv-conf` cho kubelet), giới hạn 3 dòng `nameserver` của glibc,
  và lỗi DNS trên image nền Alpine 3.17 trở xuống.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hai lần trang gốc trỏ sang tài liệu *Debugging Services* (khi Service `kube-dns` hoặc endpoint không xuất hiện) | quy trình lần từ Service về Pod là một bài riêng, không thuộc phạm vi bài này | [CP9 — Xử lý sự cố](LO-TRINH-ADMIN.md#cp9--xử-lý-sự-cố), bài Debug Services |

---

Trang này cung cấp các gợi ý để chẩn đoán sự cố DNS.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Cluster của bạn phải được cấu hình để dùng addon CoreDNS hoặc phiên bản tiền nhiệm của nó là
kube-dns.

Kubernetes server của bạn phải ở phiên bản v1.6 hoặc mới hơn. Để kiểm tra phiên bản,
hãy chạy `kubectl version`.

### Tạo một Pod đơn giản làm môi trường kiểm thử (Create a simple Pod to use as a test environment)

[`admin/dns/dnsutils.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/admin/dns/dnsutils.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
  namespace: default
spec:
  containers:
  - name: dnsutils
    image: registry.k8s.io/e2e-test-images/agnhost:2.39
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

> **Ghi chú:** Ví dụ này tạo một pod trong namespace `default`. Việc phân giải tên DNS cho
> các service phụ thuộc vào namespace của pod. Để biết thêm thông tin, xem
> [DNS cho Service và Pod](10-dns-pod-service-vi.md#các-bản-ghi-dns-dns-records).

Dùng manifest đó để tạo một Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
```
```
pod/dnsutils created
```
…và xác nhận trạng thái của nó:
```shell
kubectl get pods dnsutils
```
```
NAME       READY     STATUS    RESTARTS   AGE
dnsutils   1/1       Running   0          <some-time>
```

Khi Pod đó đang chạy, bạn có thể exec `nslookup` trong môi trường đó.
Nếu bạn thấy kết quả tương tự như sau, DNS đang hoạt động đúng.

```shell
kubectl exec -i -t dnsutils -- nslookup kubernetes.default
```
```
Server:    10.0.0.10
Address 1: 10.0.0.10

Name:      kubernetes.default
Address 1: 10.0.0.1
```

Nếu lệnh `nslookup` thất bại, hãy kiểm tra những điều sau:

### Kiểm tra cấu hình DNS cục bộ trước tiên (Check the local DNS configuration first)

Hãy xem bên trong file resolv.conf.
(Xem [Tùy chỉnh DNS Service](204-dns-custom-nameservers-vi.md) và
[Các vấn đề đã biết](#các-vấn-đề-đã-biết-known-issues) bên dưới để biết thêm thông tin)

```shell
kubectl exec -ti dnsutils -- cat /etc/resolv.conf
```

Xác nhận rằng search path và name server được thiết lập tương tự như sau
(lưu ý rằng search path có thể khác nhau tùy nhà cung cấp cloud):

```
search default.svc.cluster.local svc.cluster.local cluster.local google.internal c.gce_project_id.internal
nameserver 10.0.0.10
options ndots:5
```

Những lỗi như sau cho thấy có vấn đề với add-on CoreDNS (hoặc kube-dns) hoặc với các Service
liên quan:

```shell
kubectl exec -i -t dnsutils -- nslookup kubernetes.default
```
```
Server:    10.0.0.10
Address 1: 10.0.0.10

nslookup: can't resolve 'kubernetes.default'
```

hoặc

```shell
kubectl exec -i -t dnsutils -- nslookup kubernetes.default
```
```
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

nslookup: can't resolve 'kubernetes.default'
```

### Kiểm tra xem DNS pod có đang chạy không (Check if the DNS pod is running)

Dùng lệnh `kubectl get pods` để xác nhận DNS pod đang chạy.

```shell
kubectl get pods --namespace=kube-system -l k8s-app=kube-dns
```
```
NAME                       READY     STATUS    RESTARTS   AGE
...
coredns-7b96bf9f76-5hsxb   1/1       Running   0           1h
coredns-7b96bf9f76-mvmmt   1/1       Running   0           1h
...
```

> **Ghi chú:** Giá trị của label `k8s-app` là `kube-dns` cho cả deployment CoreDNS lẫn
> kube-dns.

Nếu bạn thấy không có Pod CoreDNS nào đang chạy, hoặc Pod ở trạng thái failed/completed,
add-on DNS có thể không được triển khai mặc định trong môi trường hiện tại của bạn và bạn
sẽ phải triển khai nó thủ công.

### Kiểm tra lỗi trong DNS pod (Check for errors in the DNS pod)

Dùng lệnh `kubectl logs` để xem log của các container DNS.

Với CoreDNS:
```shell
kubectl logs --namespace=kube-system -l k8s-app=kube-dns
```

Đây là ví dụ về log của một CoreDNS khỏe mạnh:

```
.:53
2018/08/15 14:37:17 [INFO] CoreDNS-1.2.2
2018/08/15 14:37:17 [INFO] linux/amd64, go1.10.3, 2e322f6
CoreDNS-1.2.2
linux/amd64, go1.10.3, 2e322f6
2018/08/15 14:37:17 [INFO] plugin/reload: Running configuration MD5 = 24e6c59e83ce706f07bcc82c31b1ea1c
```

Hãy xem trong log có thông điệp nào đáng ngờ hoặc bất thường không.

### DNS service có hoạt động không? (Is DNS service up?)

Xác nhận rằng DNS service đang hoạt động bằng lệnh `kubectl get service`.

```shell
kubectl get svc --namespace=kube-system
```
```
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
...
kube-dns     ClusterIP   10.0.0.10      <none>        53/UDP,53/TCP        1h
...
```

> **Ghi chú:** Tên service là `kube-dns` cho cả deployment CoreDNS lẫn kube-dns.

Nếu bạn đã tạo Service, hoặc trong trường hợp Service lẽ ra phải được tạo mặc định nhưng lại
không xuất hiện, xem
[gỡ lỗi Service](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
để biết thêm thông tin.

### Các endpoint DNS có được expose không? (Are DNS endpoints exposed?)

Bạn có thể xác nhận các endpoint DNS đã được expose bằng lệnh `kubectl get endpointslice`.

```shell
kubectl get endpointslice -l kubernetes.io/service-name=kube-dns --namespace=kube-system
```
```
NAME             ADDRESSTYPE   PORTS   ENDPOINTS                  AGE
kube-dns-zxoja   IPv4          53      10.180.3.17,10.180.3.17    1h
```

Nếu bạn không thấy các endpoint, xem phần về endpoint trong tài liệu
[gỡ lỗi Service](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/).

### Các truy vấn DNS có đang được nhận/xử lý không? (Are DNS queries being received/processed?)

Bạn có thể xác nhận CoreDNS có nhận được truy vấn hay không bằng cách thêm plugin `log` vào
cấu hình CoreDNS (tức Corefile). Corefile của CoreDNS được lưu trong một ConfigMap tên
`coredns`. Để sửa nó, dùng lệnh:

```
kubectl -n kube-system edit configmap coredns
```

Sau đó thêm `log` vào phần Corefile theo ví dụ dưới đây:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        log
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          upstream
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

Sau khi lưu các thay đổi, có thể mất một đến hai phút để Kubernetes lan truyền những thay đổi
này tới các Pod CoreDNS.

Tiếp theo, hãy thực hiện vài truy vấn và xem log theo các mục ở trên trong tài liệu này.
Nếu các Pod CoreDNS nhận được truy vấn, bạn sẽ thấy chúng trong log.

Đây là ví dụ về một truy vấn xuất hiện trong log:

```
.:53
2018/08/15 14:37:15 [INFO] CoreDNS-1.2.0
2018/08/15 14:37:15 [INFO] linux/amd64, go1.10.3, 2e322f6
CoreDNS-1.2.0
linux/amd64, go1.10.3, 2e322f6
2018/09/07 15:29:04 [INFO] plugin/reload: Running configuration MD5 = 162475cdf272d8aa601e6fe67a6ad42f
2018/09/07 15:29:04 [INFO] Reloading complete
172.17.0.18:41675 - [07/Sep/2018:15:29:11 +0000] 59925 "A IN kubernetes.default.svc.cluster.local. udp 54 false 512" NOERROR qr,aa,rd,ra 106 0.000066649s
```

### CoreDNS có đủ quyền không? (Does CoreDNS have sufficient permissions?)

CoreDNS phải có khả năng liệt kê (list) các tài nguyên liên quan tới service và endpointslice
để phân giải đúng tên service.

Thông điệp lỗi mẫu:
```
2022-03-18T07:12:15.699431183Z [INFO] 10.96.144.227:52299 - 3686 "A IN serverproxy.contoso.net.cluster.local. udp 52 false 512" SERVFAIL qr,aa,rd 145 0.000091221s
```

Trước tiên, lấy ClusterRole hiện tại của `system:coredns`:

```shell
kubectl describe clusterrole system:coredns -n kube-system
```

Output mong đợi:
```
PolicyRule:
  Resources                        Non-Resource URLs  Resource Names  Verbs
  ---------                        -----------------  --------------  -----
  endpoints                        []                 []              [list watch]
  namespaces                       []                 []              [list watch]
  pods                             []                 []              [list watch]
  services                         []                 []              [list watch]
  endpointslices.discovery.k8s.io  []                 []              [list watch]
```

Nếu thiếu quyền nào, hãy sửa ClusterRole để bổ sung:

```shell
kubectl edit clusterrole system:coredns -n kube-system
```

Ví dụ chèn thêm quyền cho EndpointSlices:
```
...
- apiGroups:
  - discovery.k8s.io
  resources:
  - endpointslices
  verbs:
  - list
  - watch
...
```

### Bạn có đang ở đúng namespace của service không? (Are you in the right namespace for the service?)

Các truy vấn DNS không chỉ định namespace bị giới hạn trong namespace của pod.

Nếu namespace của pod và của service khác nhau, truy vấn DNS phải bao gồm namespace của
service.

Truy vấn này bị giới hạn trong namespace của pod:
```shell
kubectl exec -i -t dnsutils -- nslookup <service-name>
```

Truy vấn này chỉ định namespace:
```shell
kubectl exec -i -t dnsutils -- nslookup <service-name>.<namespace>
```

Để tìm hiểu thêm về phân giải tên, xem
[DNS cho Service và Pod](10-dns-pod-service-vi.md#các-bản-ghi-dns-dns-records).

## Các vấn đề đã biết (Known issues) {#các-vấn-đề-đã-biết-known-issues}

Một số bản phân phối Linux (ví dụ Ubuntu) mặc định dùng một DNS resolver cục bộ
(systemd-resolved). Systemd-resolved di chuyển và thay thế `/etc/resolv.conf` bằng một stub
file, điều này có thể gây ra vòng lặp chuyển tiếp (forwarding loop) nghiêm trọng khi phân
giải tên trên các server upstream. Vấn đề này có thể được khắc phục thủ công bằng cách dùng
flag `--resolv-conf` của kubelet để trỏ tới file `resolv.conf` đúng (với `systemd-resolved`,
đó là `/run/systemd/resolve/resolv.conf`). kubeadm tự động phát hiện `systemd-resolved` và
điều chỉnh các flag của kubelet tương ứng.

Các bản cài đặt Kubernetes không cấu hình file `resolv.conf` của node để dùng DNS của cluster
theo mặc định, vì quá trình đó vốn dĩ phụ thuộc vào từng bản phân phối. Điều này có lẽ nên
được triển khai trong tương lai.

libc của Linux (còn gọi là glibc) mặc định giới hạn số bản ghi `nameserver` DNS ở mức 3,
trong khi Kubernetes cần chiếm 1 bản ghi `nameserver`. Điều này có nghĩa là nếu một bản cài
đặt cục bộ đã dùng đủ 3 `nameserver`, một số mục trong đó sẽ bị mất. Để khắc phục giới hạn
này, node có thể chạy `dnsmasq`, công cụ sẽ cung cấp thêm các mục `nameserver`. Bạn cũng có
thể dùng flag `--resolv-conf` của kubelet.

Nếu bạn dùng Alpine phiên bản 3.17 trở xuống làm base image, DNS có thể không hoạt động đúng
do một vấn đề thiết kế của Alpine. Cho tới musl phiên bản 1.24, cơ chế fallback sang TCP cho
DNS stub resolver vẫn chưa được đưa vào, nghĩa là mọi truy vấn DNS có kích thước trên 512
byte sẽ thất bại. Hãy nâng cấp image của bạn lên Alpine phiên bản 3.18 trở lên.

## Tiếp theo (What's next)

- Xem [Tự động co giãn DNS Service trong cluster](https://kubernetes.io/docs/tasks/administer-cluster/dns-horizontal-autoscaling/).
- Đọc [DNS cho Service và Pod](10-dns-pod-service-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu dưới đây mà không nhìn lại bài là đủ cho lần đọc ở checkpoint này.

1. `nslookup kubernetes.default` từ Pod `dnsutils` thất bại. Kể lại chuỗi các bước loại trừ
   mà bài đưa ra, theo đúng thứ tự, cho tới bước bật plugin `log`.
2. Cluster của bạn chạy CoreDNS. Vì sao lệnh lọc Pod DNS lại dùng `-l k8s-app=kube-dns` chứ
   không phải một label mang tên coredns, và tên Service bạn cần kiểm tra là gì?
3. Log CoreDNS cho thấy truy vấn tới nơi nhưng trả về `SERVFAIL` khi phân giải tên service.
   Bài chỉ ra nguyên nhân nào, và bạn kiểm tra rồi sửa bằng những lệnh gì?
4. Trong Pod `dnsutils` ở namespace `default` trên cluster lab của bạn, `nslookup web` thất
   bại nhưng `nslookup web.demo` thành công. Đây có phải sự cố DNS không? Vì sao?
5. Trên node Ubuntu, vì sao trỏ CoreDNS forward theo `/etc/resolv.conf` của node có thể gây
   vòng lặp chuyển tiếp, file đúng cần trỏ tới là gì, và vì sao cluster dựng bằng kubeadm
   thường không gặp lỗi này?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Chuỗi loại trừ: (1) xem `/etc/resolv.conf` **trong Pod** — kiểm tra `search` path,
   `nameserver` và `options ndots:5`; (2) kiểm tra Pod DNS có chạy không
   (`kubectl get pods -n kube-system -l k8s-app=kube-dns`); (3) đọc log CoreDNS
   (`kubectl logs -n kube-system -l k8s-app=kube-dns`) tìm thông điệp bất thường;
   (4) kiểm tra Service `kube-dns` tồn tại (`kubectl get svc -n kube-system`); (5) kiểm tra
   EndpointSlice có endpoint
   (`kubectl get endpointslice -l kubernetes.io/service-name=kube-dns -n kube-system`);
   (6) thêm plugin `log` vào Corefile trong ConfigMap `coredns` để xác nhận truy vấn có tới
   CoreDNS hay không.
2. Vì **cả deployment CoreDNS lẫn kube-dns đều dùng chung giá trị label `k8s-app: kube-dns`
   và chung tên Service `kube-dns`** — tên cũ được giữ để tương thích với workload phụ thuộc
   vào nó (bài [204](204-dns-custom-nameservers-vi.md) giải thích chủ đích này). Tìm theo
   label hay Service tên "coredns" sẽ không ra kết quả dù CoreDNS đang chạy — đây là chỗ dễ
   trượt khi debug.
3. Nguyên nhân: **CoreDNS thiếu quyền RBAC** — nó phải `list`/`watch` được các tài nguyên
   service và endpointslice. Kiểm tra bằng `kubectl describe clusterrole system:coredns` và
   đối chiếu bảng PolicyRule (endpoints, namespaces, pods, services,
   endpointslices.discovery.k8s.io — đều cần `[list watch]`); thiếu thì
   `kubectl edit clusterrole system:coredns` để bổ sung, ví dụ thêm block `discovery.k8s.io /
   endpointslices / list, watch`.
4. **Không phải sự cố DNS — hành vi đúng thiết kế.** Truy vấn không chỉ định namespace bị
   giới hạn trong namespace của pod (`default`), trong khi service `web` nằm ở namespace
   `demo`; khi namespace của pod và service khác nhau, truy vấn phải kèm namespace:
   `web.demo`.
5. Trên Ubuntu, `systemd-resolved` thay `/etc/resolv.conf` bằng một **stub file** trỏ về
   resolver cục bộ; nếu CoreDNS forward theo file đó, truy vấn upstream có thể quay ngược lại
   chính nó, tạo **vòng lặp chuyển tiếp nghiêm trọng** (plugin `loop` phát hiện và dừng
   CoreDNS). File đúng là **`/run/systemd/resolve/resolv.conf`**, trỏ qua flag
   `--resolv-conf` của kubelet. Cluster kubeadm thường không gặp lỗi vì **kubeadm tự phát
   hiện `systemd-resolved` và điều chỉnh flag của kubelet tương ứng**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

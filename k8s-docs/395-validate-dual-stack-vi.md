# Kiểm chứng dual-stack IPv4/IPv6 (Validate IPv4/IPv6 dual-stack)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/network/validate-dual-stack/>
>
> Tài liệu này hướng dẫn cách kiểm chứng các cluster Kubernetes đã bật dual-stack IPv4/IPv6.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy),
bài 14/14 — **bài cuối giai đoạn** · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy).

**Nói thẳng về giới hạn của cluster lab:** ba VM ở
[A1.2 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) chỉ có địa chỉ IPv4, `kubeadm init`
chỉ nhận một `--pod-network-cidr` IPv4, và Service CIDR là dải IPv4 mặc định của kubeadm. Cluster
lab là **single-stack IPv4**, nên **không bước nào của bài này ra kết quả dual-stack**. Cách dùng
bài đúng đắn ở đây là chạy chính những lệnh kiểm chứng đó để **chứng minh cluster của bạn là
single-stack** — mỗi lệnh trả về một dòng thay vì hai — rồi ghi nhớ dấu hiệu để nhận ra cluster
dual-stack khi gặp. Bài nối tiếp bài [85](85-dual-stack-vi.md) đã đọc ở mạch chính.

**Phải hiểu ở lần đọc này:**

- Ba điều kiện tiên quyết ở mục *Trước khi bạn bắt đầu*: hạ tầng cấp được cho node interface
  IPv4/IPv6 **định tuyến được**, network plugin **hỗ trợ dual-stack**, và cluster **đã bật
  dual-stack**. Thiếu một trong ba thì không có gì để kiểm chứng.
- Kiểm chứng ở mức node: `.spec.podCIDRs` phải có **đúng một khối IPv4 và một khối IPv6**, và
  `.status.addresses` phải có **hai dòng `InternalIP`**.
- Kiểm chứng ở mức Pod: `.status.podIPs` chứa cả hai địa chỉ, và bài đưa ba đường nhìn thấy chúng
  — `kubectl get pods` với `-o go-template`, Downward API dùng `fieldPath: status.podIPs` (biến
  `MY_POD_IPS`, giá trị là danh sách phân tách bằng dấu phẩy), và file `/etc/hosts` trong container
  có **hai dòng cho cùng một tên Pod**.
- Ba hành vi của Service theo `.spec.ipFamilyPolicy`, ở mục *Kiểm chứng các Service*: không khai
  báo gì → `SingleStack`, ClusterIP lấy từ dải `service-cluster-ip-range` **được cấu hình đầu
  tiên**; khai báo `ipFamilies` với `IPv6` đứng đầu → vẫn `SingleStack` nhưng lấy từ dải IPv6; đặt
  `PreferDualStack` → cấp **cả hai**, và `.spec.clusterIP` là địa chỉ theo họ của **phần tử đầu
  tiên** trong `.spec.ipFamilies`.
- Bẫy đọc kết quả, nêu ngay trong ô *Ghi chú*: `kubectl get svc` **chỉ hiển thị IP chính** ở cột
  `CLUSTER-IP`. Muốn thấy đủ cả hai phải dùng `kubectl describe` và đọc dòng `IPs:`, hoặc đọc
  trường `.spec.clusterIPs`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mọi giá trị IPv6 trong output mẫu | cluster lab là single-stack IPv4 theo [A1.2](labs/LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) và [A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) — không có gì để đối chiếu | chỉ đo được trên một cluster dual-stack thật; ở đây dùng chính các lệnh đó để chứng minh cluster của bạn là single-stack |
| Cách **bật** dual-stack cho một cluster | bài này chỉ *kiểm chứng*, không bật | khái niệm ở bài [85](85-dual-stack-vi.md) đã đọc; việc thêm hoặc bớt họ địa chỉ cho cluster đang chạy ở bài [394](394-reconfigure-default-service-ip-ranges-vi.md) vừa đọc |
| Mục *Tạo một Service dual-stack có cân bằng tải* | cần nhà cung cấp cloud cấp external load balancer đã bật IPv6; cluster lab không có cloud provider | ngoài phạm vi lộ trình — đường đưa dịch vụ ra ngoài của cluster lab là Ingress, đã làm ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) |
| Mốc phiên bản GA của dual-stack ở mục *Trước khi bạn bắt đầu* | baseline của lộ trình đã vượt mốc này từ lâu | [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) là nơi duy nhất giữ số phiên bản |

---

Tài liệu này hướng dẫn cách kiểm chứng các cluster Kubernetes đã bật dual-stack IPv4/IPv6.

## Trước khi bạn bắt đầu (Before you begin)

* Nhà cung cấp phải hỗ trợ mạng dual-stack (nhà cung cấp cloud hoặc hạ tầng nào khác đều phải
  có khả năng cấp cho các node Kubernetes những network interface IPv4/IPv6 định tuyến được).
* Một [network plugin](183-network-plugins-vi.md) có hỗ trợ mạng dual-stack.
* Cluster đã [bật dual-stack](85-dual-stack-vi.md).

Máy chủ Kubernetes của bạn phải ở phiên bản bằng hoặc mới hơn v1.23. Để kiểm tra phiên bản,
hãy chạy `kubectl version`.

> **Ghi chú:** Bạn vẫn có thể kiểm chứng trên phiên bản cũ hơn, nhưng tính năng này chỉ đạt
> trạng thái GA và được hỗ trợ chính thức kể từ v1.23.

## Kiểm chứng việc cấp phát địa chỉ (Validate addressing)

### Kiểm chứng địa chỉ của node (Validate node addressing)

Mỗi Node dual-stack phải được cấp phát một khối IPv4 duy nhất và một khối IPv6 duy nhất.
Hãy kiểm chứng rằng các dải địa chỉ Pod IPv4/IPv6 đã được cấu hình bằng cách chạy lệnh dưới đây.
Thay tên node mẫu bằng một Node dual-stack hợp lệ trong cluster của bạn. Trong ví dụ này,
tên của Node là `k8s-linuxpool1-34450317-0`:

```shell
kubectl get nodes k8s-linuxpool1-34450317-0 -o go-template --template='{{range .spec.podCIDRs}}{{printf "%s\n" .}}{{end}}'
```

```
10.244.1.0/24
2001:db8::/64
```

Phải có đúng một khối IPv4 và một khối IPv6 được cấp phát.

Hãy kiểm chứng rằng node đã nhận diện được cả interface IPv4 lẫn IPv6.
Thay tên node bằng một node hợp lệ trong cluster.
Trong ví dụ này, tên node là `k8s-linuxpool1-34450317-0`:

```shell
kubectl get nodes k8s-linuxpool1-34450317-0 -o go-template --template='{{range .status.addresses}}{{printf "%s: %s\n" .type .address}}{{end}}'
```

```
Hostname: k8s-linuxpool1-34450317-0
InternalIP: 10.0.0.5
InternalIP: 2001:db8:10::5
```

### Kiểm chứng địa chỉ của Pod (Validate Pod addressing)

Hãy kiểm chứng rằng một Pod đã được gán cả địa chỉ IPv4 lẫn IPv6. Thay tên Pod bằng một Pod
hợp lệ trong cluster của bạn. Trong ví dụ này, tên Pod là `pod01`:

```shell
kubectl get pods pod01 -o go-template --template='{{range .status.podIPs}}{{printf "%s\n" .ip}}{{end}}'
```

```
10.244.1.4
2001:db8::4
```

Bạn cũng có thể kiểm chứng các IP của Pod thông qua Downward API bằng fieldPath `status.podIPs`.
Đoạn cấu hình dưới đây minh họa cách bạn phơi bày các IP của Pod ra một biến môi trường tên là
`MY_POD_IPS` bên trong container.

```yaml
        env:
        - name: MY_POD_IPS
          valueFrom:
            fieldRef:
              fieldPath: status.podIPs
```

Lệnh dưới đây in ra giá trị của biến môi trường `MY_POD_IPS` từ bên trong container. Giá trị này
là một danh sách phân tách bằng dấu phẩy, tương ứng với các địa chỉ IPv4 và IPv6 của Pod.

```shell
kubectl exec -it pod01 -- set | grep MY_POD_IPS
```

```
MY_POD_IPS=10.244.1.4,2001:db8::4
```

Các địa chỉ IP của Pod cũng sẽ được ghi vào `/etc/hosts` bên trong container.
Lệnh dưới đây chạy `cat` trên file `/etc/hosts` của một Pod dual-stack.
Từ kết quả đầu ra, bạn có thể xác nhận cả địa chỉ IPv4 lẫn địa chỉ IPv6 của Pod.

```shell
kubectl exec -it pod01 -- cat /etc/hosts
```

```
# Kubernetes-managed hosts file.
127.0.0.1    localhost
::1    localhost ip6-localhost ip6-loopback
fe00::0    ip6-localnet
fe00::0    ip6-mcastprefix
fe00::1    ip6-allnodes
fe00::2    ip6-allrouters
10.244.1.4    pod01
2001:db8::4    pod01
```

## Kiểm chứng các Service (Validate Services)

Hãy tạo Service dưới đây, trong đó không khai báo tường minh `.spec.ipFamilyPolicy`.
Kubernetes sẽ gán cho Service một cluster IP lấy từ dải `service-cluster-ip-range` được cấu hình
đầu tiên, và đặt `.spec.ipFamilyPolicy` thành `SingleStack`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  labels:
    app.kubernetes.io/name: MyApp
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
```

Dùng `kubectl` để xem YAML của Service.

```shell
kubectl get svc my-service -o yaml
```

Service có `.spec.ipFamilyPolicy` được đặt là `SingleStack` và `.spec.clusterIP` được đặt là một
địa chỉ IPv4 lấy từ dải đầu tiên được cấu hình qua flag `--service-cluster-ip-range` trên
kube-controller-manager.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
spec:
  clusterIP: 10.0.217.164
  clusterIPs:
  - 10.0.217.164
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 9376
  selector:
    app.kubernetes.io/name: MyApp
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

Hãy tạo Service dưới đây, trong đó khai báo tường minh `IPv6` là phần tử đầu tiên của mảng
`.spec.ipFamilies`. Kubernetes sẽ gán cho Service một cluster IP lấy từ dải IPv6 đã được cấu hình
trong `service-cluster-ip-range`, và đặt `.spec.ipFamilyPolicy` thành `SingleStack`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  labels:
    app.kubernetes.io/name: MyApp
spec:
  ipFamilies:
  - IPv6
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
```

Dùng `kubectl` để xem YAML của Service.

```shell
kubectl get svc my-service -o yaml
```

Service có `.spec.ipFamilyPolicy` được đặt là `SingleStack` và `.spec.clusterIP` được đặt là một
địa chỉ IPv6 lấy từ dải IPv6 được cấu hình qua flag `--service-cluster-ip-range` trên
kube-controller-manager.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: MyApp
  name: my-service
spec:
  clusterIP: 2001:db8:fd00::5118
  clusterIPs:
  - 2001:db8:fd00::5118
  ipFamilies:
  - IPv6
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app.kubernetes.io/name: MyApp
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

Hãy tạo Service dưới đây, trong đó khai báo tường minh `PreferDualStack` cho
`.spec.ipFamilyPolicy`. Kubernetes sẽ gán cả địa chỉ IPv4 lẫn IPv6 (vì cluster này đã bật
dual-stack) và chọn `.spec.ClusterIP` từ danh sách `.spec.ClusterIPs` dựa trên họ địa chỉ
(address family) của phần tử đầu tiên trong mảng `.spec.ipFamilies`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  labels:
    app.kubernetes.io/name: MyApp
spec:
  ipFamilyPolicy: PreferDualStack
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
```

> **Ghi chú:** Lệnh `kubectl get svc` chỉ hiển thị IP chính (primary IP) ở cột `CLUSTER-IP`.
>
> ```shell
> kubectl get svc -l app.kubernetes.io/name=MyApp
> ```
>
> ```
> NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
> my-service   ClusterIP   10.0.216.242   <none>        80/TCP    5s
> ```

Hãy dùng `kubectl describe` để kiểm chứng rằng Service nhận được cluster IP từ cả khối địa chỉ
IPv4 lẫn khối địa chỉ IPv6. Sau đó bạn có thể kiểm chứng việc truy cập tới service qua các IP và
port đó.

```shell
kubectl describe svc -l app.kubernetes.io/name=MyApp
```

```
Name:              my-service
Namespace:         default
Labels:            app.kubernetes.io/name=MyApp
Annotations:       <none>
Selector:          app.kubernetes.io/name=MyApp
Type:              ClusterIP
IP Family Policy:  PreferDualStack
IP Families:       IPv4,IPv6
IP:                10.0.216.242
IPs:               10.0.216.242,2001:db8:fd00::af55
Port:              <unset>  80/TCP
TargetPort:        9376/TCP
Endpoints:         <none>
Session Affinity:  None
Events:            <none>
```

### Tạo một Service dual-stack có cân bằng tải (Create a dual-stack load balanced Service)

Nếu nhà cung cấp cloud hỗ trợ việc cấp phát (provision) các external load balancer đã bật IPv6,
hãy tạo Service dưới đây với `PreferDualStack` trong `.spec.ipFamilyPolicy`, `IPv6` là phần tử
đầu tiên của mảng `.spec.ipFamilies`, và trường `type` được đặt là `LoadBalancer`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  labels:
    app.kubernetes.io/name: MyApp
spec:
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
  - IPv6
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
```

Kiểm tra Service:

```shell
kubectl get svc -l app.kubernetes.io/name=MyApp
```

Hãy kiểm chứng rằng Service nhận được một địa chỉ `CLUSTER-IP` từ khối địa chỉ IPv6, kèm theo một
`EXTERNAL-IP`. Sau đó bạn có thể kiểm chứng việc truy cập tới service qua IP và port đó.

```
NAME         TYPE           CLUSTER-IP            EXTERNAL-IP        PORT(S)        AGE
my-service   LoadBalancer   2001:db8:fd00::7ebc   2603:1030:805::5   80:30790/TCP   35s
```

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Bài nêu ba điều kiện tiên quyết. Kể đủ ba, và giải thích vì sao thiếu một trong ba thì mọi
   bước kiểm chứng phía sau đều vô nghĩa.
2. Trên cluster lab, bạn đọc `.spec.podCIDRs` của node `lab-k8s-worker1` bằng `kubectl get nodes`
   với `-o go-template`. Kết quả ra mấy dòng, giá trị nằm trong dải nào, và điều đó chứng minh gì
   về cluster lab?
3. **Câu bẫy.** Trên một cluster dual-stack, bạn tạo Service với `ipFamilyPolicy: PreferDualStack`
   rồi chạy `kubectl get svc` và chỉ thấy **một** địa chỉ IPv4 ở cột `CLUSTER-IP`. Kết luận
   "Service này chỉ có IPv4" đúng hay sai? Nhìn vào đâu để biết chắc?
4. Bạn tạo một Service trên cluster dual-stack mà **không** khai báo `.spec.ipFamilyPolicy`.
   Kubernetes đặt giá trị nào cho `ipFamilyPolicy`, và ClusterIP lấy từ dải nào? Nếu muốn Service
   vẫn single-stack nhưng ClusterIP là IPv6 thì khai báo gì?
5. Kể ba đường mà bài đưa ra để nhìn thấy cả hai địa chỉ IP của một Pod dual-stack.

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **(a)** Nhà cung cấp hạ tầng phải cấp cho node các network interface IPv4/IPv6 **định tuyến
   được**; **(b)** network plugin phải **hỗ trợ mạng dual-stack**; **(c)** cluster phải **đã bật
   dual-stack**. Thiếu một trong ba thì cluster không bao giờ cấp được cặp địa chỉ hai họ, nên các
   lệnh kiểm chứng chỉ trả về **một** giá trị — không có gì để xác nhận.
2. Ra **đúng một dòng**, một khối `/24` nằm trong `10.244.0.0/16`. Bài đòi node dual-stack phải có
   **đúng một khối IPv4 và một khối IPv6**; ở đây thiếu hẳn khối IPv6, nên đó là **bằng chứng
   cluster lab là single-stack IPv4** — đúng như `kubeadm init` chỉ nhận một `--pod-network-cidr`
   IPv4.
3. **Sai.** Bài ghi rõ trong ô *Ghi chú*: `kubectl get svc` **chỉ hiển thị IP chính** ở cột
   `CLUSTER-IP`, dù Service có hai địa chỉ. Muốn biết chắc thì dùng **`kubectl describe svc`** và
   đọc dòng `IPs:` — nó liệt kê cả hai — cùng dòng `IP Families: IPv4,IPv6`; hoặc đọc thẳng trường
   `.spec.clusterIPs`.
4. Kubernetes đặt `.spec.ipFamilyPolicy` thành **`SingleStack`** và lấy ClusterIP từ **dải
   `service-cluster-ip-range` được cấu hình đầu tiên** — trong ví dụ của bài là dải IPv4. Muốn
   single-stack nhưng IPv6 thì khai báo tường minh **`ipFamilies` với `IPv6` là phần tử đầu**;
   `ipFamilyPolicy` vẫn là `SingleStack`, chỉ khác dải được chọn.
5. **(a)** `kubectl get pods` với `-o go-template` duyệt `.status.podIPs`; **(b)** Downward API với
   `fieldPath: status.podIPs` phơi ra một biến môi trường (ví dụ `MY_POD_IPS`), giá trị là danh
   sách **phân tách bằng dấu phẩy**; **(c)** `cat /etc/hosts` trong container — Pod dual-stack có
   **hai dòng** cho cùng một tên Pod, một IPv4 và một IPv6.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là **bài cuối của giai đoạn 21**:
tiếp theo hãy làm **Checkpoint** ghi ở cuối
[mục giai đoạn 21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy) trên cluster lab —
sửa Corefile của CoreDNS thêm một domain chuyển tiếp rồi kiểm chứng bằng `nslookup` từ trong Pod;
dùng quy trình gỡ lỗi DNS của bài [205](205-dns-debugging-resolution-vi.md) để tìm nguyên nhân một
Pod không phân giải được tên; viết một NetworkPolicy chặn toàn bộ ingress rồi mở đúng một cổng và
chứng minh bằng `curl` từ Pod khác. Xong Checkpoint mới sang
[giai đoạn 22](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu).

# Kiểm chứng dual-stack IPv4/IPv6 (Validate IPv4/IPv6 dual-stack)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/network/validate-dual-stack/>
>
> Tài liệu này hướng dẫn cách kiểm chứng các cluster Kubernetes đã bật dual-stack IPv4/IPv6.

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

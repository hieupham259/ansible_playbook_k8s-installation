# Sử dụng user namespace với Pod (Use a User Namespace With a Pod)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/user-namespaces/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Trang này hướng dẫn cách cấu hình user namespace cho các Pod. Điều này cho phép bạn cô lập
người dùng chạy bên trong container khỏi người dùng trên host.

Một tiến trình chạy với quyền root trong container có thể chạy dưới một người dùng khác
(không phải root) trên host; nói cách khác, tiến trình đó có đầy đủ đặc quyền đối với các thao
tác bên trong user namespace, nhưng không có đặc quyền đối với các thao tác bên ngoài
namespace đó.

Bạn có thể dùng tính năng này để giảm thiệt hại mà một container bị xâm nhập có thể gây ra cho
host hoặc cho các Pod khác trên cùng node. Có [một số lỗ hổng bảo mật][KEP-vulns] được đánh
giá mức **HIGH** hoặc **CRITICAL** đã không thể khai thác được khi user namespace được kích
hoạt. Người ta cũng kỳ vọng user namespace sẽ giảm nhẹ một số lỗ hổng trong tương lai.

Khi không dùng user namespace, một container chạy với quyền root — trong trường hợp thoát khỏi
container (container breakout) — sẽ có đặc quyền root trên node. Và nếu container được cấp một
capability nào đó, capability đó cũng có hiệu lực trên host. Không điều nào trong số này còn
đúng khi user namespace được sử dụng.

[KEP-vulns]: https://github.com/kubernetes/enhancements/tree/217d790720c5aef09b8bd4d6ca96284a0affe6c2/keps/sig-node/127-user-namespaces#motivation

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Server Kubernetes của bạn phải ở phiên bản v1.25 hoặc mới hơn. Để kiểm tra phiên bản, hãy nhập
`kubectl version`.

> **Ghi chú:** Mục này đề cập đến một sản phẩm hoặc dự án bên thứ ba cung cấp chức năng mà
> Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về sản phẩm hay dự
> án bên thứ ba đó. Xem
> [hướng dẫn website của CNCF](https://github.com/cncf/foundation/blob/master/website-guidelines.md)
> để biết thêm chi tiết.

* Hệ điều hành của node cần là Linux
* Bạn cần chạy được các lệnh trên host
* Bạn cần có khả năng exec vào các Pod

Cluster mà bạn đang dùng **bắt buộc** phải có ít nhất một node đáp ứng các
[yêu cầu](./55-user-namespaces-vi.md#trước-khi-bạn-bắt-đầu-before-you-begin)
để dùng user namespace với Pod.

Nếu bạn có nhiều loại node lẫn lộn và chỉ một số node hỗ trợ user namespace cho Pod, bạn cũng
cần đảm bảo rằng các Pod dùng user namespace được
[lập lịch (schedule)](./138-assign-pod-node-vi.md) đến các node phù hợp.

## Chạy một Pod dùng user namespace (Run a Pod that uses a user namespace) {#create-pod}

User namespace cho một Pod được bật bằng cách đặt trường `hostUsers` trong `.spec` thành
`false`. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: userns
spec:
  hostUsers: false
  containers:
  - name: shell
    command: ["sleep", "infinity"]
    image: debian
```

1. Tạo Pod trên cluster của bạn:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/user-namespaces-stateless.yaml
   ```

1. Exec vào Pod và chạy `readlink /proc/self/ns/user`:

   ```shell
   kubectl exec -ti userns -- bash
   ```

Chạy lệnh này:

```shell
readlink /proc/self/ns/user
```

Output tương tự như sau:

```shell
user:[4026531837]
```

Chạy thêm:

```shell
cat /proc/self/uid_map
```

Output tương tự như sau:

```shell
0  833617920      65536
```

Sau đó, mở một shell trên host và chạy các lệnh tương tự.

Lệnh `readlink` cho biết user namespace mà tiến trình đang chạy trong đó. Kết quả này phải
khác nhau khi chạy trên host và khi chạy bên trong container.

Con số cuối cùng trong file `uid_map` bên trong container phải là 65536, còn trên host thì nó
phải là một con số lớn hơn.

Nếu bạn đang chạy kubelet bên trong một user namespace, bạn cần so sánh output khi chạy lệnh
trong Pod với output khi chạy trên host:

```shell
readlink /proc/$pid/ns/user
```

trong đó thay `$pid` bằng PID của kubelet.

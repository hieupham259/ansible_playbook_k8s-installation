# Cấu hình Security Context cho Pod hoặc Container (Configure a Security Context for a Pod or Container)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/

Security context định nghĩa các thiết lập về đặc quyền (privilege) và kiểm soát truy cập
(access control) cho một Pod hoặc Container. Các thiết lập security context bao gồm, nhưng
không giới hạn ở:

* Discretionary Access Control (kiểm soát truy cập tùy quyền): Quyền truy cập vào một đối tượng,
  ví dụ một file, dựa trên
  [user ID (UID) và group ID (GID)](https://wiki.archlinux.org/index.php/users_and_groups).

* [Security Enhanced Linux (SELinux)](https://en.wikipedia.org/wiki/Security-Enhanced_Linux):
  Các đối tượng được gán nhãn bảo mật (security label).

* Chạy ở chế độ đặc quyền (privileged) hoặc không đặc quyền (unprivileged).

* [Linux Capabilities](https://linux-audit.com/linux-capabilities-hardening-linux-binaries-by-removing-setuid/):
  Cấp cho một tiến trình (process) một số đặc quyền, nhưng không phải toàn bộ đặc quyền của
  user root.

* [AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/):
  Dùng các profile chương trình để hạn chế khả năng của từng chương trình riêng lẻ.

* [Seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/): Lọc các system call
  của một tiến trình.

* `allowPrivilegeEscalation`: Kiểm soát việc một tiến trình có thể giành được nhiều đặc quyền
  hơn tiến trình cha của nó hay không. Giá trị boolean này trực tiếp quyết định việc flag
  [`no_new_privs`](https://www.kernel.org/doc/Documentation/prctl/no_new_privs.txt)
  có được đặt trên tiến trình của container hay không.
  `allowPrivilegeEscalation` luôn là true khi container:

  - chạy ở chế độ privileged, hoặc
  - có `CAP_SYS_ADMIN`

* `readOnlyRootFilesystem`: Mount root filesystem của container ở chế độ chỉ đọc (read-only).

Các gạch đầu dòng trên chưa phải là tập hợp đầy đủ các thiết lập security context — hãy xem
[SecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#securitycontext-v1-core)
để có danh sách toàn diện.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn v1.36. Để kiểm tra phiên bản, nhập
`kubectl version`.

## Đặt security context cho một Pod (Set the security context for a Pod) {#set-the-security-context-for-a-pod}

Để chỉ định các thiết lập bảo mật cho một Pod, hãy thêm field `securityContext` vào
đặc tả (specification) của Pod. Field `securityContext` là một object
[PodSecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podsecuritycontext-v1-core).
Các thiết lập bảo mật mà bạn chỉ định cho một Pod áp dụng cho tất cả các Container trong Pod đó.
Dưới đây là file cấu hình cho một Pod có `securityContext` và một volume kiểu `emptyDir`:

[`pods/security/security-context.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/security/security-context.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    supplementalGroups: [4000]
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox:1.28
    command: [ "sh", "-c", "sleep 1h" ]
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo
    securityContext:
      allowPrivilegeEscalation: false
```

Trong file cấu hình này, field `runAsUser` chỉ định rằng với mọi Container trong Pod, tất cả
các tiến trình đều chạy với user ID 1000. Field `runAsGroup` chỉ định group ID chính (primary
group ID) là 3000 cho tất cả các tiến trình bên trong mọi container của Pod. Nếu field này bị
bỏ qua, group ID chính của các container sẽ là root(0). Mọi file được tạo ra cũng sẽ thuộc sở
hữu của user 1000 và group 3000 khi `runAsGroup` được chỉ định. Vì field `fsGroup` được chỉ
định, tất cả tiến trình của container cũng thuộc nhóm bổ sung (supplementary group) có ID 2000.
Chủ sở hữu của volume `/data/demo` và mọi file được tạo trong volume đó sẽ là Group ID 2000.
Ngoài ra, khi field `supplementalGroups` được chỉ định, tất cả tiến trình của container cũng
thuộc các group được chỉ định. Nếu field này bị bỏ qua, nó được coi là rỗng.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/security/security-context.yaml
```

Xác nhận rằng Container của Pod đang chạy:

```shell
kubectl get pod security-context-demo
```

Mở một shell vào Container đang chạy:

```shell
kubectl exec -it security-context-demo -- sh
```

Trong shell của bạn, liệt kê các tiến trình đang chạy:

```shell
ps
```

Kết quả cho thấy các tiến trình đang chạy dưới user 1000, chính là giá trị của `runAsUser`:

```none
PID   USER     TIME  COMMAND
    1 1000      0:00 sleep 1h
    6 1000      0:00 sh
...
```

Trong shell của bạn, di chuyển tới `/data` và liệt kê thư mục duy nhất trong đó:

```shell
cd /data
ls -l
```

Kết quả cho thấy thư mục `/data/demo` có group ID 2000, chính là giá trị của `fsGroup`.

```none
drwxrwsrwx 2 root 2000 4096 Jun  6 20:08 demo
```

Trong shell của bạn, di chuyển tới `/data/demo` và tạo một file:

```shell
cd demo
echo hello > testfile
```

Liệt kê file trong thư mục `/data/demo`:

```shell
ls -l
```

Kết quả cho thấy `testfile` có group ID 2000, chính là giá trị của `fsGroup`.

```none
-rw-r--r-- 1 1000 2000 6 Jun  6 20:08 testfile
```

Chạy lệnh sau:

```shell
id
```

Kết quả tương tự như sau:

```none
uid=1000 gid=3000 groups=2000,3000,4000
```

Từ kết quả trên, bạn có thể thấy `gid` là 3000, giống với giá trị của field `runAsGroup`.
Nếu `runAsGroup` bị bỏ qua, `gid` sẽ vẫn là 0 (root) và tiến trình sẽ có thể tương tác với
các file thuộc sở hữu của group root(0) và các group có quyền cần thiết đối với group root
(0). Bạn cũng có thể thấy rằng `groups` chứa các group ID được chỉ định bởi `fsGroup` và
`supplementalGroups`, bên cạnh `gid`.

Thoát khỏi shell của bạn:

```shell
exit
```

### Tư cách thành viên group ngầm định được định nghĩa trong `/etc/group` của container image (Implicit group memberships defined in `/etc/group` in the container image)

Theo mặc định, Kubernetes gộp (merge) thông tin group từ Pod với thông tin được định nghĩa
trong `/etc/group` của container image.

[`pods/security/security-context-5.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/security/security-context-5.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    supplementalGroups: [4000]
  containers:
  - name: sec-ctx-demo
    image: registry.k8s.io/e2e-test-images/agnhost:2.45
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      allowPrivilegeEscalation: false
```

Security context của Pod này chứa `runAsUser`, `runAsGroup` và `supplementalGroups`.
Tuy nhiên, bạn sẽ thấy rằng các nhóm bổ sung thực tế được gắn vào tiến trình của container
sẽ bao gồm cả các group ID đến từ `/etc/group` trong container image.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/security/security-context-5.yaml
```

Xác nhận rằng Container của Pod đang chạy:

```shell
kubectl get pod security-context-demo
```

Mở một shell vào Container đang chạy:

```shell
kubectl exec -it security-context-demo -- sh
```

Kiểm tra danh tính (identity) của tiến trình:

```shell
id
```

Kết quả tương tự như sau:

```none
uid=1000 gid=3000 groups=3000,4000,50000
```

Bạn có thể thấy rằng `groups` bao gồm group ID `50000`. Đó là vì user (`uid=1000`), vốn được
định nghĩa trong image, thuộc về group (`gid=50000`) được định nghĩa trong `/etc/group` bên
trong container image.

Kiểm tra `/etc/group` trong container image:

```shell
cat /etc/group
```

Bạn có thể thấy uid `1000` thuộc về group `50000`.

```none
...
user-defined-in-image:x:1000:
group-defined-in-image:x:50000:user-defined-in-image
```

Thoát khỏi shell của bạn:

```shell
exit
```

> **Ghi chú:** Các nhóm bổ sung được _gộp ngầm định_ có thể gây ra các vấn đề bảo mật, đặc
> biệt là khi truy cập các volume (xem
> [kubernetes/kubernetes#112879](https://issue.k8s.io/112879) để biết chi tiết).
> Nếu bạn muốn tránh điều này, hãy xem mục bên dưới.

## Cấu hình kiểm soát SupplementalGroups chi tiết cho một Pod (Configure fine-grained SupplementalGroups control for a Pod) {#supplementalgroupspolicy}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [beta]`

Tính năng này có thể được bật bằng cách đặt
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`SupplementalGroupsPolicy` cho kubelet và kube-apiserver, đồng thời đặt field
`.spec.securityContext.supplementalGroupsPolicy` cho pod.

Field `supplementalGroupsPolicy` định nghĩa chính sách tính toán các nhóm bổ sung cho các
tiến trình container trong một pod. Field này có hai giá trị hợp lệ:

* `Merge`: Tư cách thành viên group được định nghĩa trong `/etc/group` cho user chính của
  container sẽ được gộp vào. Đây là chính sách mặc định nếu không được chỉ định.

* `Strict`: Chỉ các group ID trong các field `fsGroup`, `supplementalGroups` hoặc `runAsGroup`
  được gắn làm nhóm bổ sung cho các tiến trình container.
  Điều này có nghĩa là không có tư cách thành viên group nào từ `/etc/group` của user chính
  của container được gộp vào.

Khi tính năng này được bật, nó cũng cho thấy danh tính tiến trình được gắn vào tiến trình
container đầu tiên trong field `.status.containerStatuses[].user.linux`. Điều này hữu ích để
phát hiện xem có group ID ngầm định nào được gắn vào hay không.

[`pods/security/security-context-6.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/security/security-context-6.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    supplementalGroups: [4000]
    supplementalGroupsPolicy: Strict
  containers:
  - name: sec-ctx-demo
    image: registry.k8s.io/e2e-test-images/agnhost:2.45
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      allowPrivilegeEscalation: false
```

Manifest của pod này định nghĩa `supplementalGroupsPolicy=Strict`. Bạn có thể thấy rằng không
có tư cách thành viên group nào định nghĩa trong `/etc/group` được gộp vào các nhóm bổ sung
của các tiến trình container.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/security/security-context-6.yaml
```

Xác nhận rằng Container của Pod đang chạy:

```shell
kubectl get pod security-context-demo
```

Kiểm tra danh tính của tiến trình:

```shell
kubectl exec -it security-context-demo -- id
```

Kết quả tương tự như sau:

```none
uid=1000 gid=3000 groups=3000,4000
```

Xem trạng thái (status) của Pod:

```shell
kubectl get pod security-context-demo -o yaml
```

Bạn có thể thấy field `status.containerStatuses[].user.linux` cho biết danh tính tiến trình
được gắn vào tiến trình container đầu tiên.

```none
...
status:
  containerStatuses:
  - name: sec-ctx-demo
    user:
      linux:
        gid: 3000
        supplementalGroups:
        - 3000
        - 4000
        uid: 1000
...
```

> **Ghi chú:** Xin lưu ý rằng các giá trị trong field `status.containerStatuses[].user.linux`
> là danh tính tiến trình _được gắn đầu tiên_ cho tiến trình container đầu tiên trong
> container. Nếu container có đủ đặc quyền để thực hiện các system call liên quan tới danh
> tính tiến trình (ví dụ [`setuid(2)`](https://man7.org/linux/man-pages/man2/setuid.2.html),
> [`setgid(2)`](https://man7.org/linux/man-pages/man2/setgid.2.html) hoặc
> [`setgroups(2)`](https://man7.org/linux/man-pages/man2/setgroups.2.html), v.v.),
> thì tiến trình container có thể thay đổi danh tính của nó. Do đó, danh tính tiến trình
> _thực tế_ sẽ mang tính động (dynamic).

### Các triển khai (Implementations) {#implementations-supplementalgroupspolicy}

> **Ghi chú:** Mục này liên kết tới các dự án bên thứ ba cung cấp chức năng mà Kubernetes
> cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được
> liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc
> [hướng dẫn nội dung](https://kubernetes.io/docs/contribute/style/content-guide/) trước khi
> gửi thay đổi.

Các container runtime sau đây được biết là hỗ trợ kiểm soát SupplementalGroups chi tiết.

Ở mức CRI:
- [containerd](https://containerd.io/), từ v2.0
- [CRI-O](https://cri-o.io/), từ v1.31

Bạn có thể xem tính năng này có được hỗ trợ hay không trong trạng thái của Node.

```yaml
apiVersion: v1
kind: Node
...
status:
  features:
    supplementalGroupsPolicy: true
```

> **Ghi chú:** Ở bản phát hành alpha (từ v1.31 đến v1.32), khi một pod với
> `SupplementalGroupsPolicy=Strict` được lập lịch (schedule) lên một node KHÔNG hỗ trợ tính
> năng này (tức là `.status.features.supplementalGroupsPolicy=false`), chính sách nhóm bổ
> sung của pod sẽ _âm thầm_ rơi về (fall back) chính sách `Merge`.
>
> Tuy nhiên, kể từ bản phát hành beta (v1.33), để thực thi chính sách chặt chẽ hơn, __việc
> tạo pod như vậy sẽ bị kubelet từ chối vì node không thể bảo đảm chính sách được chỉ
> định__. Khi pod của bạn bị từ chối, bạn sẽ thấy các event cảnh báo với
> `reason=SupplementalGroupsPolicyNotSupported` như dưới đây:
>
> ```yaml
> apiVersion: v1
> kind: Event
> ...
> type: Warning
> reason: SupplementalGroupsPolicyNotSupported
> message: "SupplementalGroupsPolicy=Strict is not supported in this node"
> involvedObject:
>   apiVersion: v1
>   kind: Pod
>   ...
> ```

## Cấu hình chính sách thay đổi quyền và quyền sở hữu volume cho Pod (Configure volume permission and ownership change policy for Pods)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [stable]`

Theo mặc định, Kubernetes thay đổi một cách đệ quy quyền sở hữu (ownership) và quyền
(permission) đối với nội dung của từng volume để khớp với `fsGroup` được chỉ định trong
`securityContext` của Pod khi volume đó được mount.
Với các volume lớn, việc kiểm tra và thay đổi quyền sở hữu cùng quyền truy cập có thể mất
rất nhiều thời gian, làm chậm quá trình khởi động Pod. Bạn có thể dùng field
`fsGroupChangePolicy` bên trong `securityContext` để kiểm soát cách Kubernetes kiểm tra và
quản lý quyền sở hữu cùng quyền truy cập cho một volume.

**fsGroupChangePolicy** - `fsGroupChangePolicy` định nghĩa hành vi thay đổi quyền sở hữu và
  quyền của volume trước khi được đưa vào bên trong một Pod.
  Field này chỉ áp dụng cho các loại volume hỗ trợ kiểm soát quyền sở hữu và quyền bằng
  `fsGroup`. Field này có hai giá trị khả dĩ:

* _OnRootMismatch_: Chỉ thay đổi quyền và quyền sở hữu nếu quyền và quyền sở hữu của thư mục
  gốc không khớp với quyền mong đợi của volume.
  Điều này có thể giúp rút ngắn thời gian cần để thay đổi quyền sở hữu và quyền của một volume.
* _Always_: Luôn thay đổi quyền và quyền sở hữu của volume mỗi khi volume được mount.

Ví dụ:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 3000
  fsGroup: 2000
  fsGroupChangePolicy: "OnRootMismatch"
```

> **Ghi chú:** Field này không có tác dụng đối với các loại volume tạm thời (ephemeral) như
> [`secret`](91-volumes-vi.md#secret),
> [`configMap`](91-volumes-vi.md#configmap)
> và [`emptyDir`](91-volumes-vi.md#emptydir).

## Ủy quyền việc thay đổi quyền và quyền sở hữu volume cho CSI driver (Delegating volume permission and ownership change to CSI driver)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

Nếu bạn triển khai một driver
[Container Storage Interface (CSI)](https://github.com/container-storage-interface/spec/blob/master/spec.md)
hỗ trợ `NodeServiceCapability` `VOLUME_MOUNT_GROUP`, thì quá trình đặt quyền sở hữu và quyền
của file dựa trên `fsGroup` được chỉ định trong `securityContext` sẽ do CSI driver thực hiện
thay vì Kubernetes. Trong trường hợp này, vì Kubernetes không thực hiện bất kỳ thay đổi quyền
sở hữu và quyền nào, `fsGroupChangePolicy` không có hiệu lực, và như đặc tả CSI quy định,
driver được kỳ vọng sẽ mount volume với `fsGroup` được cung cấp, tạo ra một volume mà
`fsGroup` có thể đọc/ghi được.

## Đặt security context cho một Container (Set the security context for a Container)

Để chỉ định các thiết lập bảo mật cho một Container, hãy thêm field `securityContext` vào
manifest của Container. Field `securityContext` là một object
[SecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#securitycontext-v1-core).
Các thiết lập bảo mật mà bạn chỉ định cho một Container chỉ áp dụng cho riêng Container đó,
và chúng ghi đè các thiết lập ở cấp Pod khi có sự trùng lặp. Thiết lập ở cấp Container không
ảnh hưởng đến các Volume của Pod.

Dưới đây là file cấu hình cho một Pod có một Container. Cả Pod và Container đều có field
`securityContext`:

[`pods/security/security-context-2.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/security/security-context-2.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-2
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: sec-ctx-demo-2
    image: gcr.io/google-samples/hello-app:2.0
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/security/security-context-2.yaml
```

Xác nhận rằng Container của Pod đang chạy:

```shell
kubectl get pod security-context-demo-2
```

Mở một shell vào Container đang chạy:

```shell
kubectl exec -it security-context-demo-2 -- sh
```

Trong shell của bạn, liệt kê các tiến trình đang chạy:

```shell
ps aux
```

Kết quả cho thấy các tiến trình đang chạy dưới user 2000. Đây là giá trị của `runAsUser`
được chỉ định cho Container. Nó ghi đè giá trị 1000 được chỉ định cho Pod.

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
2000         1  0.0  0.0   4336   764 ?        Ss   20:36   0:00 /bin/sh -c node server.js
2000         8  0.1  0.5 772124 22604 ?        Sl   20:36   0:00 node server.js
...
```

Thoát khỏi shell của bạn:

```shell
exit
```

## Đặt capabilities cho một Container (Set capabilities for a Container) {#set-capabilities-for-a-container}

Với [Linux capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html),
bạn có thể cấp một số đặc quyền nhất định cho một tiến trình mà không cần cấp toàn bộ đặc
quyền của user root. Để thêm hoặc bỏ Linux capabilities cho một Container, hãy thêm field
`capabilities` vào mục `securityContext` trong manifest của Container.

Trước tiên, hãy xem điều gì xảy ra khi bạn không thêm field `capabilities`.
Dưới đây là file cấu hình không thêm hay bỏ bất kỳ capability nào của Container:

[`pods/security/security-context-3.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/security/security-context-3.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-3
spec:
  containers:
  - name: sec-ctx-3
    image: gcr.io/google-samples/hello-app:2.0
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/security/security-context-3.yaml
```

Xác nhận rằng Container của Pod đang chạy:

```shell
kubectl get pod security-context-demo-3
```

Mở một shell vào Container đang chạy:

```shell
kubectl exec -it security-context-demo-3 -- sh
```

Trong shell của bạn, liệt kê các tiến trình đang chạy:

```shell
ps aux
```

Kết quả cho thấy các process ID (PID) của Container:

```
USER  PID %CPU %MEM    VSZ   RSS TTY   STAT START   TIME COMMAND
root    1  0.0  0.0   4336   796 ?     Ss   18:17   0:00 /bin/sh -c node server.js
root    5  0.1  0.5 772124 22700 ?     Sl   18:17   0:00 node server.js
```

Trong shell của bạn, xem trạng thái của tiến trình 1:

```shell
cd /proc/1
cat status
```

Kết quả cho thấy bitmap capability của tiến trình:

```
...
CapPrm:	00000000a80425fb
CapEff:	00000000a80425fb
...
```

Ghi lại bitmap capability này, rồi thoát khỏi shell của bạn:

```shell
exit
```

Tiếp theo, chạy một Container giống hệt container trước, ngoại trừ việc nó được đặt thêm
các capability bổ sung.

Dưới đây là file cấu hình cho một Pod chạy một Container. Cấu hình này thêm các capability
`CAP_NET_ADMIN` và `CAP_SYS_TIME`:

[`pods/security/security-context-4.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/security/security-context-4.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-4
spec:
  containers:
  - name: sec-ctx-4
    image: gcr.io/google-samples/hello-app:2.0
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/security/security-context-4.yaml
```

Mở một shell vào Container đang chạy:

```shell
kubectl exec -it security-context-demo-4 -- sh
```

Trong shell của bạn, xem các capability của tiến trình 1:

```shell
cd /proc/1
cat status
```

Kết quả cho thấy bitmap capability của tiến trình:

```
...
CapPrm:	00000000aa0435fb
CapEff:	00000000aa0435fb
...
```

So sánh capability của hai Container:

```
00000000a80425fb
00000000aa0435fb
```

Trong bitmap capability của container thứ nhất, các bit 12 và 25 không được đặt. Trong
container thứ hai, các bit 12 và 25 được đặt. Bit 12 là `CAP_NET_ADMIN`, và bit 25 là
`CAP_SYS_TIME`. Xem
[capability.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/capability.h)
để biết định nghĩa của các hằng số capability.

> **Ghi chú:** Các hằng số capability của Linux có dạng `CAP_XXX`.
> Nhưng khi bạn liệt kê capability trong manifest của container, bạn phải bỏ phần `CAP_`
> của hằng số. Ví dụ, để thêm `CAP_SYS_TIME`, hãy đưa `SYS_TIME` vào danh sách capability
> của bạn.

## Đặt Seccomp Profile cho một Container (Set the Seccomp Profile for a Container) {#set-the-seccomp-profile-for-a-container}

Để đặt Seccomp profile cho một Container, hãy thêm field `seccompProfile` vào mục
`securityContext` trong manifest của Pod hoặc Container. Field `seccompProfile` là một object
[SeccompProfile](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#seccompprofile-v1-core)
gồm `type` và `localhostProfile`. Các giá trị hợp lệ cho `type` bao gồm `RuntimeDefault`,
`Unconfined` và `Localhost`. `localhostProfile` chỉ được đặt khi `type: Localhost`. Nó chỉ ra
đường dẫn của profile đã được cấu hình sẵn trên node, tương đối so với vị trí Seccomp profile
được cấu hình của kubelet (cấu hình bằng flag `--root-dir`).

Dưới đây là ví dụ đặt Seccomp profile thành profile mặc định của container runtime trên node:

```yaml
...
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

Dưới đây là ví dụ đặt Seccomp profile thành một file đã cấu hình sẵn tại
`<kubelet-root-dir>/seccomp/my-profiles/profile-allow.json`:

```yaml
...
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: my-profiles/profile-allow.json
```

## Đặt AppArmor Profile cho một Container (Set the AppArmor Profile for a Container)

Để đặt AppArmor profile cho một Container, hãy thêm field `appArmorProfile` vào mục
`securityContext` của Container. Field `appArmorProfile` là một object
[AppArmorProfile](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#apparmorprofile-v1-core)
gồm `type` và `localhostProfile`. Các giá trị hợp lệ cho `type` bao gồm `RuntimeDefault`
(mặc định), `Unconfined` và `Localhost`. `localhostProfile` chỉ được đặt khi `type` là
`Localhost`. Nó chỉ ra tên của profile đã được cấu hình sẵn trên node. Profile này cần được
nạp lên tất cả các node phù hợp với Pod, vì bạn không biết trước pod sẽ được lập lịch lên
node nào.
Các cách thiết lập profile tùy chỉnh được thảo luận trong
[Thiết lập node với các profile](https://kubernetes.io/docs/tutorials/security/apparmor/#setting-up-nodes-with-profiles).

Lưu ý: Nếu `containers[*].securityContext.appArmorProfile.type` được đặt tường minh là
`RuntimeDefault`, thì Pod sẽ không được chấp nhận (admit) nếu AppArmor không được bật trên
Node. Tuy nhiên, nếu `containers[*].securityContext.appArmorProfile.type` không được chỉ
định, thì giá trị mặc định (cũng là `RuntimeDefault`) sẽ chỉ được áp dụng nếu node đã bật
AppArmor. Nếu node tắt AppArmor thì Pod vẫn được chấp nhận nhưng Container sẽ không bị giới
hạn bởi profile `RuntimeDefault`.

Dưới đây là ví dụ đặt AppArmor profile thành profile mặc định của container runtime trên node:

```yaml
...
containers:
- name: container-1
  securityContext:
    appArmorProfile:
      type: RuntimeDefault
```

Dưới đây là ví dụ đặt AppArmor profile thành một profile đã cấu hình sẵn có tên
`k8s-apparmor-example-deny-write`:

```yaml
...
containers:
- name: container-1
  securityContext:
    appArmorProfile:
      type: Localhost
      localhostProfile: k8s-apparmor-example-deny-write
```

Để biết thêm chi tiết, hãy xem
[Hạn chế quyền truy cập tài nguyên của Container bằng AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/).

## Gán nhãn SELinux cho một Container (Assign SELinux labels to a Container) {#assign-selinux-labels-to-a-container}

Để gán nhãn SELinux cho một Container, hãy thêm field `seLinuxOptions` vào mục
`securityContext` trong manifest của Pod hoặc Container. Field `seLinuxOptions` là một object
[SELinuxOptions](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#selinuxoptions-v1-core).
Dưới đây là ví dụ áp dụng một SELinux level:

```yaml
...
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

> **Ghi chú:** Để gán nhãn SELinux, module bảo mật SELinux phải được nạp trên hệ điều hành
> của host. Trên các worker node Windows và Linux không hỗ trợ SELinux, field này và mọi
> feature gate SELinux được mô tả bên dưới đều không có tác dụng.

### Gán lại nhãn SELinux cho volume một cách hiệu quả (Efficient SELinux volume relabeling)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

> **Ghi chú:** Kubernetes v1.27 đã giới thiệu một dạng ban đầu, còn giới hạn, của hành vi
> này, chỉ áp dụng cho các volume (và PersistentVolumeClaim) dùng access mode
> `ReadWriteOncePod`.
>
> Kubernetes v1.36 nâng các
> [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
> `SELinuxChangePolicy` và `SELinuxMount` lên GA để mở rộng cải thiện hiệu năng đó cho các
> loại PersistentVolumeClaim khác, như được giải thích chi tiết bên dưới. `SELinuxMount` vẫn
> bị tắt theo mặc định.

Khi feature gate `SELinuxMount` bị tắt (mặc định trong Kubernetes 1.36 và mọi bản phát hành
trước đó), container runtime mặc định sẽ gán nhãn SELinux một cách đệ quy cho tất cả các
file trên tất cả các volume của Pod. Để tăng tốc quá trình này, Kubernetes có thể thay đổi
nhãn SELinux của một volume ngay lập tức bằng cách dùng mount option `-o context=<label>`.

Để hưởng lợi từ việc tăng tốc này, tất cả các điều kiện sau phải được thỏa mãn:

* Pod phải dùng PersistentVolumeClaim với `accessModes` phù hợp và các
  [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/):
  * Hoặc volume có `accessModes: ["ReadWriteOncePod"]`.
  * Hoặc volume có thể dùng bất kỳ access mode nào khác, feature gate `SELinuxMount` được
    bật, và Pod có `spec.securityContext.seLinuxChangePolicy` là nil (mặc định) hoặc
    `MountOption`.
* Pod (hoặc tất cả các Container của nó dùng PersistentVolumeClaim đó) phải có
  `seLinuxOptions` được đặt.
* PersistentVolume tương ứng phải là một trong hai loại:
  * Một volume dùng loại volume in-tree kiểu cũ (legacy) `iscsi`, `rbd` hoặc `fc`.
  * Hoặc một volume dùng CSI driver. CSI driver phải công bố rằng nó hỗ trợ mount với
    `-o context` bằng cách đặt `spec.seLinuxMount: true` trong instance CSIDriver của nó.

Khi bất kỳ điều kiện nào trong số này không được thỏa mãn, việc gán lại nhãn SELinux diễn ra
theo cách khác: container runtime thay đổi đệ quy nhãn SELinux cho tất cả các inode (file và
thư mục) trong volume. Nói rõ hơn, điều này áp dụng cho các volume tạm thời (ephemeral) của
Kubernetes như `secret`, `configMap` và `projected`, cũng như tất cả các volume mà instance
CSIDriver của chúng không công bố tường minh việc mount với `-o context`.

Khi cơ chế tăng tốc này được dùng, tất cả các Pod dùng đồng thời cùng một volume áp dụng được
trên cùng một node **phải có cùng nhãn SELinux**. Một Pod có nhãn SELinux khác sẽ không khởi
động được và sẽ ở trạng thái `ContainerCreating` cho đến khi tất cả các Pod có nhãn SELinux
khác đang dùng volume đó bị xóa.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Với các Pod muốn từ chối (opt-out) việc gán lại nhãn bằng mount option, chúng có thể đặt
`spec.securityContext.seLinuxChangePolicy` thành `Recursive`. Điều này là bắt buộc khi nhiều
pod chia sẻ một volume duy nhất trên cùng một node, nhưng chúng chạy với các nhãn SELinux khác
nhau vẫn cho phép truy cập đồng thời vào volume. Ví dụ, một pod đặc quyền chạy với nhãn
`spc_t` và một pod không đặc quyền chạy với nhãn mặc định `container_file_t`.
Khi `spec.securityContext.seLinuxChangePolicy` không được đặt (hoặc với giá trị mặc định
`MountOption`), chỉ một trong các pod như vậy có thể chạy trên một node, pod còn lại sẽ bị
ContainerCreating với lỗi
`conflicting SELinux labels of volume <name of the volume>: <label of the running pod> and <label of the pod that can't start>`.

#### SELinuxWarningController

Để giúp dễ dàng nhận diện các Pod bị ảnh hưởng bởi thay đổi trong việc gán lại nhãn SELinux
cho volume, một controller mới tên là `SELinuxWarningController` đã được đưa vào
kube-controller-manager. Nó bị tắt theo mặc định và có thể được bật bằng cách đặt
[flag dòng lệnh](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)
`--controllers=*,selinux-warning-controller`, hoặc bằng cách đặt field
`genericControllerManagerConfiguration.controllers`
[trong KubeControllerManagerConfiguration](https://kubernetes.io/docs/reference/config-api/kube-controller-manager-config.v1alpha1/#controllermanager-config-k8s-io-v1alpha1-GenericControllerManagerConfiguration).
Controller này yêu cầu feature gate `SELinuxChangePolicy` được bật.

Khi được bật, controller sẽ quan sát các Pod đang chạy và khi phát hiện hai Pod dùng cùng
một volume với nhãn SELinux khác nhau:

1. Nó phát ra một event cho cả hai Pod. `kubectl describe pod <pod-name>` khi đó hiển thị
   `SELinuxLabel "<label on the pod>" conflicts with pod <the other pod name> that uses the same volume as this pod
   with SELinuxLabel "<the other pod label>". If both pods land on the same node, only one of them may access the volume`.
2. Tăng metric `selinux_warning_controller_selinux_volume_conflict`. Metric này có cả tên
   pod + namespace làm label để dễ dàng nhận diện các pod bị ảnh hưởng.

Quản trị viên cluster có thể dùng thông tin này để nhận diện các pod bị ảnh hưởng bởi thay
đổi được lên kế hoạch và chủ động cho các Pod từ chối cơ chế tối ưu hóa này (tức là đặt
`spec.securityContext.seLinuxChangePolicy: Recursive`).

> **Cảnh báo:** Chúng tôi đặc biệt khuyến nghị các cluster có dùng SELinux hãy bật controller
> này và bảo đảm metric `selinux_warning_controller_selinux_volume_conflict` không báo cáo
> bất kỳ xung đột nào trước khi bật feature gate `SELinuxMount` hoặc nâng cấp lên phiên bản
> mà `SELinuxMount` được bật theo mặc định.

#### Feature gates

Các feature gate sau kiểm soát hành vi gán lại nhãn SELinux cho volume:

* `SELinuxMountReadWriteOncePod`: bật tối ưu hóa cho các volume có
  `accessModes: ["ReadWriteOncePod"]`. Đây là một feature gate rất an toàn để bật, vì không
  thể xảy ra trường hợp hai pod cùng chia sẻ một volume với access mode này. Feature gate
  này được bật theo mặc định kể từ 1.28 và là GA ở 1.36.
* `SELinuxChangePolicy`: bật field `spec.securityContext.seLinuxChangePolicy` trong Pod và
  SELinuxWarningController liên quan trong kube-controller-manager. Tính năng này có thể
  được dùng trước khi bật `SELinuxMount` để kiểm tra các Pod đang chạy trên cluster, và để
  chủ động cho các Pod từ chối cơ chế tối ưu hóa.
  Feature gate này yêu cầu `SELinuxMountReadWriteOncePod` được bật. Nó ở trạng thái beta và
  được bật theo mặc định kể từ 1.33 và là GA ở 1.36.
* `SELinuxMount` bật tối ưu hóa cho tất cả các volume đủ điều kiện. Vì nó có thể làm hỏng
  các workload hiện có, chúng tôi khuyến nghị bật feature gate `SELinuxChangePolicy` +
  SELinuxWarningController trước để kiểm tra tác động của thay đổi.
  Feature gate này yêu cầu `SELinuxMountReadWriteOncePod` và `SELinuxChangePolicy` được bật.
  Nó ở trạng thái beta, nhưng bị tắt theo mặc định ở 1.33.

## Quản lý truy cập vào filesystem `/proc` (Managing access to the `/proc` filesystem) {#proc-access}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Với các runtime tuân theo đặc tả OCI runtime, các container mặc định chạy ở chế độ mà trong
đó có nhiều đường dẫn vừa bị che (masked) vừa ở chế độ chỉ đọc (read-only).
Kết quả là container có các đường dẫn này hiện diện bên trong mount namespace của container,
và chúng có thể hoạt động tương tự như khi container là một host tách biệt, nhưng tiến trình
của container không thể ghi vào chúng. Danh sách các đường dẫn bị che và chỉ đọc như sau:

- Đường dẫn bị che (Masked Paths):
  - `/proc/asound`
  - `/proc/acpi`
  - `/proc/kcore`
  - `/proc/keys`
  - `/proc/latency_stats`
  - `/proc/timer_list`
  - `/proc/timer_stats`
  - `/proc/sched_debug`
  - `/proc/scsi`
  - `/sys/firmware`
  - `/sys/devices/virtual/powercap`

- Đường dẫn chỉ đọc (Read-Only Paths):
  - `/proc/bus`
  - `/proc/fs`
  - `/proc/irq`
  - `/proc/sys`
  - `/proc/sysrq-trigger`

Với một số Pod, bạn có thể muốn bỏ qua việc che các đường dẫn mặc định đó.
Bối cảnh phổ biến nhất cho nhu cầu này là khi bạn cố gắng chạy container bên trong một
container của Kubernetes (bên trong một pod).

Field `procMount` của `securityContext` cho phép người dùng yêu cầu `/proc` của container
được `Unmasked` (không bị che), hoặc được mount ở chế độ đọc-ghi bởi tiến trình của
container. Điều này cũng áp dụng cho `/sys/firmware`, vốn không nằm trong `/proc`.

```yaml
...
securityContext:
  procMount: Unmasked
```

> **Ghi chú:** Việc đặt `procMount` thành Unmasked yêu cầu giá trị `spec.hostUsers` trong
> spec của pod phải là `false`. Nói cách khác: một container muốn có `/proc` không bị che
> hoặc `/sys` không bị che thì cũng phải nằm trong một
> [user namespace](55-user-namespaces-vi.md).
> Kubernetes v1.12 đến v1.29 đã không thực thi yêu cầu này.

## Thảo luận (Discussion)

Security context của một Pod áp dụng cho các Container của Pod và cũng áp dụng cho các
Volume của Pod khi phù hợp. Cụ thể, `fsGroup` và `seLinuxOptions` được áp dụng cho Volume
như sau:

* `fsGroup`: Các Volume hỗ trợ quản lý quyền sở hữu sẽ được sửa đổi để thuộc sở hữu và có
  thể ghi được bởi GID được chỉ định trong `fsGroup`. Xem
  [tài liệu thiết kế Ownership Management](https://git.k8s.io/design-proposals-archive/storage/volume-ownership-management.md)
  để biết thêm chi tiết.

* `seLinuxOptions`: Các Volume hỗ trợ gán nhãn SELinux sẽ được gán lại nhãn để có thể truy
  cập được bởi nhãn được chỉ định trong `seLinuxOptions`. Thông thường, bạn chỉ cần đặt mục
  `level`. Mục này đặt nhãn
  [Multi-Category Security (MCS)](https://selinuxproject.org/page/NB_MLS)
  cho tất cả các Container trong Pod cũng như các Volume.

> **Cảnh báo:** Sau khi bạn chỉ định một nhãn MCS cho một Pod, tất cả các Pod có cùng nhãn
> đều có thể truy cập Volume đó. Nếu bạn cần bảo vệ giữa các Pod với nhau, bạn phải gán một
> nhãn MCS duy nhất cho từng Pod.

## Dọn dẹp (Clean up)

Xóa các Pod:

```shell
kubectl delete pod security-context-demo
kubectl delete pod security-context-demo-2
kubectl delete pod security-context-demo-3
kubectl delete pod security-context-demo-4
```

## Tiếp theo (What's next)

* [PodSecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podsecuritycontext-v1-core)
* [SecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#securitycontext-v1-core)
* [Hướng dẫn cấu hình CRI Plugin](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)
* [Tài liệu thiết kế Security Contexts](https://git.k8s.io/design-proposals-archive/auth/security_context.md)
* [Tài liệu thiết kế Ownership Management](https://git.k8s.io/design-proposals-archive/storage/volume-ownership-management.md)
* [PodSecurity Admission](116-pod-security-admission-vi.md)
* [Tài liệu thiết kế AllowPrivilegeEscalation](https://git.k8s.io/design-proposals-archive/auth/no-new-privs.md)
* Để biết thêm thông tin về các cơ chế bảo mật trong Linux, xem
  [Tổng quan về các tính năng bảo mật của Linux Kernel](https://www.linux.com/learn/overview-linux-kernel-security-features)
  (Lưu ý: Một số thông tin đã lỗi thời)
* Đọc về [User Namespaces](55-user-namespaces-vi.md)
  cho các pod Linux.
* [Masked Paths trong đặc tả OCI Runtime](https://github.com/opencontainers/runtime-spec/blob/f66aad47309/config-linux.md#masked-paths)

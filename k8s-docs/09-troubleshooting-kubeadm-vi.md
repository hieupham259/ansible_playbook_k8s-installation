# Xử lý sự cố kubeadm (Troubleshooting kubeadm)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm), bài 9/9 ·
Kiểm chứng ở [Lab 8a](labs/LAB-8A-DUNG-CLUSTER-BANG-KUBEADM.md).

**Đây là tài liệu tra cứu, không phải bài học — đừng đọc tuần tự và đừng cố nhớ nội dung.**
Lộ trình đánh dấu rõ như vậy. Cách dùng đúng: lướt **mục lục** một lần, đọc tên từng mục `##`
để biết trong này có những triệu chứng nào, rồi **đóng lại**. Khi nào cluster hỏng thật thì mở
ra tìm mục có triệu chứng khớp.

Lý do phải đọc theo cách đó: nhiều mục ở đây là **di sản lịch sử** — lỗi RBAC khi join node
v1.18 vào cluster v1.17, Docker 1.13.1.84 trên CentOS 7, một regression của kubeadm 1.15 đã sửa
ở 1.20. Cluster lab chạy Kubernetes v1.35.6 với containerd 2.2.1 trên Ubuntu 24.04.4, nên phần
lớn danh sách này bạn **không thể** gặp. Học thuộc chúng là lãng phí; biết chúng nằm ở đâu mới
là kỹ năng.

**Phải hiểu ở lần đọc này:**

- **Cách tài liệu được tổ chức**: mỗi mục `##` là **một triệu chứng**, không phải một chủ đề.
  Thứ bạn cần nhớ là *hình dạng của triệu chứng* — một dòng log, một trạng thái Pod, một lệnh
  bị treo — đủ để tra ngược lại, chứ không phải cách sửa.
- Ít nhất một mục ở đây mô tả **trạng thái bình thường chứ không phải lỗi**: *`coredns` bị kẹt
  ở trạng thái `Pending`* nói thẳng rằng điều này **bình thường và nằm trong thiết kế**, vì
  kubeadm không phụ thuộc nhà cung cấp mạng nào và bạn phải cài Pod network trước.
- Ba mục có khả năng gặp thật trên cluster lab: *kubeadm bị treo khi chờ control plane*
  (ba nguyên nhân: mạng, cgroup driver lệch, container control plane crashloop), *Lỗi certificate
  TLS* (kubeconfig sai hoặc biến `KUBECONFIG` trỏ nhầm), và *`kubeadm bị treo khi xóa các
  container được quản lý`*.
- **Công cụ mà bài liên tục đẩy về là `crictl`** — dùng để nhìn vào container khi cluster chưa
  lên nên `kubectl` vô dụng. Đúng bộ đôi mà checkpoint giai đoạn 8 yêu cầu đọc được, cùng với
  `journalctl -u kubelet`. Lab 00 đã cài `cri-tools` và cấu hình `/etc/crictl.yaml` sẵn ở
  [A4.3](labs/LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl).
- **Đường thoát khi không mục nào khớp**, nêu ngay đầu bài: tìm issue đã có ở
  `github.com/kubernetes/kubeadm`, chưa có thì mở issue mới theo mẫu; còn nếu chỉ là chưa hiểu
  kubeadm hoạt động thế nào thì hỏi ở Slack `#kubeadm` hoặc StackOverflow với tag phù hợp.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Các mục gắn với phiên bản đã quá cũ: RBAC v1.18/v1.17, Docker 1.13.1.84 trên CentOS 7, `kubeadm reset` unmount `/var/lib/kubelet`, `context deadline exceeded` | cluster lab chạy v1.35.6, không thể gặp | không cần |
| *NIC mặc định khi dùng flannel trong Vagrant*, *IP không công khai được dùng cho container*, *`/usr` được mount ở chế độ chỉ đọc* | môi trường khác lab: Vagrant, cloud, Flatcar/CoreOS | không cần |
| *kube-proxy được lập lịch trước khi node được cloud-controller-manager khởi tạo* | chỉ xảy ra với cloud provider | không cần |
| *Xoay vòng client certificate của kubelet thất bại* | cần hiểu vòng đời và cách ký certificate | giai đoạn 18 certificate |
| *Nâng cấp thất bại do hash của etcd không thay đổi* | chỉ gặp khi nâng cấp từ v1.28.0–v1.28.2 | giai đoạn 17 nâng cấp |
| *Không thể dùng metrics-server một cách an toàn trong cluster kubeadm* | chưa cài metrics-server | giai đoạn 11 |

---

Như với bất kỳ chương trình nào, bạn có thể gặp lỗi khi cài đặt hoặc chạy kubeadm.
Trang này liệt kê một số tình huống lỗi thường gặp và cung cấp các bước có thể giúp bạn hiểu và khắc phục vấn đề.

Nếu vấn đề của bạn không được liệt kê bên dưới, vui lòng làm theo các bước sau:

- Nếu bạn cho rằng vấn đề của mình là một lỗi (bug) của kubeadm:
  - Truy cập [github.com/kubernetes/kubeadm](https://github.com/kubernetes/kubeadm/issues) và tìm kiếm các issue hiện có.
  - Nếu chưa có issue nào, vui lòng [mở một issue mới](https://github.com/kubernetes/kubeadm/issues/new) và làm theo mẫu issue (issue template).

- Nếu bạn chưa rõ về cách kubeadm hoạt động, bạn có thể hỏi trên [Slack](https://slack.k8s.io/) tại kênh `#kubeadm`,
  hoặc đặt câu hỏi trên [StackOverflow](https://stackoverflow.com/questions/tagged/kubernetes). Vui lòng thêm
  các tag liên quan như `#kubernetes` và `#kubeadm` để mọi người có thể giúp bạn.

## Không thể join một Node v1.18 vào cluster v1.17 do thiếu RBAC (Not possible to join a v1.18 Node to a v1.17 cluster due to missing RBAC)

Trong v1.18, kubeadm đã bổ sung cơ chế ngăn việc join một Node vào cluster nếu đã tồn tại một Node trùng tên.
Điều này yêu cầu thêm RBAC để người dùng bootstrap-token có thể GET một đối tượng Node.

Tuy nhiên, điều này gây ra một vấn đề: `kubeadm join` của v1.18 không thể join vào cluster được tạo bởi kubeadm v1.17.

Để khắc phục vấn đề này, bạn có hai lựa chọn:

Thực thi `kubeadm init phase bootstrap-token` trên một node control plane bằng kubeadm v1.18.
Lưu ý rằng lệnh này cũng kích hoạt toàn bộ các quyền còn lại của bootstrap-token.

hoặc

Áp dụng thủ công RBAC sau đây bằng `kubectl apply -f ...`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kubeadm:get-nodes
rules:
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubeadm:get-nodes
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubeadm:get-nodes
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: system:bootstrappers:kubeadm:default-node-token
```

## Không tìm thấy `ebtables` hoặc tệp thực thi tương tự trong quá trình cài đặt (`ebtables` or some similar executable not found during installation)

Nếu bạn thấy các cảnh báo sau khi chạy `kubeadm init`

```console
[preflight] WARNING: ebtables not found in system path
[preflight] WARNING: ethtool not found in system path
```

Thì có thể node của bạn đang thiếu `ebtables`, `ethtool` hoặc một tệp thực thi (executable) tương tự.
Bạn có thể cài đặt chúng bằng các lệnh sau:

- Với người dùng Ubuntu/Debian, chạy `apt install ebtables ethtool`.
- Với người dùng CentOS/Fedora, chạy `yum install ebtables ethtool`.

## kubeadm bị treo khi chờ control plane trong quá trình cài đặt (kubeadm blocks waiting for control plane during installation)

Nếu bạn nhận thấy `kubeadm init` bị treo sau khi in ra dòng sau:

```console
[apiclient] Created API client, waiting for the control plane to become ready
```

Điều này có thể do nhiều nguyên nhân. Phổ biến nhất là:

- vấn đề kết nối mạng. Hãy kiểm tra rằng máy của bạn có kết nối mạng đầy đủ trước khi tiếp tục.
- cgroup driver của container runtime khác với cgroup driver của kubelet. Để hiểu cách
  cấu hình đúng, xem [Cấu hình cgroup driver](218-configure-cgroup-driver-vi.md).
- các container của control plane bị crashloop hoặc bị treo. Bạn có thể kiểm tra điều này bằng cách chạy `docker ps`
  và xem xét từng container bằng cách chạy `docker logs`. Với các container runtime khác, xem
  [Gỡ lỗi node Kubernetes với crictl](307-crictl-vi.md).

## kubeadm bị treo khi xóa các container được quản lý (kubeadm blocks when removing managed containers)

Tình huống sau có thể xảy ra nếu container runtime dừng hoạt động và không xóa
bất kỳ container nào do Kubernetes quản lý:

```shell
sudo kubeadm reset
```

```console
[preflight] Running pre-flight checks
[reset] Stopping the kubelet service
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Removing kubernetes-managed containers
(block)
```

Một giải pháp khả thi là khởi động lại container runtime rồi chạy lại `kubeadm reset`.
Bạn cũng có thể dùng `crictl` để gỡ lỗi trạng thái của container runtime. Xem
[Gỡ lỗi node Kubernetes với crictl](307-crictl-vi.md).

## Pod ở trạng thái `RunContainerError`, `CrashLoopBackOff` hoặc `Error` (Pods in `RunContainerError`, `CrashLoopBackOff` or `Error` state)

Ngay sau khi chạy `kubeadm init`, không nên có pod nào ở các trạng thái này.

- Nếu có pod ở một trong các trạng thái này _ngay sau_ `kubeadm init`, vui lòng mở một
  issue trong repo của kubeadm. `coredns` (hoặc `kube-dns`) nên ở trạng thái `Pending`
  cho đến khi bạn triển khai add-on mạng (network add-on).
- Nếu bạn thấy các Pod ở trạng thái `RunContainerError`, `CrashLoopBackOff` hoặc `Error`
  sau khi đã triển khai add-on mạng mà `coredns` (hoặc `kube-dns`) vẫn không có gì thay đổi,
  rất có thể add-on Pod Network mà bạn đã cài đặt đang bị lỗi ở đâu đó.
  Bạn có thể phải cấp thêm quyền RBAC cho nó hoặc dùng phiên bản mới hơn. Vui lòng tạo
  một issue trong trình theo dõi issue của nhà cung cấp Pod Network để vấn đề được phân loại và xử lý tại đó.

## `coredns` bị kẹt ở trạng thái `Pending` (`coredns` is stuck in the `Pending` state)

Điều này là **bình thường** và nằm trong thiết kế. kubeadm không phụ thuộc vào nhà cung cấp mạng nào (network provider-agnostic),
vì vậy người quản trị nên [cài đặt add-on pod network](165-addons-vi.md)
mà mình chọn. Bạn phải cài đặt một Pod Network
trước khi CoreDNS có thể được triển khai đầy đủ. Do đó `coredns` ở trạng thái `Pending` trước khi mạng được thiết lập.

## Các service `HostPort` không hoạt động (`HostPort` services do not work)

Chức năng `HostPort` và `HostIP` có khả dụng hay không tùy thuộc vào nhà cung cấp Pod Network
của bạn. Vui lòng liên hệ tác giả của add-on Pod Network để biết
chức năng `HostPort` và `HostIP` có được hỗ trợ hay không.

Các nhà cung cấp CNI Calico, Canal và Flannel đã được xác nhận là hỗ trợ HostPort.

Để biết thêm thông tin, xem
[tài liệu CNI portmap](https://github.com/containernetworking/plugins/blob/master/plugins/meta/portmap/README.md).

Nếu nhà cung cấp mạng của bạn không hỗ trợ plugin CNI portmap, bạn có thể cần dùng
[tính năng NodePort của service](82-service-vi.md#type-nodeport)
hoặc dùng `HostNetwork=true`.

## Không thể truy cập Pod qua Service IP của chúng (Pods are not accessible via their Service IP)

- Nhiều add-on mạng chưa bật [chế độ hairpin (hairpin mode)](301-debug-service-vi.md#a-pod-fails-to-reach-itself-via-the-service-ip),
  chế độ cho phép các pod tự truy cập chính mình thông qua Service IP của chúng. Đây là một vấn đề liên quan đến
  [CNI](https://github.com/containernetworking/cni/issues/476). Vui lòng liên hệ nhà cung cấp
  add-on mạng để biết tình trạng hỗ trợ mới nhất của họ đối với chế độ hairpin.

- Nếu bạn đang dùng VirtualBox (trực tiếp hoặc thông qua Vagrant), bạn cần
  đảm bảo rằng `hostname -i` trả về một địa chỉ IP định tuyến được (routable). Mặc định, interface
  đầu tiên được kết nối vào một mạng host-only không định tuyến được. Một cách khắc phục
  là sửa `/etc/hosts`, xem
  [Vagrantfile](https://github.com/errordeveloper/k8s-playground/blob/22dd39dfc06111235620e6c4404a96ae146f26fd/Vagrantfile#L11)
  này để tham khảo ví dụ.

## Lỗi certificate TLS (TLS certificate errors)

Lỗi sau đây cho thấy khả năng certificate không khớp (certificate mismatch).

```none
# kubectl get pods
Unable to connect to the server: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")
```

- Kiểm tra rằng tệp `$HOME/.kube/config` chứa một certificate hợp lệ, và
  tạo lại certificate nếu cần. Các certificate trong tệp kubeconfig
  được mã hóa base64. Có thể dùng lệnh `base64 --decode` để giải mã certificate
  và dùng `openssl x509 -text -noout` để xem thông tin certificate.

- Bỏ thiết lập (unset) biến môi trường `KUBECONFIG` bằng:

  ```sh
  unset KUBECONFIG
  ```

  Hoặc đặt nó về vị trí `KUBECONFIG` mặc định:

  ```sh
  export KUBECONFIG=/etc/kubernetes/admin.conf
  ```

- Một cách khắc phục khác là ghi đè `kubeconfig` hiện có của người dùng "admin":

  ```sh
  mv $HOME/.kube $HOME/.kube.bak
  mkdir $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```

## Xoay vòng client certificate của kubelet thất bại (Kubelet client certificate rotation fails) {#kubelet-client-cert}

Mặc định, kubeadm cấu hình kubelet với cơ chế tự động xoay vòng (rotation) client certificate bằng cách dùng
liên kết mềm (symlink) `/var/lib/kubelet/pki/kubelet-client-current.pem` được chỉ định trong `/etc/kubernetes/kubelet.conf`.
Nếu quá trình xoay vòng này thất bại, bạn có thể thấy các lỗi như `x509: certificate has expired or is not yet valid`
trong log của kube-apiserver. Để khắc phục vấn đề, bạn phải làm theo các bước sau:

1. Sao lưu và xóa `/etc/kubernetes/kubelet.conf` cùng `/var/lib/kubelet/pki/kubelet-client*` trên node bị lỗi.
1. Từ một node control plane đang hoạt động trong cluster có `/etc/kubernetes/pki/ca.key`, thực thi
   `kubeadm kubeconfig user --org system:nodes --client-name system:node:$NODE > kubelet.conf`.
   `$NODE` phải được đặt thành tên của node đang bị lỗi trong cluster.
   Sửa thủ công tệp `kubelet.conf` thu được để điều chỉnh tên cluster và endpoint của server,
   hoặc truyền `kubeconfig user --config` (xem [Tạo tệp kubeconfig cho người dùng bổ sung](219-kubeadm-certs-vi.md#kubeconfig-additional-users)). Nếu cluster của bạn không có
   `ca.key`, bạn phải ký các certificate nhúng trong `kubelet.conf` từ bên ngoài.
1. Sao chép tệp `kubelet.conf` thu được vào `/etc/kubernetes/kubelet.conf` trên node bị lỗi.
1. Khởi động lại kubelet (`systemctl restart kubelet`) trên node bị lỗi và chờ
   `/var/lib/kubelet/pki/kubelet-client-current.pem` được tạo lại.
1. Sửa thủ công `kubelet.conf` để trỏ đến các client certificate đã được xoay vòng của kubelet, bằng cách thay thế
   `client-certificate-data` và `client-key-data` bằng:

   ```yaml
   client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
   client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
   ```

1. Khởi động lại kubelet.
1. Đảm bảo node chuyển sang trạng thái `Ready`.

## NIC mặc định khi dùng flannel làm pod network trong Vagrant (Default NIC when using flannel as the pod network in Vagrant)

Lỗi sau đây có thể cho thấy có gì đó không ổn trong pod network:

```sh
Error from server (NotFound): the server could not find the requested resource
```

- Nếu bạn đang dùng flannel làm pod network bên trong Vagrant, thì bạn sẽ phải
  chỉ định tên interface mặc định cho flannel.

  Vagrant thường gán hai interface cho tất cả các VM. Interface đầu tiên, mà mọi host
  đều được gán địa chỉ IP `10.0.2.15`, dành cho lưu lượng ra ngoài được NAT.

  Điều này có thể dẫn đến vấn đề với flannel, vì flannel mặc định chọn interface đầu tiên trên host.
  Hệ quả là tất cả các host đều nghĩ rằng chúng có cùng một địa chỉ IP công khai. Để tránh điều này,
  truyền cờ `--iface eth1` cho flannel để interface thứ hai được chọn.

## IP không công khai được dùng cho container (Non-public IP used for containers)

Trong một số tình huống, các lệnh `kubectl logs` và `kubectl run` có thể trả về
các lỗi sau trong một cluster vẫn hoạt động bình thường:

```console
Error from server: Get https://10.19.0.41:10250/containerLogs/default/mysql-ddc65b868-glc5m/mysql: dial tcp 10.19.0.41:10250: getsockopt: no route to host
```

- Điều này có thể do Kubernetes đang dùng một IP không thể giao tiếp với các IP khác trên
  cùng một subnet (bề ngoài là vậy), có thể do chính sách của nhà cung cấp máy chủ.
- DigitalOcean gán một IP công khai cho `eth0` và đồng thời một IP riêng (private) được dùng nội bộ
  làm IP neo (anchor) cho tính năng floating IP của họ, nhưng `kubelet` lại chọn IP riêng đó làm
  `InternalIP` của node thay vì IP công khai.

  Dùng `ip addr show` để kiểm tra tình huống này thay vì `ifconfig`, vì `ifconfig` sẽ
  không hiển thị địa chỉ IP bí danh (alias) gây ra vấn đề. Ngoài ra, một API endpoint dành riêng cho
  DigitalOcean cho phép truy vấn IP neo từ droplet:

  ```sh
  curl http://169.254.169.254/metadata/v1/interfaces/public/0/anchor_ipv4/address
  ```

  Cách khắc phục là chỉ cho `kubelet` biết IP nào cần dùng bằng `--node-ip`.
  Khi dùng DigitalOcean, đó có thể là IP công khai (gán cho `eth0`) hoặc
  IP riêng (gán cho `eth1`) nếu bạn muốn dùng mạng riêng tùy chọn.
  Có thể dùng mục `kubeletExtraArgs` trong
  [cấu trúc `NodeRegistrationOptions`](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/#kubeadm-k8s-io-v1beta4-NodeRegistrationOptions)
  của kubeadm cho việc này.

  Sau đó khởi động lại `kubelet`:

  ```sh
  systemctl daemon-reload
  systemctl restart kubelet
  ```

## Các pod `coredns` ở trạng thái `CrashLoopBackOff` hoặc `Error` (`coredns` pods have `CrashLoopBackOff` or `Error` state)

Nếu bạn có các node đang chạy SELinux với phiên bản Docker cũ, bạn có thể gặp tình huống
các pod `coredns` không khởi động được. Để giải quyết, bạn có thể thử một trong các phương án sau:

- Nâng cấp lên [phiên bản Docker mới hơn](00-container-runtimes-vi.md#docker).

- [Tắt SELinux](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/security-enhanced_linux/sect-security-enhanced_linux-enabling_and_disabling_selinux-disabling_selinux).

- Sửa deployment `coredns` để đặt `allowPrivilegeEscalation` thành `true`:

```bash
kubectl -n kube-system get deployment coredns -o yaml | \
  sed 's/allowPrivilegeEscalation: false/allowPrivilegeEscalation: true/g' | \
  kubectl apply -f -
```

Một nguyên nhân khác khiến CoreDNS bị `CrashLoopBackOff` là khi một Pod CoreDNS được triển khai trong Kubernetes phát hiện ra vòng lặp (loop).
[Một số cách khắc phục](https://github.com/coredns/coredns/tree/master/plugin/loop#troubleshooting-loops-in-kubernetes-clusters)
đã có sẵn để tránh việc Kubernetes cứ cố khởi động lại Pod CoreDNS mỗi khi CoreDNS phát hiện vòng lặp và thoát.

> **Cảnh báo:**
>
> Tắt SELinux hoặc đặt `allowPrivilegeEscalation` thành `true` có thể làm giảm
> tính bảo mật của cluster của bạn.

## Các pod etcd khởi động lại liên tục (etcd pods restart continually)

Nếu bạn gặp lỗi sau:

```
rpc error: code = 2 desc = oci runtime error: exec failed: container_linux.go:247: starting container process caused "process_linux.go:110: decoding init error from pipe caused \"read parent: connection reset by peer\""
```

Vấn đề này xuất hiện nếu bạn chạy CentOS 7 với Docker 1.13.1.84.
Phiên bản Docker này có thể ngăn kubelet thực thi lệnh (exec) vào trong container etcd.

Để khắc phục vấn đề, chọn một trong các phương án sau:

- Quay về (roll back) phiên bản Docker cũ hơn, chẳng hạn 1.13.1-75

  ```
  yum downgrade docker-1.13.1-75.git8633870.el7.centos.x86_64 docker-client-1.13.1-75.git8633870.el7.centos.x86_64 docker-common-1.13.1-75.git8633870.el7.centos.x86_64
  ```

- Cài đặt một trong các phiên bản mới hơn được khuyến nghị, chẳng hạn 18.06:

  ```bash
  sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  yum install docker-ce-18.06.1.ce-3.el7.x86_64
  ```

## Không thể truyền danh sách giá trị phân tách bằng dấu phẩy cho các đối số bên trong cờ `--component-extra-args`

Các cờ của `kubeadm init` như `--component-extra-args` cho phép bạn truyền đối số tùy chỉnh cho một thành phần
control plane như kube-apiserver. Tuy nhiên, cơ chế này bị hạn chế bởi kiểu dữ liệu bên dưới được dùng để phân tích cú pháp
các giá trị (`mapStringString`).

Nếu bạn quyết định truyền một đối số hỗ trợ nhiều giá trị phân tách bằng dấu phẩy, chẳng hạn
`--apiserver-extra-args "enable-admission-plugins=LimitRanger,NamespaceExists"`, cờ này sẽ thất bại với lỗi
`flag: malformed pair, expect string=string`. Điều này xảy ra vì danh sách đối số cho
`--apiserver-extra-args` mong đợi các cặp `key=value`, và trong trường hợp này `NamespacesExists` bị coi
là một key thiếu value.

Cách khác, bạn có thể thử tách các cặp `key=value` như sau:
`--apiserver-extra-args "enable-admission-plugins=LimitRanger,enable-admission-plugins=NamespaceExists"`
nhưng kết quả là key `enable-admission-plugins` chỉ nhận giá trị `NamespaceExists`.

Một cách khắc phục đã biết là dùng [tệp cấu hình](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/) của kubeadm.

## kube-proxy được lập lịch trước khi node được cloud-controller-manager khởi tạo (kube-proxy scheduled before node is initialized by cloud-controller-manager)

Trong các kịch bản dùng nhà cung cấp đám mây (cloud provider), kube-proxy có thể bị lập lịch (schedule) lên các worker node mới trước khi
cloud-controller-manager khởi tạo xong địa chỉ của node. Điều này khiến kube-proxy không
lấy được đúng địa chỉ IP của node và gây ra hiệu ứng dây chuyền đến chức năng proxy quản lý
các bộ cân bằng tải (load balancer).

Có thể thấy lỗi sau trong các Pod kube-proxy:

```
server.go:610] Failed to retrieve node IP: host IP unknown; known addresses: []
proxier.go:340] invalid nodeIP, initializing kube-proxy with 127.0.0.1 as nodeIP
```

Một giải pháp đã biết là vá (patch) DaemonSet kube-proxy để cho phép lập lịch nó trên các node
control plane bất kể điều kiện (condition) của chúng, đồng thời giữ nó không chạy trên các node khác cho đến khi các điều kiện
bảo vệ ban đầu của những node đó được gỡ bỏ:

```
kubectl -n kube-system patch ds kube-proxy -p='{
  "spec": {
    "template": {
      "spec": {
        "tolerations": [
          {
            "key": "CriticalAddonsOnly",
            "operator": "Exists"
          },
          {
            "effect": "NoSchedule",
            "key": "node-role.kubernetes.io/control-plane"
          }
        ]
      }
    }
  }
}'
```

Issue theo dõi vấn đề này ở [đây](https://github.com/kubernetes/kubeadm/issues/1027).

## `/usr` được mount ở chế độ chỉ đọc trên các node (`/usr` is mounted read-only on nodes) {#usr-mounted-read-only}

Trên các bản phân phối Linux như Fedora CoreOS hoặc Flatcar Container Linux, thư mục `/usr` được mount dưới dạng hệ thống tệp chỉ đọc (read-only).
Để [hỗ trợ flex-volume](https://github.com/kubernetes/community/blob/ab55d85/contributors/devel/sig-storage/flexvolume.md),
các thành phần Kubernetes như kubelet và kube-controller-manager dùng đường dẫn mặc định
`/usr/libexec/kubernetes/kubelet-plugins/volume/exec/`, nhưng thư mục flex-volume _phải ghi được_
thì tính năng này mới hoạt động.

> **Ghi chú:**
>
> FlexVolume đã bị loại bỏ dần (deprecated) từ bản phát hành Kubernetes v1.23.

Để khắc phục vấn đề này, bạn có thể cấu hình thư mục flex-volume bằng
[tệp cấu hình](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/) của kubeadm.

Trên Node control plane chính (được tạo bằng `kubeadm init`), truyền tệp sau
bằng `--config`:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
  - name: "volume-plugin-dir"
    value: "/opt/libexec/kubernetes/kubelet-plugins/volume/exec/"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
controllerManager:
  extraArgs:
  - name: "flex-volume-plugin-dir"
    value: "/opt/libexec/kubernetes/kubelet-plugins/volume/exec/"
```

Trên các Node đang join:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
nodeRegistration:
  kubeletExtraArgs:
  - name: "volume-plugin-dir"
    value: "/opt/libexec/kubernetes/kubelet-plugins/volume/exec/"
```

Cách khác, bạn có thể sửa `/etc/fstab` để mount `/usr` ở chế độ ghi được, nhưng xin lưu ý
rằng làm vậy là đang thay đổi một nguyên tắc thiết kế của bản phân phối Linux đó.

## `kubeadm upgrade plan` in ra thông báo lỗi `context deadline exceeded`

Thông báo lỗi này xuất hiện khi nâng cấp cluster Kubernetes bằng `kubeadm` trong
trường hợp chạy etcd bên ngoài (external etcd). Đây không phải lỗi nghiêm trọng và xảy ra vì
các phiên bản kubeadm cũ thực hiện kiểm tra phiên bản đối với cluster etcd bên ngoài.
Bạn có thể tiếp tục với `kubeadm upgrade apply ...`.

Vấn đề này đã được sửa từ phiên bản 1.19.

## `kubeadm reset` unmount `/var/lib/kubelet` (`kubeadm reset` unmounts `/var/lib/kubelet`)

Nếu `/var/lib/kubelet` đang được mount, việc thực hiện `kubeadm reset` sẽ unmount nó.

Để khắc phục vấn đề, hãy mount lại thư mục `/var/lib/kubelet` sau khi thực hiện thao tác `kubeadm reset`.

Đây là lỗi hồi quy (regression) xuất hiện trong kubeadm 1.15. Vấn đề đã được sửa trong 1.20.

## Không thể dùng metrics-server một cách an toàn trong cluster kubeadm (Cannot use the metrics-server securely in a kubeadm cluster)

Trong một cluster kubeadm, [metrics-server](https://github.com/kubernetes-sigs/metrics-server)
có thể được dùng theo cách không an toàn bằng cách truyền cờ `--kubelet-insecure-tls` cho nó. Cách này không được khuyến nghị cho các cluster production.

Nếu bạn muốn dùng TLS giữa metrics-server và kubelet thì có một vấn đề:
kubeadm triển khai một serving certificate tự ký (self-signed) cho kubelet. Điều này có thể gây ra các lỗi sau
ở phía metrics-server:

```
x509: certificate signed by unknown authority
x509: certificate is valid for IP-foo not IP-bar
```

Xem [Bật serving certificate được ký cho kubelet](219-kubeadm-certs-vi.md#kubelet-serving-certs)
để hiểu cách cấu hình các kubelet trong cluster kubeadm sao cho có serving certificate được ký đúng cách.

Xem thêm [Cách chạy metrics-server một cách an toàn](https://github.com/kubernetes-sigs/metrics-server/blob/master/FAQ.md#how-to-run-metrics-server-securely).

## Nâng cấp thất bại do hash của etcd không thay đổi (Upgrade fails due to etcd hash not changing)

Chỉ áp dụng khi nâng cấp một node control plane bằng binary kubeadm v1.28.3 trở lên,
trong đó node hiện đang được quản lý bởi kubeadm phiên bản v1.28.0, v1.28.1 hoặc v1.28.2.

Đây là thông báo lỗi bạn có thể gặp:

```
[upgrade/etcd] Failed to upgrade etcd: couldn't upgrade control plane. kubeadm has tried to recover everything into the earlier state. Errors faced: static Pod hash for component etcd on Node kinder-upgrade-control-plane-1 did not change after 5m0s: timed out waiting for the condition
[upgrade/etcd] Waiting for previous etcd to become available
I0907 10:10:09.109104    3704 etcd.go:588] [etcd] attempting to see if all cluster endpoints ([https://172.17.0.6:2379/ https://172.17.0.4:2379/ https://172.17.0.3:2379/]) are available 1/10
[upgrade/etcd] Etcd was rolled back and is now available
static Pod hash for component etcd on Node kinder-upgrade-control-plane-1 did not change after 5m0s: timed out waiting for the condition
couldn't upgrade control plane. kubeadm has tried to recover everything into the earlier state. Errors faced
k8s.io/kubernetes/cmd/kubeadm/app/phases/upgrade.rollbackOldManifests
	cmd/kubeadm/app/phases/upgrade/staticpods.go:525
k8s.io/kubernetes/cmd/kubeadm/app/phases/upgrade.upgradeComponent
	cmd/kubeadm/app/phases/upgrade/staticpods.go:254
k8s.io/kubernetes/cmd/kubeadm/app/phases/upgrade.performEtcdStaticPodUpgrade
	cmd/kubeadm/app/phases/upgrade/staticpods.go:338
...
```

Nguyên nhân của thất bại này là các phiên bản bị ảnh hưởng sinh ra tệp manifest etcd với
các giá trị mặc định không mong muốn trong PodSpec. Điều này dẫn đến sự khác biệt (diff) khi so sánh manifest,
và kubeadm sẽ mong đợi hash của Pod thay đổi, nhưng kubelet sẽ không bao giờ cập nhật hash đó.

Có hai cách khắc phục nếu bạn gặp vấn đề này trong cluster của mình:

- Có thể bỏ qua bước nâng cấp etcd giữa các phiên bản bị ảnh hưởng và v1.28.3 (trở lên) bằng cách dùng:

  ```shell
  kubeadm upgrade {apply|node} [version] --etcd-upgrade=false
  ```

  Cách này không được khuyến nghị trong trường hợp một phiên bản etcd mới được đưa vào bởi một bản vá v1.28 sau này.

- Trước khi nâng cấp, vá manifest của static pod etcd để loại bỏ các thuộc tính mặc định gây ra vấn đề:

  ```patch
  diff --git a/etc/kubernetes/manifests/etcd_defaults.yaml b/etc/kubernetes/manifests/etcd_origin.yaml
  index d807ccbe0aa..46b35f00e15 100644
  --- a/etc/kubernetes/manifests/etcd_defaults.yaml
  +++ b/etc/kubernetes/manifests/etcd_origin.yaml
  @@ -43,7 +43,6 @@ spec:
          scheme: HTTP
        initialDelaySeconds: 10
        periodSeconds: 10
  -      successThreshold: 1
        timeoutSeconds: 15
      name: etcd
      resources:
  @@ -59,26 +58,18 @@ spec:
          scheme: HTTP
        initialDelaySeconds: 10
        periodSeconds: 10
  -      successThreshold: 1
        timeoutSeconds: 15
  -    terminationMessagePath: /dev/termination-log
  -    terminationMessagePolicy: File
      volumeMounts:
      - mountPath: /var/lib/etcd
        name: etcd-data
      - mountPath: /etc/kubernetes/pki/etcd
        name: etcd-certs
  -  dnsPolicy: ClusterFirst
  -  enableServiceLinks: true
    hostNetwork: true
    priority: 2000001000
    priorityClassName: system-node-critical
  -  restartPolicy: Always
  -  schedulerName: default-scheduler
    securityContext:
      seccompProfile:
        type: RuntimeDefault
  -  terminationGracePeriodSeconds: 30
    volumes:
    - hostPath:
        path: /etc/kubernetes/pki/etcd
  ```

Có thể tìm thêm thông tin trong
[issue theo dõi](https://github.com/kubernetes/kubeadm/issues/2927) lỗi này.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Đây là tài liệu tra cứu, nên ba câu dưới đây chỉ kiểm tra **cách dùng nó**, không kiểm tra bạn
có thuộc từng lỗi hay không. Trả lời được mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 8:

1. Bạn vừa chạy `kubeadm init` trên `lab-k8s-master` xong và chưa cài Flannel. `kubectl get pods -A`
   cho thấy Pod `coredns` ở trạng thái `Pending`. Có nên mở bài này tìm cách sửa không?
2. `kubeadm init` treo ở dòng `[apiclient] Created API client, waiting for the control plane to
   become ready`. Bài liệt kê mấy nhóm nguyên nhân, và bạn dùng công cụ nào để nhìn vào
   container của control plane khi `kubectl` chưa dùng được?
3. Bạn gặp một triệu chứng mà không mục `##` nào trong bài mô tả. Bài chỉ bạn làm gì tiếp, theo
   đúng thứ tự?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không — đây không phải lỗi.** Bài có hẳn một mục *`coredns` bị kẹt ở trạng thái `Pending`*
   và câu đầu tiên của nó là: điều này **bình thường và nằm trong thiết kế**, vì kubeadm không
   phụ thuộc nhà cung cấp mạng nào nên bạn **phải cài đặt một Pod Network trước khi CoreDNS có
   thể được triển khai đầy đủ**. Đây đúng là kiểu bẫy mà một tài liệu troubleshooting hay tạo
   ra: thấy tên mục khớp với triệu chứng nên tưởng mình đang gặp lỗi, trong khi nội dung mục lại
   nói ngược lại. Mục *Pod ở trạng thái `RunContainerError`, `CrashLoopBackOff` hoặc `Error`*
   nhắc lại cùng điều đó: `coredns` **nên** ở `Pending` cho tới khi bạn triển khai add-on mạng.
2. **Ba nhóm nguyên nhân**, theo đúng thứ tự phổ biến mà bài xếp: **vấn đề kết nối mạng**;
   **cgroup driver của container runtime khác cgroup driver của kubelet**; và **các container
   control plane bị crashloop hoặc bị treo**. Công cụ để nhìn vào container là **`crictl`** —
   bài trỏ sang *Gỡ lỗi node Kubernetes với crictl* cho mọi container runtime không phải Docker,
   và đây cũng là công cụ nó nhắc lại ở mục `kubeadm reset` bị treo. Lúc này `kubectl` chưa dùng
   được vì API server chính là thứ chưa lên.
3. Theo đúng thứ tự bài nêu ở phần mở đầu: **nếu bạn cho rằng đó là bug của kubeadm** thì trước
   hết vào `github.com/kubernetes/kubeadm` và **tìm trong các issue hiện có**; **chỉ khi chưa có
   issue nào** mới mở issue mới và **làm theo mẫu issue**. Còn **nếu bạn chỉ chưa rõ kubeadm hoạt
   động thế nào** — không phải bug — thì hỏi ở kênh Slack `#kubeadm` hoặc StackOverflow, kèm tag
   `#kubernetes` và `#kubeadm`. Phân biệt "bug" với "chưa hiểu" là bước đầu tiên, không phải bước
   cuối.

</details>

Đây là bài cuối của **Giai đoạn 8**. Trả lời được ba câu này nghĩa là bạn biết dùng tài liệu tra
cứu; phần dựng thật nằm ở Lab 8a, 8b và 8c — xem [bản đồ lab](labs/README.md#4-bản-đồ-lab).

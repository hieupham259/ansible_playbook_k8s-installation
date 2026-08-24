# Thiết lập cluster etcd có tính sẵn sàng cao với kubeadm (Set up a High Availability etcd Cluster with kubeadm)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/>
>
> (Trang gốc không có phần description trong frontmatter)

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm), bài 6/9 ·
Kiểm chứng ở Lab 8c (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này **có điều kiện**: lộ trình ghi rõ nó *chỉ cần nếu chọn external etcd* ở bài
[06](06-ha-topology-vi.md), và **phải dựng xong trước** bài [08](08-high-availability-vi.md).
Nếu bạn chọn stacked thì đọc lướt để biết external etcd tốn công tới mức nào, rồi đi tiếp.

Bài viết dưới dạng runbook tám bước, nhưng thứ đáng nhớ không phải danh sách lệnh — mà là
**mô hình**: ở đây kubelet bị trưng dụng làm trình quản lý dịch vụ cho etcd, khi **chưa hề có
cluster Kubernetes nào tồn tại**. Đọc theo hướng đó thì các bước lạ (ghi đè unit kubelet, tạo
certificate trên một host rồi phát đi) đều có lý do.

**Phải hiểu ở lần đọc này:**

- Kết quả của bài là một **cluster etcd external ba thành viên**, chạy dưới dạng **static pod
  do một kubelet quản lý** — chưa có API server, chưa có Kubernetes cluster. Ba host đó cần
  container runtime, kubelet và kubeadm đã cài, cùng quyền pull image từ `registry.k8s.io`.
- Vì sao phải ghi đè unit của kubelet (bước 1): **etcd được tạo trước**, nên phải tạo unit file
  có **độ ưu tiên cao hơn** unit kubelet mà kubeadm cung cấp — `20-etcd-service-manager.conf`
  trỏ kubelet vào một `KubeletConfiguration` riêng với `staticPodPath: /etc/kubernetes/manifests`,
  `authorization.mode: AlwaysAllow` và `address: 127.0.0.1`.
- Mô hình phân phối certificate (bước 3–5): tạo **toàn bộ certificate trên một host** (`$HOST0`)
  rồi chỉ chép **những file cần thiết** sang host khác. `ca.key` **không được rời `$HOST0`** —
  đó là mục đích của hai lệnh `find /tmp/${HOST1} -name ca.key -type f -delete`. kubeadm tự
  chứa đủ cơ chế mã hóa, không cần công cụ ngoài.
- Mỗi member có **file `kubeadmcfg.yaml` riêng**: `etcd.local` với `serverCertSANs`/`peerCertSANs`
  là IP của chính host đó, còn `extraArgs` chứa `initial-cluster` liệt kê **cả ba** member,
  `initial-cluster-state: new`, và các URL peer/client.
- Hai port có hai vai trò: **2380 cho lưu lượng giữa các member** (`listen-peer-urls`,
  `initial-advertise-peer-urls`) và **2379 cho client** (`listen-client-urls`,
  `advertise-client-urls`). `kubeadm init phase etcd local` sinh static pod manifest trên từng
  host; kiểm tra bằng `etcdctl ... endpoint health`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ba cây thư mục liệt kê file bắt buộc trên `$HOST0`/`$HOST1`/`$HOST2` | là checklist lúc làm thật, không phải kiến thức | Lab 8c |
| Vai trò riêng của từng certificate: `peer`, `server`, `healthcheck-client`, `apiserver-etcd-client` | chưa học PKI của cluster | CP3 certificate |
| Cách chạy `etcdctl` bên trong container bằng `crictl run` | là bước kiểm tra tùy chọn | CP4 etcd backup |
| Ghi chú "etcd không hỗ trợ dual-stack" | dual-stack là bài riêng, đọc sau | bài [05](05-dual-stack-support-vi.md) |
| `authentication.anonymous/webhook`, `authorization.mode` trong `KubeletConfiguration` | chưa học xác thực và ủy quyền | giai đoạn 9 |

---

Theo mặc định, kubeadm chạy một instance etcd cục bộ (local) trên mỗi node control plane.
Cũng có thể coi cluster etcd là bên ngoài (external) và cung cấp (provision)
các instance etcd trên các host riêng biệt. Sự khác biệt giữa hai cách tiếp cận này được trình bày trong
trang [Các lựa chọn cho topology có tính sẵn sàng cao](06-ha-topology-vi.md).

Tác vụ này hướng dẫn quy trình tạo một cluster etcd external có tính sẵn sàng cao
gồm ba thành viên, có thể được kubeadm sử dụng trong quá trình tạo cluster.

## Trước khi bạn bắt đầu (Before you begin)

- Ba host có thể giao tiếp với nhau qua các cổng TCP 2379 và 2380. Tài liệu
  này giả định các cổng mặc định này. Tuy nhiên, chúng có thể được cấu hình thông qua
  file cấu hình kubeadm.
- Mỗi host phải có systemd và một shell tương thích bash được cài đặt.
- Mỗi host phải [có một container runtime, kubelet và kubeadm được cài đặt](01-install-kubeadm-vi.md).
- Mỗi host cần có quyền truy cập đến container image registry của Kubernetes (`registry.k8s.io`) hoặc liệt kê/pull image etcd cần thiết bằng
  `kubeadm config images list/pull`. Hướng dẫn này sẽ thiết lập các instance etcd dưới dạng
  [static pod](293-static-pod-tasks-vi.md) được quản lý bởi một kubelet.
- Một hạ tầng nào đó để sao chép file giữa các host. Ví dụ, `ssh` và `scp`
  có thể đáp ứng yêu cầu này.

## Thiết lập cluster (Setting up the cluster)

Cách tiếp cận chung là tạo tất cả các certificate trên một node và chỉ phân phối
các file _cần thiết_ đến các node khác.

> **Ghi chú:** kubeadm chứa tất cả các cơ chế mã hóa (cryptographic machinery) cần thiết để tạo
> các certificate được mô tả bên dưới; không cần công cụ mã hóa nào khác cho
> ví dụ này.

> **Ghi chú:** Các ví dụ bên dưới dùng địa chỉ IPv4 nhưng bạn cũng có thể cấu hình kubeadm, kubelet và etcd
> để dùng địa chỉ IPv6. Dual-stack được hỗ trợ bởi một số tùy chọn của Kubernetes, nhưng etcd thì không. Để biết thêm chi tiết
> về hỗ trợ dual-stack của Kubernetes, xem [Hỗ trợ dual-stack với kubeadm](05-dual-stack-support-vi.md).

1. Cấu hình kubelet để trở thành trình quản lý dịch vụ (service manager) cho etcd.

   > **Ghi chú:** Bạn phải làm điều này trên mọi host mà etcd sẽ chạy trên đó.

   Vì etcd được tạo trước, bạn phải ghi đè mức ưu tiên của service bằng cách tạo một unit file mới
   có độ ưu tiên cao hơn unit file kubelet do kubeadm cung cấp.

   ```sh
   cat << EOF > /etc/systemd/system/kubelet.service.d/kubelet.conf
   # Thay "systemd" bằng cgroup driver của container runtime mà bạn dùng. Giá trị mặc định trong kubelet là "cgroupfs".
   # Thay giá trị của "containerRuntimeEndpoint" nếu dùng một container runtime khác (nếu cần).
   #
   apiVersion: kubelet.config.k8s.io/v1beta1
   kind: KubeletConfiguration
   authentication:
     anonymous:
       enabled: false
     webhook:
       enabled: false
   authorization:
     mode: AlwaysAllow
   cgroupDriver: systemd
   address: 127.0.0.1
   containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock
   staticPodPath: /etc/kubernetes/manifests
   EOF

   cat << EOF > /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
   [Service]
   ExecStart=
   ExecStart=/usr/bin/kubelet --config=/etc/systemd/system/kubelet.service.d/kubelet.conf
   Restart=always
   EOF

   systemctl daemon-reload
   systemctl restart kubelet
   ```

   Kiểm tra trạng thái của kubelet để đảm bảo nó đang chạy.

   ```sh
   systemctl status kubelet
   ```

1. Tạo các file cấu hình cho kubeadm.

   Tạo một file cấu hình kubeadm cho mỗi host sẽ có một thành viên etcd
   chạy trên đó bằng script sau.

   ```sh
   # Cập nhật HOST0, HOST1 và HOST2 bằng địa chỉ IP của các host của bạn
   export HOST0=10.0.0.6
   export HOST1=10.0.0.7
   export HOST2=10.0.0.8

   # Cập nhật NAME0, NAME1 và NAME2 bằng hostname của các host của bạn
   export NAME0="infra0"
   export NAME1="infra1"
   export NAME2="infra2"

   # Tạo các thư mục tạm để lưu các file sẽ được chuyển đến các host khác
   mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

   HOSTS=(${HOST0} ${HOST1} ${HOST2})
   NAMES=(${NAME0} ${NAME1} ${NAME2})

   for i in "${!HOSTS[@]}"; do
   HOST=${HOSTS[$i]}
   NAME=${NAMES[$i]}
   cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
   ---
   apiVersion: "kubeadm.k8s.io/v1beta4"
   kind: InitConfiguration
   nodeRegistration:
       name: ${NAME}
   localAPIEndpoint:
       advertiseAddress: ${HOST}
   ---
   apiVersion: "kubeadm.k8s.io/v1beta4"
   kind: ClusterConfiguration
   etcd:
       local:
           serverCertSANs:
           - "${HOST}"
           peerCertSANs:
           - "${HOST}"
           extraArgs:
           - name: initial-cluster
             value: ${NAMES[0]}=https://${HOSTS[0]}:2380,${NAMES[1]}=https://${HOSTS[1]}:2380,${NAMES[2]}=https://${HOSTS[2]}:2380
           - name: initial-cluster-state
             value: new
           - name: name
             value: ${NAME}
           - name: listen-peer-urls
             value: https://${HOST}:2380
           - name: listen-client-urls
             value: https://${HOST}:2379
           - name: advertise-client-urls
             value: https://${HOST}:2379
           - name: initial-advertise-peer-urls
             value: https://${HOST}:2380
   EOF
   done
   ```

1. Tạo certificate authority (CA).

   Nếu bạn đã có sẵn một CA thì việc duy nhất cần làm là sao chép file `crt` và
   `key` của CA vào `/etc/kubernetes/pki/etcd/ca.crt` và
   `/etc/kubernetes/pki/etcd/ca.key`. Sau khi các file đó đã được sao chép,
   hãy chuyển sang bước tiếp theo, "Tạo certificate cho từng thành viên".

   Nếu bạn chưa có CA, hãy chạy lệnh này trên `$HOST0` (nơi bạn
   đã tạo các file cấu hình cho kubeadm).

   ```
   kubeadm init phase certs etcd-ca
   ```

   Lệnh này tạo ra hai file:

   - `/etc/kubernetes/pki/etcd/ca.crt`
   - `/etc/kubernetes/pki/etcd/ca.key`

1. Tạo certificate cho từng thành viên.

   ```sh
   kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
   kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
   cp -R /etc/kubernetes/pki /tmp/${HOST2}/
   # dọn dẹp các certificate không thể tái sử dụng
   find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

   kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
   kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
   cp -R /etc/kubernetes/pki /tmp/${HOST1}/
   find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

   kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
   kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
   kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
   # Không cần di chuyển các certificate vì chúng dành cho HOST0

   # dọn dẹp các certificate không nên bị sao chép ra khỏi host này
   find /tmp/${HOST2} -name ca.key -type f -delete
   find /tmp/${HOST1} -name ca.key -type f -delete
   ```

1. Sao chép các certificate và cấu hình kubeadm.

   Các certificate đã được tạo và bây giờ chúng phải được chuyển đến
   các host tương ứng.

   ```sh
   USER=ubuntu
   HOST=${HOST1}
   scp -r /tmp/${HOST}/* ${USER}@${HOST}:
   ssh ${USER}@${HOST}
   USER@HOST $ sudo -Es
   root@HOST $ chown -R root:root pki
   root@HOST $ mv pki /etc/kubernetes/
   ```

1. Đảm bảo tất cả các file mong đợi đều tồn tại.

   Danh sách đầy đủ các file bắt buộc trên `$HOST0` là:

   ```
   /tmp/${HOST0}
   └── kubeadmcfg.yaml
   ---
   /etc/kubernetes/pki
   ├── apiserver-etcd-client.crt
   ├── apiserver-etcd-client.key
   └── etcd
       ├── ca.crt
       ├── ca.key
       ├── healthcheck-client.crt
       ├── healthcheck-client.key
       ├── peer.crt
       ├── peer.key
       ├── server.crt
       └── server.key
   ```

   Trên `$HOST1`:

   ```
   $HOME
   └── kubeadmcfg.yaml
   ---
   /etc/kubernetes/pki
   ├── apiserver-etcd-client.crt
   ├── apiserver-etcd-client.key
   └── etcd
       ├── ca.crt
       ├── healthcheck-client.crt
       ├── healthcheck-client.key
       ├── peer.crt
       ├── peer.key
       ├── server.crt
       └── server.key
   ```

   Trên `$HOST2`:

   ```
   $HOME
   └── kubeadmcfg.yaml
   ---
   /etc/kubernetes/pki
   ├── apiserver-etcd-client.crt
   ├── apiserver-etcd-client.key
   └── etcd
       ├── ca.crt
       ├── healthcheck-client.crt
       ├── healthcheck-client.key
       ├── peer.crt
       ├── peer.key
       ├── server.crt
       └── server.key
   ```

1. Tạo các static pod manifest.

   Bây giờ khi các certificate và cấu hình đã sẵn sàng, đã đến lúc tạo các
   manifest. Trên mỗi host, hãy chạy lệnh `kubeadm` để tạo static manifest
   cho etcd.

   ```sh
   root@HOST0 $ kubeadm init phase etcd local --config=/tmp/${HOST0}/kubeadmcfg.yaml
   root@HOST1 $ kubeadm init phase etcd local --config=$HOME/kubeadmcfg.yaml
   root@HOST2 $ kubeadm init phase etcd local --config=$HOME/kubeadmcfg.yaml
   ```

1. Tùy chọn: Kiểm tra sức khỏe (health) của cluster.

    Nếu `etcdctl` không có sẵn, bạn có thể chạy công cụ này bên trong một container image.
    Bạn sẽ làm điều đó trực tiếp với container runtime của mình bằng một công cụ như
    `crictl run` chứ không phải thông qua Kubernetes.

    ```sh
    ETCDCTL_API=3 etcdctl \
    --cert /etc/kubernetes/pki/etcd/peer.crt \
    --key /etc/kubernetes/pki/etcd/peer.key \
    --cacert /etc/kubernetes/pki/etcd/ca.crt \
    --endpoints https://${HOST0}:2379 endpoint health
    ...
    https://[HOST0 IP]:2379 is healthy: successfully committed proposal: took = 16.283339ms
    https://[HOST1 IP]:2379 is healthy: successfully committed proposal: took = 19.44402ms
    https://[HOST2 IP]:2379 is healthy: successfully committed proposal: took = 35.926451ms
    ```

    - Đặt `${HOST0}` thành địa chỉ IP của host mà bạn đang kiểm tra.

## Tiếp theo (What's next)

Khi bạn đã có một cluster etcd với 3 thành viên hoạt động, bạn có thể tiếp tục thiết lập một
control plane có tính sẵn sàng cao bằng
[phương pháp external etcd với kubeadm](08-high-availability-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 8:

1. Chạy hết tám bước của bài này xong, bạn đã có một cluster Kubernetes chưa? Kubelet trên ba
   host đó đang nói chuyện với API server nào?
2. Vì sao bài bắt tạo `20-etcd-service-manager.conf` thay vì để kubelet chạy bằng unit mà gói
   `kubeadm` đã cài sẵn?
3. Sau khi làm xong, `ca.key` của etcd nằm trên host nào? Vì sao bài cố ý xóa nó khỏi
   `/tmp/${HOST1}` và `/tmp/${HOST2}` trước khi `scp`?
4. Port 2379 và 2380 khác vai trò thế nào? Nếu firewall chỉ mở 2379 giữa ba host thì hỏng ở đâu?
5. Cluster lab của bạn (`k8s-master` 192.168.100.111 + hai worker) hiện chạy etcd ở đâu? Bài
   này có áp dụng cho nó không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Chưa.** Bạn mới có **một cluster etcd external ba thành viên**, chạy dưới dạng **static
   pod được quản lý bởi một kubelet**. Kubelet ở đây **không nói chuyện với API server nào cả**
   — cấu hình mà bài ghi cho nó đặt `address: 127.0.0.1` và `authorization.mode: AlwaysAllow`,
   đúng nghĩa một trình quản lý dịch vụ cục bộ. Phần dựng control plane nằm ở bài
   [08](08-high-availability-vi.md), và mục *Tiếp theo* của chính bài này trỏ sang đó.
2. Vì **etcd được tạo trước** khi có cluster, nên kubelet phải chạy ở một chế độ khác chế độ
   bình thường của kubeadm. Bài yêu cầu **ghi đè mức ưu tiên của service bằng cách tạo một unit
   file mới có độ ưu tiên cao hơn unit file kubelet do kubeadm cung cấp** — file
   `20-etcd-service-manager.conf` xóa `ExecStart` cũ rồi trỏ kubelet vào `kubelet.conf` riêng
   với `staticPodPath: /etc/kubernetes/manifests`. Unit gốc của kubeadm giả định có API server
   để bootstrap, thứ chưa tồn tại ở bước này.
3. Chỉ trên **`$HOST0`** — nơi bạn chạy `kubeadm init phase certs etcd-ca`. Danh sách file bắt
   buộc trong bài phản ánh đúng điều đó: `$HOST0` có cả `etcd/ca.crt` và `etcd/ca.key`, còn
   `$HOST1` và `$HOST2` **chỉ có `ca.crt`**. Lý do là **CA key ký được certificate mới**; các
   host member chỉ cần certificate đã ký của chính chúng, nên bài chép **các file *cần thiết***
   chứ không chép cả thư mục. Cùng logic với lệnh `find /etc/kubernetes/pki -not -name ca.crt
   -not -name ca.key -type f -delete` giữa các vòng: dọn certificate của host trước để không
   phát nhầm sang host sau.
4. **2380 là port peer** — lưu lượng giữa các etcd member với nhau (`listen-peer-urls`,
   `initial-advertise-peer-urls`, và các URL trong `initial-cluster`). **2379 là port client** —
   nơi `kube-apiserver` và `etcdctl` kết nối vào (`listen-client-urls`, `advertise-client-urls`).
   Mở mỗi 2379 thì `etcdctl ... endpoint health` có thể chạm được từng endpoint, nhưng ba member
   **không hình thành được cluster** vì không nói chuyện với nhau; mục *Trước khi bạn bắt đầu*
   yêu cầu **cả hai** port TCP 2379 và 2380 thông giữa ba host.
5. Etcd của cluster lab chạy **cục bộ ngay trên `k8s-master`**: câu đầu bài nói theo mặc định
   **kubeadm chạy một instance etcd cục bộ trên mỗi node control plane**, và Lab 00 dùng đúng
   mặc định đó. Nên bài này **không áp dụng** cho cluster lab — nó chỉ dùng khi bạn cố ý chọn
   external etcd, tức Lab 8c với bộ VM riêng, và phải làm **trước** khi dựng control plane.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

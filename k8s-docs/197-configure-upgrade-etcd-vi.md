# Vận hành cluster etcd cho Kubernetes (Operating etcd clusters for Kubernetes)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối — nhánh `/docs/tasks/`](00-ALO-TRINH-ADMIN.md#checkpoint-tiếp-nối--nhánh-docstasks)
→ [CP4 — etcd, backup và khôi phục thảm họa](00-ALO-TRINH-ADMIN.md#cp4--etcd-backup-và-khôi-phục-thảm-họa),
bài chính của checkpoint · Kiểm chứng bằng **bài tập bắt buộc của CP4**: backup etcd → cố ý xóa
vài Deployment → restore từ snapshot → chứng minh cluster trở về trạng thái cũ.

Bài này viết chung cho cả người tự dựng etcd bên ngoài kubeadm. Cluster lab của bạn dùng etcd
dạng static Pod do kubeadm dựng sẵn trên `k8s-master`, nên các mục khởi động etcd thủ công chỉ
cần đọc để hiểu bối cảnh; trọng tâm của lần đọc này là **backup và restore**.

**Phải hiểu ở lần đọc này:**

- Vì sao sức khỏe etcd quyết định sức khỏe cluster: etcd là hệ phân tán dựa trên leader, cần số
  member lẻ, nhạy với I/O đĩa và mạng; thiếu tài nguyên → heartbeat timeout → không bầu được
  leader → cluster **không thay đổi được trạng thái hiện tại**, tức không schedule Pod mới
  (Pod đã chạy vẫn chạy).
- Ranh giới `etcdctl` / `etcdutl`: `etcdctl` là client thao tác **qua mạng** với cluster đang
  chạy (member list, snapshot save…); `etcdutl` thao tác **trực tiếp trên file dữ liệu**
  (restore, defrag, kiểm tra snapshot). Dùng `etcdctl` cho snapshot status/restore đã lỗi thời
  từ v3.5.x.
- Quy trình backup: `etcdctl snapshot save` trên member đang chạy, kèm `--endpoints`,
  `--cacert`, `--cert`, `--key` (lấy từ mô tả Pod etcd); verify bằng
  `etcdutl snapshot status`. Snapshot chứa toàn bộ trạng thái Kubernetes → phải mã hóa file.
- Quy trình restore đúng thứ tự: **dừng tất cả API server → restore mọi etcd instance bằng
  `etcdutl snapshot restore --data-dir` → khởi động lại API server**, và nên restart cả
  kube-scheduler, kube-controller-manager, kubelet để không dùng dữ liệu cũ.
- Nguyên tắc vận hành member: thay member hỏng **ngay** và từng cái một (remove trước, add
  sau); không auto-scale etcd — production nên chạy tĩnh 5 member; truy cập etcd tương đương
  quyền root cluster nên chỉ API server được truy cập, giới hạn bằng TLS client certificate
  (`--client-cert-auth` + `--trusted-ca-file`).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Khởi động etcd thủ công bằng lệnh `etcd` (một node, nhiều node, sau load balancer) | cluster lab dùng etcd static Pod do kubeadm dựng, không khởi động tay | bài [07](07-setup-ha-etcd-with-kubeadm-vi.md) khi dựng etcd cluster ngoài |
| Tự sinh cặp key/cert cho etcd bằng script tls-setup | kubeadm đã sinh sẵn PKI trong `/etc/kubernetes/pki/etcd/` | [CP3 — vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#cp3--vòng-đời-chứng-chỉ) |
| Chống phân mảnh bằng công cụ `etcd-defrag` và CronJob | công cụ bên thứ ba, chỉ cần khi database tiến gần storage quota | khi vận hành thật, theo tài liệu maintenance của etcd |

---

etcd là một kho lưu trữ key-value nhất quán (consistent) và có tính sẵn sàng cao
(highly-available), được dùng làm nơi lưu trữ nền (backing store) cho toàn bộ dữ liệu cluster
của Kubernetes.

Nếu cluster Kubernetes của bạn dùng etcd làm backing store, hãy đảm bảo bạn có kế hoạch
[sao lưu (back up)](#backing-up-an-etcd-cluster) cho dữ liệu này.

Bạn có thể tìm thông tin chuyên sâu về etcd trong [tài liệu](https://etcd.io/docs/) chính thức.

## Trước khi bạn bắt đầu (Before you begin)

Trước khi làm theo các bước trong trang này để triển khai, quản lý, sao lưu hoặc khôi phục
etcd, bạn cần hiểu những kỳ vọng thông thường khi vận hành một cluster etcd. Hãy tham khảo
[tài liệu etcd](https://etcd.io/docs/) để có thêm bối cảnh.

Các điểm chính bao gồm:

* Phiên bản etcd tối thiểu được khuyến nghị để chạy trong production là `3.4.29+` và `3.5.11+`.

* etcd là một hệ thống phân tán dựa trên leader (leader-based). Hãy đảm bảo leader định kỳ
  gửi heartbeat đúng hạn tới tất cả các follower để giữ cluster ổn định.

* Bạn nên chạy etcd như một cluster với số lượng member lẻ.

* Cố gắng đảm bảo không xảy ra tình trạng thiếu hụt tài nguyên (resource starvation).

  Hiệu năng và độ ổn định của cluster rất nhạy cảm với mạng và I/O đĩa. Bất kỳ sự thiếu hụt
  tài nguyên nào cũng có thể dẫn đến heartbeat timeout, gây bất ổn cho cluster. Một etcd bất
  ổn nghĩa là không có leader nào được bầu. Trong hoàn cảnh đó, cluster không thể thực hiện
  bất kỳ thay đổi nào đối với trạng thái hiện tại của nó, đồng nghĩa với việc không có Pod
  mới nào được lập lịch (schedule).

### Yêu cầu tài nguyên cho etcd (Resource requirements for etcd)

Vận hành etcd với tài nguyên hạn chế chỉ phù hợp cho mục đích thử nghiệm. Để triển khai trong
production, cần cấu hình phần cứng nâng cao. Trước khi triển khai etcd trong production, hãy
xem [tài liệu tham chiếu về yêu cầu tài nguyên](https://etcd.io/docs/current/op-guide/hardware/#example-hardware-configurations).

Giữ cho cluster etcd ổn định là yếu tố then chốt đối với sự ổn định của cluster Kubernetes.
Vì vậy, hãy chạy cluster etcd trên các máy chuyên dụng hoặc môi trường được cô lập để
[đảm bảo yêu cầu tài nguyên](https://etcd.io/docs/current/op-guide/hardware/).

### Công cụ (Tools)

Tùy vào kết quả cụ thể mà bạn đang hướng tới, bạn sẽ cần công cụ `etcdctl` hoặc công cụ
`etcdutl` (có thể cần cả hai).

## Hiểu về etcdctl và etcdutl (Understanding etcdctl and etcdutl)

`etcdctl` và `etcdutl` là các công cụ dòng lệnh dùng để tương tác với cluster etcd, nhưng
chúng phục vụ những mục đích khác nhau:

- `etcdctl`: Đây là client dòng lệnh chính để tương tác với etcd **qua mạng**. Nó được dùng
  cho các thao tác hằng ngày như quản lý key và value, quản trị cluster, kiểm tra sức khỏe,
  và nhiều việc khác.

- `etcdutl`: Đây là tiện ích quản trị được thiết kế để thao tác **trực tiếp trên các file dữ
  liệu** của etcd, bao gồm di chuyển (migrate) dữ liệu giữa các phiên bản etcd, chống phân
  mảnh (defragment) database, khôi phục (restore) snapshot, và kiểm tra tính nhất quán của
  dữ liệu. Với các thao tác qua mạng, nên dùng `etcdctl`.

Để biết thêm thông tin về `etcdutl`, bạn có thể tham khảo
[tài liệu khôi phục của etcd](https://etcd.io/docs/v3.5/op-guide/recovery/).

## Khởi động cluster etcd (Starting etcd clusters)

Mục này trình bày cách khởi động một cluster etcd một node (single-node) và nhiều node
(multi-node).

Hướng dẫn này giả định rằng `etcd` đã được cài đặt sẵn.

### Cluster etcd một node (Single-node etcd cluster)

Chỉ dùng cluster etcd một node cho mục đích thử nghiệm.

1. Chạy lệnh sau:

   ```sh
   etcd --listen-client-urls=http://$PRIVATE_IP:2379 \
      --advertise-client-urls=http://$PRIVATE_IP:2379
   ```

2. Khởi động Kubernetes API server với flag `--etcd-servers=$PRIVATE_IP:2379`.

   Đảm bảo `PRIVATE_IP` được đặt bằng IP client của etcd.

### Cluster etcd nhiều node (Multi-node etcd cluster)

Để có độ bền dữ liệu (durability) và tính sẵn sàng cao (high availability), hãy chạy etcd như
một cluster nhiều node trong production và sao lưu nó định kỳ. Một cluster 5 member được
khuyến nghị trong production. Để biết thêm thông tin, xem
[tài liệu FAQ](https://etcd.io/docs/current/faq/#what-is-failure-tolerance).

Vì bạn đang dùng Kubernetes, bạn có lựa chọn chạy etcd như một container bên trong một hoặc
nhiều Pod. Công cụ `kubeadm` mặc định thiết lập etcd dưới dạng static pod, hoặc bạn có thể
triển khai một [cluster riêng biệt](07-setup-ha-etcd-with-kubeadm-vi.md) và chỉ định kubeadm
dùng cluster etcd đó làm backing store cho control plane.

Bạn cấu hình một cluster etcd hoặc bằng thông tin member tĩnh (static) hoặc bằng khám phá
động (dynamic discovery). Để biết thêm thông tin về việc lập cluster, xem
[tài liệu clustering của etcd](https://etcd.io/docs/current/op-guide/clustering/).

Lấy ví dụ, xét một cluster etcd 5 member chạy với các client URL sau: `http://$IP1:2379`,
`http://$IP2:2379`, `http://$IP3:2379`, `http://$IP4:2379`, và `http://$IP5:2379`. Để khởi
động một Kubernetes API server:

1. Chạy lệnh sau:

   ```shell
   etcd --listen-client-urls=http://$IP1:2379,http://$IP2:2379,http://$IP3:2379,http://$IP4:2379,http://$IP5:2379 --advertise-client-urls=http://$IP1:2379,http://$IP2:2379,http://$IP3:2379,http://$IP4:2379,http://$IP5:2379
   ```

2. Khởi động các Kubernetes API server với flag
   `--etcd-servers=$IP1:2379,$IP2:2379,$IP3:2379,$IP4:2379,$IP5:2379`.

   Đảm bảo các biến `IP<n>` được đặt bằng các địa chỉ IP client của bạn.

### Cluster etcd nhiều node với bộ cân bằng tải (Multi-node etcd cluster with load balancer)

Để chạy một cluster etcd có cân bằng tải:

1. Thiết lập một cluster etcd.
2. Cấu hình một bộ cân bằng tải (load balancer) đứng trước cluster etcd.
   Ví dụ, gọi địa chỉ của bộ cân bằng tải là `$LB`.
3. Khởi động các Kubernetes API Server với flag `--etcd-servers=$LB:2379`.

## Bảo mật cluster etcd (Securing etcd clusters)

Quyền truy cập vào etcd tương đương với quyền root trong cluster, vì vậy lý tưởng nhất là chỉ
API server mới được truy cập nó. Xét đến độ nhạy cảm của dữ liệu, khuyến nghị chỉ cấp quyền
cho những node thực sự cần truy cập cluster etcd.

Để bảo mật etcd, hãy thiết lập các quy tắc firewall hoặc dùng các tính năng bảo mật do etcd
cung cấp. Các tính năng bảo mật của etcd dựa trên hạ tầng khóa công khai x509 (x509 Public
Key Infrastructure — PKI). Để bắt đầu, hãy thiết lập các kênh truyền thông an toàn bằng cách
sinh một cặp key và certificate. Ví dụ, dùng cặp `peer.key` và `peer.cert` để bảo mật truyền
thông giữa các member etcd, và `client.key` cùng `client.cert` để bảo mật truyền thông giữa
etcd và các client của nó. Xem [các script ví dụ](https://github.com/coreos/etcd/tree/master/hack/tls-setup)
do dự án etcd cung cấp để sinh các cặp key và file CA phục vụ xác thực client.

### Bảo mật kênh truyền thông (Securing communication)

Để cấu hình etcd với truyền thông an toàn giữa các peer, chỉ định các flag
`--peer-key-file=peer.key` và `--peer-cert-file=peer.cert`, và dùng HTTPS làm schema của URL.

Tương tự, để cấu hình etcd với truyền thông client an toàn, chỉ định các flag
`--key=k8sclient.key` và `--cert=k8sclient.cert`, và dùng HTTPS làm schema của URL. Dưới đây
là một ví dụ về lệnh phía client sử dụng truyền thông an toàn:

```
ETCDCTL_API=3 etcdctl --endpoints 10.2.0.9:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  member list
```

### Giới hạn truy cập cluster etcd (Limiting access of etcd clusters)

Sau khi đã cấu hình truyền thông an toàn, hãy giới hạn quyền truy cập cluster etcd chỉ cho
các Kubernetes API server bằng xác thực TLS.

Ví dụ, xét cặp key `k8sclient.key` và `k8sclient.cert` được tin cậy bởi CA `etcd.ca`. Khi
etcd được cấu hình với `--client-cert-auth` cùng với TLS, nó xác minh certificate từ các
client bằng các CA của hệ thống hoặc CA được truyền vào qua flag `--trusted-ca-file`. Chỉ
định các flag `--client-cert-auth=true` và `--trusted-ca-file=etcd.ca` sẽ giới hạn quyền
truy cập cho các client có certificate `k8sclient.cert`.

Một khi etcd đã được cấu hình đúng, chỉ những client có certificate hợp lệ mới truy cập được
nó. Để cấp quyền truy cập cho các Kubernetes API server, cấu hình chúng với các flag
`--etcd-certfile=k8sclient.cert`, `--etcd-keyfile=k8sclient.key` và `--etcd-cafile=ca.cert`.

> **Ghi chú:**
> Kubernetes không có kế hoạch hỗ trợ cơ chế xác thực (authentication) riêng của etcd.

## Thay thế một etcd member bị lỗi (Replacing a failed etcd member)

Cluster etcd đạt tính sẵn sàng cao bằng cách chịu đựng được một số ít member gặp sự cố. Tuy
nhiên, để cải thiện sức khỏe tổng thể của cluster, hãy thay thế các member bị lỗi ngay lập
tức. Khi có nhiều member gặp sự cố, hãy thay thế chúng từng cái một. Việc thay thế một member
bị lỗi gồm hai bước: xóa member bị lỗi và thêm một member mới.

Mặc dù etcd giữ các member ID duy nhất trong nội bộ, khuyến nghị dùng một tên duy nhất cho
mỗi member để tránh sai sót của con người. Ví dụ, xét một cluster etcd 3 member. Gọi các URL
là `member1=http://10.0.0.1`, `member2=http://10.0.0.2`, và `member3=http://10.0.0.3`. Khi
`member1` gặp sự cố, thay thế nó bằng `member4=http://10.0.0.4`.

1. Lấy member ID của `member1` bị lỗi:

   ```shell
   etcdctl --endpoints=http://10.0.0.2,http://10.0.0.3 member list
   ```

   Thông báo sau được hiển thị:

   ```console
   8211f1d0f64f3269, started, member1, http://10.0.0.1:2380, http://10.0.0.1:2379
   91bc3c398fb3c146, started, member2, http://10.0.0.2:2380, http://10.0.0.2:2379
   fd422379fda50e48, started, member3, http://10.0.0.3:2380, http://10.0.0.3:2379
   ```

1. Thực hiện một trong hai cách sau:

   1. Nếu mỗi Kubernetes API server được cấu hình để giao tiếp với tất cả các member etcd,
      hãy loại member bị lỗi khỏi flag `--etcd-servers`, rồi khởi động lại từng Kubernetes
      API server.
   1. Nếu mỗi Kubernetes API server chỉ giao tiếp với một member etcd duy nhất, hãy dừng
      Kubernetes API server đang giao tiếp với etcd bị lỗi.

1. Dừng etcd server trên node bị hỏng. Có khả năng các client khác ngoài Kubernetes API
   server cũng đang tạo lưu lượng tới etcd, và nên dừng toàn bộ lưu lượng để ngăn các thao
   tác ghi vào thư mục dữ liệu (data directory).

1. Xóa member bị lỗi:

   ```shell
   etcdctl member remove 8211f1d0f64f3269
   ```

   Thông báo sau được hiển thị:

   ```console
   Removed member 8211f1d0f64f3269 from cluster
   ```

1. Thêm member mới:

   ```shell
   etcdctl member add member4 --peer-urls=http://10.0.0.4:2380
   ```

   Thông báo sau được hiển thị:

   ```console
   Member 2be1eb8f84b7f63e added to cluster ef37ad9dc622a7c4
   ```

1. Khởi động member vừa thêm trên máy có IP `10.0.0.4`:

   ```shell
   export ETCD_NAME="member4"
   export ETCD_INITIAL_CLUSTER="member2=http://10.0.0.2:2380,member3=http://10.0.0.3:2380,member4=http://10.0.0.4:2380"
   export ETCD_INITIAL_CLUSTER_STATE=existing
   etcd [flags]
   ```

1. Thực hiện một trong hai cách sau:

   1. Nếu mỗi Kubernetes API server được cấu hình để giao tiếp với tất cả các member etcd,
      hãy thêm member vừa được thêm vào flag `--etcd-servers`, rồi khởi động lại từng
      Kubernetes API server.
   1. Nếu mỗi Kubernetes API server chỉ giao tiếp với một member etcd duy nhất, hãy khởi
      động Kubernetes API server đã bị dừng ở bước 2. Sau đó cấu hình các client của
      Kubernetes API server để định tuyến các request trở lại Kubernetes API server đã bị
      dừng. Việc này thường có thể thực hiện bằng cách cấu hình một bộ cân bằng tải.

Để biết thêm thông tin về việc cấu hình lại cluster, xem
[tài liệu tái cấu hình etcd](https://etcd.io/docs/current/op-guide/runtime-configuration/#remove-a-member).

## Sao lưu cluster etcd (Backing up an etcd cluster) {#backing-up-an-etcd-cluster}

Tất cả các object của Kubernetes được lưu trong etcd. Việc sao lưu định kỳ dữ liệu cluster
etcd là quan trọng để khôi phục cluster Kubernetes trong các kịch bản thảm họa (disaster),
chẳng hạn mất toàn bộ các node control plane. File snapshot chứa toàn bộ trạng thái
Kubernetes và các thông tin quan trọng. Để giữ an toàn cho dữ liệu Kubernetes nhạy cảm, hãy
mã hóa (encrypt) các file snapshot.

Việc sao lưu một cluster etcd có thể được thực hiện theo hai cách: snapshot tích hợp sẵn của
etcd (built-in snapshot) và snapshot volume (volume snapshot).

### Snapshot tích hợp sẵn (Built-in snapshot)

etcd hỗ trợ snapshot tích hợp sẵn. Một snapshot có thể được tạo từ một member đang chạy
(live) bằng lệnh `etcdctl snapshot save`, hoặc bằng cách sao chép file `member/snap/db` từ
một [thư mục dữ liệu](https://etcd.io/docs/current/op-guide/configuration/#--data-dir) etcd
hiện không được tiến trình etcd nào sử dụng. Việc tạo snapshot không ảnh hưởng đến hiệu năng
của member.

Dưới đây là một ví dụ tạo snapshot của keyspace được phục vụ bởi `$ENDPOINT` vào file
`snapshot.db`:

```shell
ETCDCTL_API=3 etcdctl --endpoints $ENDPOINT snapshot save snapshot.db
```

Kiểm tra snapshot:

#### Dùng etcdutl (Use etcdutl)

Ví dụ dưới đây minh họa cách dùng công cụ `etcdutl` để kiểm tra một snapshot:

```shell
etcdutl --write-out=table snapshot status snapshot.db 
```

Lệnh này sẽ sinh ra output tương tự ví dụ dưới đây:

```console
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| fe01cf57 |       10 |          7 | 2.1 MB     |
+----------+----------+------------+------------+
```

#### Dùng etcdctl — đã lỗi thời (Use etcdctl, Deprecated)

> **Ghi chú:**
> Việc dùng `etcdctl snapshot status` đã bị **loại bỏ dần (deprecated)** kể từ etcd v3.5.x
> và dự kiến bị gỡ bỏ khỏi etcd v3.6. Khuyến nghị dùng
> [`etcdutl`](https://github.com/etcd-io/etcd/blob/main/etcdutl/README.md) thay thế.

Ví dụ dưới đây minh họa cách dùng công cụ `etcdctl` để kiểm tra một snapshot:

```shell
export ETCDCTL_API=3
etcdctl --write-out=table snapshot status snapshot.db
```

Lệnh này sẽ sinh ra output tương tự ví dụ dưới đây:

```console
Deprecated: Use `etcdutl snapshot status` instead.

+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| fe01cf57 |       10 |          7 | 2.1 MB     |
+----------+----------+------------+------------+
```

### Snapshot volume (Volume snapshot)

Nếu etcd đang chạy trên một storage volume hỗ trợ sao lưu, chẳng hạn Amazon Elastic Block
Store, hãy sao lưu dữ liệu etcd bằng cách tạo snapshot của storage volume đó.

### Tạo snapshot với các tùy chọn của etcdctl (Snapshot using etcdctl options)

Chúng ta cũng có thể tạo snapshot bằng nhiều tùy chọn khác nhau mà etcdctl cung cấp.
Ví dụ:

```shell
ETCDCTL_API=3 etcdctl -h 
``` 

sẽ liệt kê các tùy chọn khả dụng của etcdctl. Ví dụ, bạn có thể tạo snapshot bằng cách chỉ
định endpoint, các certificate và key như dưới đây:

```shell
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file> \
  snapshot save <backup-file-location>
```

trong đó `trusted-ca-file`, `cert-file` và `key-file` có thể lấy được từ phần mô tả
(description) của Pod etcd.

## Mở rộng cluster etcd (Scaling out etcd clusters)

Mở rộng (scale out) cluster etcd làm tăng tính sẵn sàng bằng cách đánh đổi hiệu năng. Việc
mở rộng không làm tăng hiệu năng hay năng lực của cluster. Nguyên tắc chung là không mở rộng
hay thu hẹp (scale out/in) cluster etcd. Không cấu hình bất kỳ nhóm auto scaling nào cho
cluster etcd. Khuyến nghị mạnh mẽ rằng luôn chạy một cluster etcd tĩnh 5 member cho các
cluster Kubernetes production ở mọi quy mô được hỗ trợ chính thức.

Một cách mở rộng hợp lý là nâng cấp cluster 3 member lên 5 member khi cần độ tin cậy cao hơn.
Xem [tài liệu tái cấu hình etcd](https://etcd.io/docs/current/op-guide/runtime-configuration/#remove-a-member)
để biết cách thêm member vào một cluster đang tồn tại.

## Khôi phục cluster etcd (Restoring an etcd cluster)

> **Thận trọng:**
> Nếu có bất kỳ API server nào đang chạy trong cluster của bạn, bạn không nên cố khôi phục
> các instance của etcd. Thay vào đó, hãy làm theo các bước sau để khôi phục etcd:
>
> - dừng *tất cả* các instance API server
> - khôi phục trạng thái trên tất cả các instance etcd
> - khởi động lại tất cả các instance API server
>
> Dự án Kubernetes cũng khuyến nghị khởi động lại các thành phần Kubernetes
> (`kube-scheduler`, `kube-controller-manager`, `kubelet`) để đảm bảo chúng không phụ thuộc
> vào dữ liệu đã cũ (stale). Trên thực tế, việc khôi phục mất một chút thời gian. Trong quá
> trình khôi phục, các thành phần quan trọng sẽ mất leader lock và tự khởi động lại.

etcd hỗ trợ khôi phục từ các snapshot được tạo bởi một tiến trình etcd thuộc cùng phiên bản
[major.minor](https://semver.org/). Khôi phục từ một phiên bản patch khác của etcd cũng được
hỗ trợ. Thao tác khôi phục (restore) được dùng để khôi phục dữ liệu của một cluster đã hỏng.

Trước khi bắt đầu thao tác khôi phục, phải có sẵn một file snapshot. Nó có thể là file
snapshot từ một lần sao lưu trước đó, hoặc từ một
[thư mục dữ liệu](https://etcd.io/docs/current/op-guide/configuration/#--data-dir) còn sót lại.

#### Dùng etcdutl (Use etcdutl)

Khi khôi phục cluster bằng [`etcdutl`](https://github.com/etcd-io/etcd/blob/main/etcdutl/README.md),
dùng tùy chọn `--data-dir` để chỉ định thư mục mà cluster sẽ được khôi phục vào:

```shell
etcdutl --data-dir <data-dir-location> snapshot restore snapshot.db
```

trong đó `<data-dir-location>` là thư mục sẽ được tạo ra trong quá trình khôi phục.

#### Dùng etcdctl — đã lỗi thời (Use etcdctl, Deprecated)

> **Ghi chú:**
> Việc dùng `etcdctl` cho thao tác khôi phục đã bị **loại bỏ dần (deprecated)** kể từ etcd
> v3.5.x và dự kiến bị gỡ bỏ khỏi etcd v3.6. Khuyến nghị dùng
> [`etcdutl`](https://github.com/etcd-io/etcd/blob/main/etcdutl/README.md) thay thế.

Ví dụ dưới đây minh họa cách dùng công cụ `etcdctl` cho thao tác khôi phục:

```shell
export ETCDCTL_API=3
etcdctl --data-dir <data-dir-location> snapshot restore snapshot.db
```

Nếu `<data-dir-location>` trùng với thư mục cũ, hãy xóa nó và dừng tiến trình etcd trước khi
khôi phục cluster. Nếu không, hãy thay đổi cấu hình etcd và khởi động lại tiến trình etcd sau
khi khôi phục để nó dùng thư mục dữ liệu mới: trước tiên đổi `volumes.hostPath.path` của
`name: etcd-data` trong `/etc/kubernetes/manifests/etcd.yaml` thành `<data-dir-location>`,
sau đó thực thi `kubectl -n kube-system delete pod <name-of-etcd-pod>` hoặc
`systemctl restart kubelet.service` (hoặc cả hai).

Để biết thêm thông tin và các ví dụ về khôi phục cluster từ một file snapshot, xem
[tài liệu khôi phục thảm họa của etcd](https://etcd.io/docs/current/op-guide/recovery/#restoring-a-cluster).

Nếu các URL truy cập của cluster đã khôi phục thay đổi so với cluster trước đó, Kubernetes
API server phải được cấu hình lại tương ứng. Trong trường hợp này, khởi động lại các
Kubernetes API server với flag `--etcd-servers=$NEW_ETCD_CLUSTER` thay cho flag
`--etcd-servers=$OLD_ETCD_CLUSTER`. Thay `$NEW_ETCD_CLUSTER` và `$OLD_ETCD_CLUSTER` bằng các
địa chỉ IP tương ứng. Nếu có một bộ cân bằng tải đứng trước cluster etcd, bạn có thể cần cập
nhật bộ cân bằng tải thay vì API server.

Nếu phần lớn (majority) các member etcd đã hỏng vĩnh viễn, cluster etcd được coi là đã hỏng.
Trong kịch bản này, Kubernetes không thể thực hiện bất kỳ thay đổi nào đối với trạng thái
hiện tại của nó. Mặc dù các Pod đã được lập lịch có thể tiếp tục chạy, không Pod mới nào có
thể được lập lịch. Trong những trường hợp như vậy, hãy khôi phục cluster etcd và có thể phải
cấu hình lại các Kubernetes API server để khắc phục sự cố.

## Nâng cấp cluster etcd (Upgrading etcd clusters)

> **Thận trọng:**
> Trước khi bắt đầu nâng cấp, hãy sao lưu cluster etcd của bạn trước.

Để biết chi tiết về nâng cấp etcd, tham khảo tài liệu
[nâng cấp etcd](https://etcd.io/docs/latest/upgrades/).

## Bảo trì cluster etcd (Maintaining etcd clusters)

Để biết thêm chi tiết về bảo trì etcd, vui lòng tham khảo tài liệu
[bảo trì etcd](https://etcd.io/docs/latest/op-guide/maintenance/).

### Chống phân mảnh cluster (Cluster defragmentation)

> **Ghi chú:** Mục này liên kết đến một dự án bên thứ ba cung cấp chức năng mà Kubernetes
> cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về dự án này. Để thêm một dự
> án vào danh sách này, hãy đọc
> [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> trước khi gửi thay đổi.

Chống phân mảnh (defragmentation) là một thao tác tốn kém, vì vậy nên thực hiện nó càng ít
càng tốt. Mặt khác, cũng cần đảm bảo không member etcd nào vượt quá hạn mức lưu trữ (storage
quota). Dự án Kubernetes khuyến nghị rằng khi bạn thực hiện chống phân mảnh, hãy dùng một
công cụ như [etcd-defrag](https://github.com/ahrtr/etcd-defrag).

Bạn cũng có thể chạy công cụ chống phân mảnh như một CronJob của Kubernetes để đảm bảo việc
chống phân mảnh diễn ra đều đặn. Xem
[`etcd-defrag-cronjob.yaml`](https://github.com/ahrtr/etcd-defrag/blob/main/doc/etcd-defrag-cronjob.yaml)
để biết chi tiết.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở CP4:

1. Cluster lab của bạn chỉ có một member etcd trên `k8s-master`. Giả sử etcd mất khả năng ghi
   (đĩa hỏng) nhưng hai worker vẫn chạy bình thường — theo lập luận của bài, chuyện gì xảy ra
   với các Pod đang chạy và với các Pod mới?
2. Vì sao quy trình khôi phục bắt buộc phải **dừng tất cả API server trước**, khôi phục xong
   mới khởi động lại — thay vì restore "nóng" trong khi API server vẫn chạy? Và vì sao nên
   restart thêm `kube-scheduler`, `kube-controller-manager`, `kubelet`?
3. **Câu bẫy.** `etcdctl` và `etcdutl` đều có lệnh `snapshot` — vậy khi tạo snapshot và khi
   restore snapshot, mỗi việc dùng công cụ nào? Vì sao không dùng một công cụ cho cả hai?
4. Thay một member hỏng trong cluster 3 member: thứ tự các bước chính là gì, và vì sao bài
   dặn phải dừng cả etcd server trên node hỏng trước khi remove?
5. Vì sao bài nói "truy cập etcd tương đương quyền root trong cluster", và cặp flag nào trên
   etcd giới hạn để chỉ client mang certificate hợp lệ (như API server) mới truy cập được?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Khi etcd không còn ghi được (không có leader hoặc member duy nhất hỏng), **cluster không
   thể thay đổi trạng thái hiện tại**: các Pod đã được lập lịch **có thể tiếp tục chạy** vì
   kubelet và container runtime trên worker không cần etcd để duy trì container, nhưng
   **không Pod mới nào được lập lịch** và mọi thay đổi object đều thất bại. Đây chính là lý
   do bài xếp sự ổn định của etcd là điều kiện sống còn của cluster.
2. Vì nếu API server còn chạy trong lúc dữ liệu etcd bị thay bên dưới, nó sẽ phục vụ và ghi
   đè dựa trên **dữ liệu đã cũ (stale)**, làm trạng thái sau khôi phục không nhất quán. Thứ
   tự đúng: **dừng tất cả instance API server → khôi phục trạng thái trên tất cả instance
   etcd → khởi động lại API server**. Restart thêm `kube-scheduler`,
   `kube-controller-manager`, `kubelet` để chúng không tiếp tục làm việc dựa trên cache cũ;
   trong lúc khôi phục các thành phần này cũng mất leader lock và tự khởi động lại.
3. **Tạo snapshot dùng `etcdctl snapshot save`** — vì tạo snapshot là thao tác **qua mạng**
   với member đang chạy, đúng vai trò client của `etcdctl`. **Restore (và kiểm tra snapshot)
   dùng `etcdutl`** — vì restore là thao tác **trực tiếp trên file dữ liệu**, không đi qua
   cluster đang chạy. Chỗ dễ nhầm: `etcdctl snapshot status` và `etcdctl snapshot restore`
   vẫn tồn tại nhưng đã **deprecated từ v3.5.x** và dự kiến gỡ ở v3.6, nên không lấy chúng
   làm thói quen.
4. Thứ tự: lấy member ID (`etcdctl member list` qua các member còn sống) → gỡ member hỏng
   khỏi cấu hình/`--etcd-servers` của API server (hoặc dừng API server nói chuyện với nó) →
   **dừng etcd trên node hỏng** → `etcdctl member remove` → `etcdctl member add` member mới
   với `ETCD_INITIAL_CLUSTER_STATE=existing` → cập nhật lại API server. Phải dừng etcd trên
   node hỏng vì ngoài API server có thể còn client khác đang tạo lưu lượng; dừng hết lưu
   lượng để **ngăn các thao tác ghi vào thư mục dữ liệu** của member đã bị loại.
5. Vì **toàn bộ object Kubernetes nằm trong etcd** — ai đọc/ghi được etcd thì đọc/ghi được
   mọi thứ trong cluster (kể cả Secret), bỏ qua toàn bộ lớp xác thực và phân quyền của API
   server. Cặp flag giới hạn: **`--client-cert-auth=true` + `--trusted-ca-file=<CA>`** — etcd
   chỉ chấp nhận client có certificate do CA đó ký; API server được cấp quyền qua
   `--etcd-certfile`, `--etcd-keyfile`, `--etcd-cafile`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng, rồi làm bài tập bắt buộc của
[CP4](00-ALO-TRINH-ADMIN.md#cp4--etcd-backup-và-khôi-phục-thảm-họa) trên cluster lab trước khi
sang checkpoint kế.

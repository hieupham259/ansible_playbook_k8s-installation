# Các Volume (Volumes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/volumes/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](LO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 2/16 · Kiểm chứng ở
Lab 6a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này rất dài vì mục *Các loại volume* liệt kê **toàn bộ** loại volume Kubernetes từng hỗ
trợ, kể cả những loại đã bị gỡ bỏ hoặc vô hiệu hóa. Bạn chỉ cần đọc kỹ năm loại: `emptyDir`,
`hostPath`, `configMap`, `secret`, `persistentVolumeClaim`. Phần còn lại của danh sách lướt
qua để biết nó tồn tại, đừng học thuộc tham số.

**Phải hiểu ở lần đọc này:**

- Cách khai báo và mount: volume liệt kê ở `.spec.volumes`, còn nơi mount khai riêng cho
  **từng container** ở `.spec.containers[*].volumeMounts`; volume không mount lồng trong volume
  khác — mục *Cách volume hoạt động*.
- Ranh giới vòng đời của `emptyDir`: container crash rồi khởi động lại thì dữ liệu còn, nhưng
  Pod bị gỡ khỏi node thì dữ liệu **xóa vĩnh viễn** — mục *emptyDir*.
- `emptyDir.medium: "Memory"` đổi backing sang tmpfs, và file ghi vào đó bị tính vào **memory
  limit** của container; không đặt `sizeLimit` thì volume lấy bằng allocatable memory của node.
- Vì sao `hostPath` là lối thoát nguy hiểm: lộ credential và API đặc quyền của node, cùng một
  PodTemplate chạy khác nhau trên các node khác nhau, và dung lượng nó dùng **không** được tính
  vào ephemeral storage — mục *hostPath*.
- `configMap` và `secret` luôn được mount `readOnly` (secret nằm trên tmpfs), và container mount
  chúng qua `subPath` sẽ **không nhận cập nhật**; `persistentVolumeClaim` chỉ là một loại volume
  nữa, nó là cửa dẫn sang bài [92](92-persistent-volumes-vi.md).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `gcePersistentDisk`, `gitRepo`, `portworxVolume`, *flexVolume* | driver in-tree đã lỗi thời, bị gỡ hoặc bị vô hiệu hóa | không cần |
| `fc`, `iscsi`, `nfs`, `image` | cluster lab chưa có backend lưu trữ nào loại này | không cần |
| `local` | chỉ dùng được dưới dạng PersistentVolume tạo tĩnh | bài [92](92-persistent-volumes-vi.md), [96](96-storage-classes-vi.md) |
| `downwardAPI`, `projected` | đã gặp ở giai đoạn 3; bản đầy đủ là một bài riêng | bài [93](93-projected-volumes-vi.md) |
| *csi* và *Di trú từ plugin in-tree sang driver CSI* | lúc này chỉ cần biết CSI là cách nối lưu trữ ngoài vào cluster | bài [96](96-storage-classes-vi.md) và Lab 6a |
| *Sử dụng subPath với biến môi trường được mở rộng* | mẹo cấu hình, không phải cơ chế lưu trữ | không cần |
| *Lan truyền mount* | tính năng cấp thấp, chỉ dành cho driver lưu trữ và container đặc quyền | giai đoạn 9 |
| *Mount chỉ đọc đệ quy* | phụ thuộc kernel và container runtime | giai đoạn 9 |

---

_Volume_ trong Kubernetes cung cấp một cách để các container trong một Pod
truy cập và chia sẻ dữ liệu thông qua hệ thống file (filesystem). Có nhiều loại volume khác nhau
mà bạn có thể dùng cho các mục đích khác nhau, chẳng hạn:

- điền nội dung một file cấu hình dựa trên một ConfigMap hoặc một Secret
- cung cấp không gian nháp tạm thời (scratch space) cho một Pod
- chia sẻ một filesystem giữa hai container khác nhau trong cùng một Pod
- chia sẻ một filesystem giữa hai Pod khác nhau (kể cả khi các Pod đó chạy trên các node khác nhau)
- lưu trữ dữ liệu bền vững để dữ liệu vẫn khả dụng ngay cả khi Pod khởi động lại hoặc bị thay thế
- truyền thông tin cấu hình cho một ứng dụng chạy trong container, dựa trên thông tin chi tiết
  của Pod chứa container đó
  (ví dụ: cho một sidecar container biết Pod đang chạy trong namespace nào)
- cung cấp quyền truy cập chỉ đọc (read-only) vào dữ liệu trong một container image khác

Việc chia sẻ dữ liệu có thể diễn ra giữa các process cục bộ khác nhau bên trong một container,
giữa các container khác nhau, hoặc giữa các Pod.

## Tại sao volume quan trọng (Why volumes are important)

- **Tính bền vững của dữ liệu (data persistence):** Các file trên đĩa trong một container là
  tạm thời (ephemeral), điều này gây ra một số vấn đề cho các ứng dụng không đơn giản
  khi chạy trong container. Một vấn đề xảy ra khi container bị crash hoặc bị dừng;
  trạng thái của container không được lưu lại, vì vậy toàn bộ các file được tạo hoặc chỉnh sửa
  trong suốt vòng đời của container đều bị mất.
  Sau khi crash, kubelet khởi động lại container với trạng thái sạch.

- **Lưu trữ dùng chung (shared storage):** Một vấn đề khác xảy ra khi nhiều container cùng chạy
  trong một `Pod` và cần chia sẻ file với nhau. Việc thiết lập và truy cập
  một filesystem dùng chung cho tất cả các container có thể là một thách thức.

Lớp trừu tượng volume của Kubernetes
có thể giúp bạn giải quyết cả hai vấn đề này.

Trước khi tìm hiểu về volume, PersistentVolume và PersistentVolumeClaim, bạn nên đọc về
[Pod](./46-pods-vi.md) và bảo đảm rằng bạn hiểu cách
Kubernetes dùng Pod để chạy container.

## Cách volume hoạt động (How volumes work)

Kubernetes hỗ trợ nhiều loại volume. Một Pod
có thể dùng đồng thời số lượng bất kỳ các loại volume.
Các loại [volume tạm thời (ephemeral volume)](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/) có vòng đời gắn với một Pod cụ thể,
nhưng [persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) tồn tại vượt ra ngoài
vòng đời của bất kỳ Pod riêng lẻ nào. Khi một Pod không còn tồn tại, Kubernetes hủy các ephemeral volume;
tuy nhiên, Kubernetes không hủy các persistent volume.
Với mọi loại volume trong một Pod nhất định, dữ liệu được bảo toàn qua các lần khởi động lại container.

Về bản chất, một volume là một thư mục, có thể chứa sẵn dữ liệu bên trong,
mà các container trong một Pod có thể truy cập được. Thư mục đó được tạo ra như thế nào,
phương tiện lưu trữ (medium) đứng sau nó, và nội dung của nó được quyết định bởi
loại volume cụ thể được sử dụng.

Để dùng một volume, hãy chỉ định các volume cần cung cấp cho Pod trong `.spec.volumes`
và khai báo nơi mount các volume đó vào container trong `.spec.containers[*].volumeMounts`.

Khi một Pod được khởi chạy, một process trong container nhìn thấy một filesystem được ghép từ
nội dung ban đầu của container image, cộng với các volume
(nếu được định nghĩa) mount bên trong container.
Process đó thấy một root filesystem ban đầu trùng khớp với nội dung của container image.
Mọi thao tác ghi vào bên trong cây phân cấp filesystem đó, nếu được phép, sẽ ảnh hưởng tới
những gì process này nhìn thấy khi nó truy cập filesystem sau đó.
Các volume được mount tại [các đường dẫn được chỉ định](#using-subpath) bên trong
filesystem của container.
Với mỗi container được định nghĩa trong một Pod, bạn phải chỉ định một cách độc lập
nơi mount từng volume mà container đó sử dụng.

Volume không thể mount bên trong volume khác (nhưng xem [Sử dụng subPath](#using-subpath)
để biết một cơ chế liên quan). Ngoài ra, một volume không thể chứa hard link trỏ tới
bất kỳ thứ gì trong một volume khác.

## Các loại volume (Types of volumes) {#volume-types}

Kubernetes hỗ trợ một số loại volume.

### configMap

Một [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
cung cấp một cách để tiêm (inject) dữ liệu cấu hình vào các Pod.
Dữ liệu lưu trong một ConfigMap có thể được tham chiếu trong một volume loại
`configMap` rồi được sử dụng bởi các ứng dụng đóng gói container chạy trong Pod.

Khi tham chiếu một ConfigMap, bạn cung cấp tên của ConfigMap trong
volume. Bạn có thể tùy chỉnh đường dẫn sử dụng cho một mục (entry) cụ thể
trong ConfigMap. Cấu hình sau đây cho thấy cách mount
ConfigMap `log-config` vào một Pod có tên `configmap-pod`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox:1.28
      command: ['sh', '-c', 'echo "The app is running!" && tail -f /dev/null']
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level.conf
```

ConfigMap `log-config` được mount dưới dạng một volume, và toàn bộ nội dung lưu trong
mục `log_level` của nó được mount vào Pod tại đường dẫn `/etc/config/log_level.conf`.
Lưu ý rằng đường dẫn này được suy ra từ `mountPath` của volume và `path`
ứng với khóa `log_level`.

> **Ghi chú:**
>
> * Bạn phải [tạo một ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-a-configmap)
>   trước khi có thể sử dụng nó.
>
> * Một ConfigMap luôn được mount ở chế độ `readOnly`.
>
> * Container dùng ConfigMap qua kiểu mount volume [`subPath`](#using-subpath) sẽ không
>   nhận được cập nhật khi ConfigMap thay đổi.
>
> * Dữ liệu văn bản được phơi bày (expose) thành các file theo bảng mã ký tự UTF-8.
>   Với các bảng mã ký tự khác, hãy dùng `binaryData`.

### downwardAPI {#downwardapi}

Một volume `downwardAPI` cung cấp dữ liệu downward API
cho các ứng dụng. Bên trong volume, bạn có thể thấy dữ liệu
được phơi bày dưới dạng các file chỉ đọc ở định dạng văn bản thuần (plain text).

> **Ghi chú:**
> Container dùng downward API qua kiểu mount volume [`subPath`](#using-subpath) sẽ không
> nhận được cập nhật khi giá trị của các trường thay đổi.

Xem [Phơi bày thông tin Pod cho container thông qua file](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)
để tìm hiểu thêm.

### emptyDir {#emptydir}

Với một Pod định nghĩa volume `emptyDir`, volume này được tạo khi Pod được gán vào một node.
Đúng như tên gọi, volume `emptyDir` ban đầu là trống rỗng. Tất cả các container trong Pod đều có thể
đọc và ghi cùng những file trong volume `emptyDir`, dù volume đó có thể được mount tại
cùng một đường dẫn hoặc các đường dẫn khác nhau trong mỗi container. Khi một Pod bị gỡ khỏi node
vì bất kỳ lý do gì, dữ liệu trong `emptyDir` bị xóa vĩnh viễn.

> **Ghi chú:**
> Container bị crash *không* làm Pod bị gỡ khỏi node. Dữ liệu trong volume `emptyDir`
> vẫn an toàn khi container bị crash.

Một số công dụng của `emptyDir`:

* không gian nháp (scratch space), ví dụ cho thuật toán merge sort trên đĩa
* lưu điểm kiểm tra (checkpoint) của một phép tính dài để khôi phục sau crash
* giữ các file mà một container quản lý nội dung (content-manager) tải về trong khi một
  container webserver phục vụ dữ liệu đó

Trường `emptyDir.medium` kiểm soát nơi lưu trữ các volume `emptyDir`. Theo
mặc định, volume `emptyDir` được lưu trên bất kỳ phương tiện nào đứng sau node
như đĩa, SSD, hoặc lưu trữ mạng, tùy vào môi trường của bạn. Nếu bạn đặt
trường `emptyDir.medium` là `"Memory"`, Kubernetes sẽ mount một tmpfs (filesystem
trên RAM) cho bạn thay thế. Mặc dù tmpfs rất nhanh, hãy lưu ý rằng, khác với
đĩa, các file bạn ghi sẽ được tính vào giới hạn bộ nhớ (memory limit) của container đã ghi chúng.

Có thể chỉ định giới hạn kích thước cho phương tiện mặc định, giới hạn này khống chế dung lượng
của volume `emptyDir`. Dung lượng lưu trữ được cấp phát từ
[lưu trữ tạm thời của node (node ephemeral storage)](https://kubernetes.io/docs/concepts/storage/ephemeral-storage/#setting-requests-and-limits-for-local-ephemeral-storage).
Nếu phần lưu trữ đó bị lấp đầy bởi một nguồn khác (ví dụ file log hoặc image overlay),
`emptyDir` có thể hết dung lượng trước khi chạm tới giới hạn này.
Nếu không chỉ định kích thước, các volume dựa trên bộ nhớ (memory-backed) sẽ có kích thước
bằng lượng bộ nhớ cấp phát được (allocatable memory) của node.

> **Thận trọng:**
> Vui lòng xem [tại đây](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#memory-backed-emptydir)
> các điểm cần lưu ý về quản lý tài nguyên khi sử dụng `emptyDir` dựa trên bộ nhớ.

#### Ví dụ cấu hình emptyDir (emptyDir configuration example)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 500Mi
```

#### Ví dụ cấu hình emptyDir dùng bộ nhớ (emptyDir memory configuration example)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 500Mi
      medium: Memory
```

### fc (fibre channel) {#fc}

Loại volume `fc` cho phép mount một volume lưu trữ khối (block storage) fibre channel
có sẵn vào một Pod. Bạn có thể chỉ định một hoặc nhiều target world wide name (WWN)
bằng tham số `targetWWNs` trong cấu hình Volume của bạn. Nếu nhiều WWN được chỉ định,
targetWWNs kỳ vọng rằng các WWN đó đến từ các kết nối đa đường dẫn (multi-path).

> **Ghi chú:**
> Bạn phải cấu hình FC SAN Zoning để cấp phát và che chắn (mask) các LUN (volume) đó
> cho các target WWN từ trước, để các host Kubernetes có thể truy cập chúng.

### gcePersistentDisk (đã lỗi thời — deprecated) {#gcepersistentdisk}

Trong Kubernetes v1.36, mọi thao tác đối với loại `gcePersistentDisk` in-tree
đều được chuyển hướng tới driver CSI `pd.csi.storage.gke.io`.

Driver lưu trữ in-tree `gcePersistentDisk` đã bị đánh dấu lỗi thời (deprecated) trong bản phát hành Kubernetes v1.17
và sau đó bị loại bỏ hoàn toàn trong bản phát hành v1.28.

Dự án Kubernetes khuyến nghị bạn sử dụng driver lưu trữ bên thứ ba
[Google Compute Engine Persistent Disk CSI](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver)
thay thế.

### gitRepo (đã vô hiệu hóa — disabled) {#gitrepo}

> **Cảnh báo:**
>
> Kubernetes v1.36 *không* bao gồm volume driver `gitRepo`.
> Phiên bản cuối cùng còn cung cấp cách sử dụng driver này là Kubernetes
> v1.35, và nó đã bị đánh dấu lỗi thời kể từ bản phát hành minor [v1.11](https://kubernetes.io/releases/1.11).
>
> Để tạo một Pod có repository Git được mount, bạn có thể mount một
> volume [`emptyDir`](#emptydir) vào một [init container](./50-init-containers-vi.md)
> để clone repo bằng Git, rồi mount [EmptyDir](#emptydir) đó vào container của Pod.
>
> ---
>
> Bạn có thể hạn chế việc sử dụng volume `gitRepo` trong cluster của mình bằng
> [các policy](https://kubernetes.io/docs/concepts/policy/), chẳng hạn như
> [ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/).
> Bạn có thể dùng biểu thức Common Expression Language (CEL) sau như một
> phần của policy để từ chối việc sử dụng volume `gitRepo`:
>
> ```cel
> !has(object.spec.volumes) || !object.spec.volumes.exists(v, has(v.gitRepo))
> ```

### hostPath {#hostpath}

Volume `hostPath` mount một file hoặc thư mục từ filesystem của node chủ (host node)
vào Pod của bạn. Đây không phải là thứ hầu hết các Pod cần đến, nhưng nó cung cấp
một lối thoát (escape hatch) mạnh mẽ cho một số ứng dụng.

> **Cảnh báo:**
> Sử dụng loại volume `hostPath` tiềm ẩn nhiều rủi ro bảo mật.
> Nếu có thể tránh dùng volume `hostPath`, bạn nên tránh. Ví dụ,
> hãy định nghĩa một [PersistentVolume `local`](#local) và dùng nó thay thế.
>
> Nếu bạn hạn chế quyền truy cập vào các thư mục cụ thể trên node bằng
> kiểm tra lúc admission (admission-time validation), hạn chế đó chỉ hiệu quả khi bạn
> đồng thời yêu cầu mọi mount của volume `hostPath` đó phải là
> **chỉ đọc (read only)**. Nếu bạn cho phép một Pod không đáng tin cậy mount đọc-ghi
> bất kỳ đường dẫn host nào, các container trong Pod đó có thể phá vỡ (subvert)
> mount đọc-ghi trên host.
>
> ---
>
> Hãy cẩn trọng khi dùng volume `hostPath`, dù chúng được mount chỉ đọc
> hay đọc-ghi, bởi vì:
>
> * Quyền truy cập vào filesystem của host có thể làm lộ các thông tin xác thực (credential) đặc quyền
>   của hệ thống (chẳng hạn của kubelet) hoặc các API đặc quyền
>   (chẳng hạn socket của container runtime), có thể bị lợi dụng để thoát khỏi container
>   (container escape) hoặc tấn công các phần khác của cluster.
> * Các Pod có cấu hình giống hệt nhau (chẳng hạn được tạo từ một PodTemplate) có thể
>   hành xử khác nhau trên các node khác nhau do các file trên các node khác nhau.
> * Việc sử dụng volume `hostPath` không được tính là sử dụng lưu trữ tạm thời (ephemeral storage).
>   Bạn cần tự mình giám sát việc sử dụng đĩa vì việc dùng đĩa `hostPath`
>   quá mức sẽ dẫn tới áp lực đĩa (disk pressure) trên node.

Một số công dụng của `hostPath`:

* chạy một container cần truy cập các thành phần hệ thống ở cấp node
  (chẳng hạn một container chuyển log hệ thống tới một vị trí tập trung,
  truy cập các log đó qua mount chỉ đọc của `/var/log`)
* cung cấp một file cấu hình lưu trên hệ thống host ở chế độ chỉ đọc
  cho một static Pod;
  khác với Pod thông thường, static Pod không thể truy cập ConfigMap

#### Các loại volume `hostPath` (`hostPath` volume types)

Ngoài thuộc tính bắt buộc `path`, bạn có thể tùy chọn chỉ định
`type` cho một volume `hostPath`.

Các giá trị khả dụng cho `type` là:

<!-- chuỗi rỗng được biểu diễn bằng ký tự U+200C ZERO WIDTH NON-JOINER -->

| Giá trị | Hành vi |
|:------|:---------|
| `‌""` | Chuỗi rỗng (mặc định) dành cho tương thích ngược, nghĩa là sẽ không có kiểm tra nào được thực hiện trước khi mount volume `hostPath`. |
| `DirectoryOrCreate` | Nếu không có gì tồn tại tại đường dẫn đã cho, một thư mục trống sẽ được tạo ở đó khi cần, với quyền được đặt là 0755, có cùng group và quyền sở hữu với Kubelet. |
| `Directory` | Một thư mục phải tồn tại tại đường dẫn đã cho. |
| `FileOrCreate` | Nếu không có gì tồn tại tại đường dẫn đã cho, một file trống sẽ được tạo ở đó khi cần, với quyền được đặt là 0644, có cùng group và quyền sở hữu với Kubelet. |
| `File` | Một file phải tồn tại tại đường dẫn đã cho. |
| `Socket` | Một UNIX socket phải tồn tại tại đường dẫn đã cho. |
| `CharDevice` | _(Chỉ với node Linux)_ Một character device phải tồn tại tại đường dẫn đã cho. |
| `BlockDevice` | _(Chỉ với node Linux)_ Một block device phải tồn tại tại đường dẫn đã cho. |

> **Thận trọng:**
> Chế độ `FileOrCreate` **không** tạo thư mục cha của file. Nếu thư mục cha
> của file được mount không tồn tại, Pod sẽ không khởi động được. Để bảo đảm chế độ này hoạt động,
> bạn có thể thử mount thư mục và file một cách riêng rẽ, như trong
> [ví dụ `FileOrCreate`](#hostpath-fileorcreate-example) cho `hostPath`.

Một số file hoặc thư mục được tạo trên host bên dưới có thể chỉ
truy cập được bởi root. Khi đó bạn hoặc phải chạy process của mình dưới quyền root trong một
[privileged container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
hoặc thay đổi quyền của file trên host để có thể đọc từ (hoặc ghi vào) volume `hostPath`.

#### Ví dụ cấu hình hostPath (hostPath configuration example)

##### Linux node

```yaml
---
# Manifest này mount /data/foo trên host thành /foo bên trong
# container duy nhất chạy trong Pod hostpath-example-linux.
#
# Mount vào container ở chế độ chỉ đọc.
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-example-linux
spec:
  os: { name: linux }
  nodeSelector:
    kubernetes.io/os: linux
  containers:
  - name: example-container
    image: registry.k8s.io/test-webserver
    volumeMounts:
    - mountPath: /foo
      name: example-volume
      readOnly: true
  volumes:
  - name: example-volume
    # mount /data/foo, nhưng chỉ khi thư mục đó đã tồn tại
    hostPath:
      path: /data/foo # vị trí thư mục trên host
      type: Directory # trường này là tùy chọn
```

##### Windows node

```yaml
---
# Manifest này mount C:\Data\foo trên host thành C:\foo, bên trong
# container duy nhất chạy trong Pod hostpath-example-windows.
#
# Mount vào container ở chế độ chỉ đọc.
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-example-windows
spec:
  os: { name: windows }
  nodeSelector:
    kubernetes.io/os: windows
  containers:
  - name: example-container
    image: microsoft/windowsservercore:1709
    volumeMounts:
    - name: example-volume
      mountPath: "C:\\foo"
      readOnly: true
  volumes:
    # mount C:\Data\foo từ host, nhưng chỉ khi thư mục đó đã tồn tại
  - name: example-volume
    hostPath:
      path: "C:\\Data\\foo" # vị trí thư mục trên host
      type: Directory       # trường này là tùy chọn
```

#### Ví dụ cấu hình hostPath FileOrCreate (hostPath FileOrCreate configuration example) {#hostpath-fileorcreate-example}

Manifest sau đây định nghĩa một Pod mount `/var/local/aaa`
bên trong container duy nhất của Pod. Nếu node chưa có
đường dẫn `/var/local/aaa`, kubelet sẽ tạo nó
dưới dạng thư mục rồi mount vào Pod.

Nếu `/var/local/aaa` đã tồn tại nhưng không phải là thư mục,
Pod sẽ thất bại. Ngoài ra, kubelet còn cố gắng tạo
một file tên `/var/local/aaa/1.txt` bên trong thư mục đó
(nhìn từ phía host); nếu đã có thứ gì đó tồn tại tại
đường dẫn này và không phải là một file thông thường, Pod sẽ thất bại.

Đây là manifest ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-webserver
spec:
  os: { name: linux }
  nodeSelector:
    kubernetes.io/os: linux
  containers:
  - name: test-webserver
    image: registry.k8s.io/test-webserver:latest
    volumeMounts:
    - mountPath: /var/local/aaa
      name: mydir
    - mountPath: /var/local/aaa/1.txt
      name: myfile
  volumes:
  - name: mydir
    hostPath:
      # Bảo đảm thư mục chứa file được tạo.
      path: /var/local/aaa
      type: DirectoryOrCreate
  - name: myfile
    hostPath:
      path: /var/local/aaa/1.txt
      type: FileOrCreate
```

### image

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Nguồn volume `image` đại diện cho một đối tượng OCI (một container image hoặc
artifact) có sẵn trên máy chủ (host machine) của kubelet.

Một ví dụ về việc sử dụng nguồn volume `image`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: image-volume
spec:
  containers:
  - name: shell
    command: ["sleep", "infinity"]
    image: debian
    volumeMounts:
    - name: volume
      mountPath: /volume
  volumes:
  - name: volume
    image:
      reference: quay.io/crio/artifact:v2
      pullPolicy: IfNotPresent
```

Volume được phân giải (resolve) lúc Pod khởi động, tùy vào giá trị `pullPolicy`
được cung cấp:

`Always`
: kubelet luôn cố gắng pull tham chiếu (reference) này. Nếu việc pull thất bại,
  kubelet đặt Pod thành `Failed`.

`Never`
: kubelet không bao giờ pull tham chiếu và chỉ dùng image hoặc artifact có sẵn cục bộ.
  Pod trở thành `Failed` nếu có bất kỳ layer nào của image chưa có sẵn cục bộ,
  hoặc nếu manifest của image đó chưa được cache.

`IfNotPresent`
: kubelet pull nếu tham chiếu chưa có sẵn trên đĩa. Pod trở thành
  `Failed` nếu tham chiếu chưa có và việc pull thất bại.

Volume được phân giải lại nếu Pod bị xóa và được tạo lại, nghĩa là
nội dung mới từ xa (remote) sẽ trở nên khả dụng khi Pod được tạo lại. Việc phân giải
hoặc pull image thất bại trong lúc Pod khởi động sẽ chặn các container khởi động
và có thể gây thêm độ trễ đáng kể. Các thất bại sẽ được thử lại theo cơ chế backoff
thông thường của volume và sẽ được báo cáo trong reason và message của Pod.

Các loại đối tượng có thể được mount bởi volume này do phần triển khai
container runtime trên máy chủ quyết định. Tối thiểu, chúng phải bao gồm
mọi loại hợp lệ được hỗ trợ bởi trường container image. Đối tượng OCI được
mount vào một thư mục duy nhất (`spec.containers[*].volumeMounts[*].mountPath`)
và sẽ được mount ở chế độ chỉ đọc.

Ngoài ra:

- Mount [`subPath`](#using-subpath) hoặc
  [`subPathExpr`](#using-subpath-expanded-environment)
  cho container (`spec.containers[*].volumeMounts[*].subPath`, `spec.containers[*].volumeMounts[*].subPathExpr`)
  chỉ được hỗ trợ từ Kubernetes v1.33.
- Trường `spec.securityContext.fsGroupChangePolicy` không có tác dụng đối với
  loại volume này.
- [Admission Controller `AlwaysPullImages`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages)
  cũng hoạt động với nguồn volume này giống như với các container image.

Các trường sau khả dụng cho loại `image`:

`reference`
: Tham chiếu artifact sẽ được sử dụng. Ví dụ, bạn có thể chỉ định
  `registry.k8s.io/conformance:v1.36.0` để nạp các
  file từ image kiểm thử conformance của Kubernetes. Hành xử theo cùng cách như
  `pod.spec.containers[*].image`. Pull secret sẽ được tập hợp theo cùng cách
  như với container image, bằng cách tra cứu thông tin xác thực của node, image pull secret
  của service account, và image pull secret trong spec của Pod. Trường này là tùy chọn để cho phép
  các tầng quản lý cấu hình cấp cao hơn đặt mặc định hoặc ghi đè container image trong
  các workload controller như Deployment và StatefulSet.
  [Thêm thông tin về container image](./40-images-vi.md).

`pullPolicy`
: Chính sách pull các đối tượng OCI. Các giá trị có thể là: `Always`, `Never`, hoặc
  `IfNotPresent`. Mặc định là `Always` nếu tag `:latest` được chỉ định, ngược lại
  là `IfNotPresent`.

Xem ví dụ [_Sử dụng Image Volume với một Pod_](https://kubernetes.io/docs/tasks/configure-pod-container/image-volumes)
để biết thêm chi tiết về cách sử dụng nguồn volume này.

#### Trạng thái Pod và volume `image` (Pod status and `image` volumes) {#image-volume-pod-status}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [alpha]`

Nếu [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`ImageVolumeWithDigest` được bật trong cluster của bạn,
thì mỗi khi bạn chỉ định một volume `image` cho Pod,
kubelet sẽ cập nhật trạng thái của Pod để ghi lại _digest_
của container image đang được sử dụng làm nguồn volume.

Đây là một ví dụ giản lược về một Pod đang chạy, biểu diễn dưới dạng YAML, bao gồm cả phần cập nhật trạng thái.
Lưu ý trường `ImageRef` mới dưới `volumeMounts` trong trạng thái container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
spec:
  containers:
  - name: shell
    command: ["sleep", "infinity"]
    image: docker.io/library/debian:12
    volumeMounts:
    - name: artifact
      mountPath: /data
  volumes:
  - name: artifact
    image:
      reference: quay.io/crio/artifact:v2
      pullPolicy: IfNotPresent
status:
  containerStatuses:
  - containerID: containerd://examplecontainerid1234567890abcdef
    image: docker.io/library/debian:12
    imageID: docker-pullable://docker.io/library/debian@sha256:3f1d6c17773a45c97bd8f158d665c9709d7b29ed7917ac934086ad96f92e4510
    volumeMounts:
    - name: artifact
      mountPath: /data
      readOnly: true
      imageRef: quay.io/crio/artifact@sha256:abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890
```

### iscsi

Volume `iscsi` cho phép mount một volume iSCSI (SCSI over IP) có sẵn
vào Pod của bạn. Khác với `emptyDir` vốn bị xóa khi Pod bị gỡ bỏ,
nội dung của volume `iscsi` được bảo toàn và volume chỉ đơn thuần
bị unmount. Điều này có nghĩa là một volume iscsi có thể được nạp sẵn dữ liệu,
và dữ liệu đó có thể được chia sẻ giữa các Pod.

> **Ghi chú:**
> Bạn phải có server iSCSI của riêng mình đang chạy với volume đã được tạo trước khi có thể sử dụng nó.

Một tính năng của iSCSI là nó có thể được mount ở chế độ chỉ đọc bởi nhiều consumer
đồng thời. Nghĩa là bạn có thể nạp sẵn dữ liệu (dataset) vào volume
rồi phục vụ song song từ bao nhiêu Pod tùy nhu cầu. Đáng tiếc,
volume iSCSI chỉ có thể được mount bởi một consumer duy nhất ở chế độ đọc-ghi.
Không cho phép nhiều bên ghi đồng thời.

### local

Volume `local` đại diện cho một thiết bị lưu trữ cục bộ đã được mount như một đĩa,
phân vùng (partition) hoặc thư mục.

Volume local chỉ có thể được dùng như một PersistentVolume được tạo tĩnh. Không hỗ trợ
cấp phát động (dynamic provisioning).

So với volume `hostPath`, volume `local` được dùng theo cách bền vững và
di động (portable) mà không cần lập lịch Pod vào node một cách thủ công. Hệ thống nhận biết
các ràng buộc node của volume bằng cách nhìn vào node affinity trên PersistentVolume.

Tuy nhiên, volume `local` phụ thuộc vào tính khả dụng của node
bên dưới và không phù hợp với mọi ứng dụng. Nếu một node trở nên không khỏe mạnh (unhealthy),
volume `local` sẽ trở nên không truy cập được đối với Pod. Pod dùng volume này
không thể chạy. Các ứng dụng dùng volume `local` phải có khả năng chịu được
mức khả dụng suy giảm này, cũng như nguy cơ mất dữ liệu, tùy vào
đặc tính độ bền (durability) của đĩa bên dưới.

Ví dụ sau cho thấy một PersistentVolume dùng volume `local` và
`nodeAffinity`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node
```

Bạn phải đặt `nodeAffinity` cho PersistentVolume khi dùng volume `local`.
Scheduler của Kubernetes dùng `nodeAffinity` của PersistentVolume để lập lịch
các Pod này vào đúng node.

`volumeMode` của PersistentVolume có thể được đặt là "Block" (thay cho giá trị
mặc định "Filesystem") để phơi bày volume local như một thiết bị khối thô (raw block device).

Khi dùng volume local, khuyến nghị tạo một StorageClass với
`volumeBindingMode` đặt là `WaitForFirstConsumer`. Để biết thêm chi tiết, xem
ví dụ về [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/#local) local.
Việc trì hoãn gắn kết (binding) volume bảo đảm rằng quyết định gắn kết PersistentVolumeClaim
cũng sẽ được đánh giá cùng với mọi ràng buộc node khác mà Pod có thể có,
chẳng hạn yêu cầu tài nguyên node, node selector, Pod affinity, và Pod anti-affinity.

Một trình cấp phát tĩnh bên ngoài (external static provisioner) có thể được chạy riêng để
quản lý tốt hơn vòng đời của volume local. Lưu ý rằng provisioner này chưa hỗ trợ
cấp phát động. Để xem ví dụ về cách chạy một external local provisioner, xem
[hướng dẫn sử dụng local volume provisioner](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner).

> **Ghi chú:**
> PersistentVolume local yêu cầu người dùng dọn dẹp và xóa thủ công
> nếu external static provisioner không được dùng để quản lý vòng đời
> của volume.

### nfs

Volume `nfs` cho phép mount một NFS (Network File System) share có sẵn
vào một Pod. Khác với `emptyDir` vốn bị xóa khi Pod bị
gỡ bỏ, nội dung của volume `nfs` được bảo toàn và volume chỉ đơn thuần
bị unmount. Điều này có nghĩa là một volume NFS có thể được nạp sẵn dữ liệu, và
dữ liệu đó có thể được chia sẻ giữa các Pod. NFS có thể được mount bởi nhiều
bên ghi (writer) đồng thời.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /my-nfs-data
      name: test-volume
  volumes:
  - name: test-volume
    nfs:
      server: my-nfs-server.example.com
      path: /my-nfs-volume
      readOnly: true
```

> **Ghi chú:**
> Bạn phải có server NFS của riêng mình đang chạy với share đã được export trước khi có thể sử dụng nó.
>
> Cũng lưu ý rằng bạn không thể chỉ định các tùy chọn mount NFS trong spec của Pod. Bạn có thể đặt tùy chọn mount
> ở phía server hoặc dùng [/etc/nfsmount.conf](https://man7.org/linux/man-pages/man5/nfsmount.conf.5.html).
> Bạn cũng có thể mount volume NFS thông qua PersistentVolume — cách này cho phép bạn đặt tùy chọn mount.

### persistentVolumeClaim {#persistentvolumeclaim}

Volume `persistentVolumeClaim` được dùng để mount một
[PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) vào một Pod. PersistentVolumeClaim
là cách để người dùng "yêu cầu" (claim) lưu trữ bền vững (chẳng hạn một volume iSCSI)
mà không cần biết chi tiết về môi trường cloud cụ thể.

Xem thông tin về [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) để biết thêm
chi tiết.

### portworxVolume (đã lỗi thời — deprecated) {#portworxvolume}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [deprecated]`

`portworxVolume` là một tầng lưu trữ khối co giãn (elastic block storage) chạy siêu hội tụ (hyperconverged) với
Kubernetes. [Portworx](https://portworx.com/use-case/kubernetes-storage/) lấy dấu vân tay (fingerprint) lưu trữ
trong một server, phân tầng dựa trên năng lực, và gom dung lượng từ nhiều server.
Portworx chạy in-guest trong máy ảo hoặc trên các node Linux vật lý (bare metal).

`portworxVolume` có thể được tạo động thông qua Kubernetes, hoặc cũng có thể
được cấp phát sẵn (pre-provisioned) và tham chiếu bên trong một Pod.
Đây là một Pod ví dụ tham chiếu một volume Portworx đã được cấp phát sẵn:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-portworx-volume-pod
spec:
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /mnt
      name: pxvol
  volumes:
  - name: pxvol
    # Volume Portworx này phải tồn tại từ trước.
    portworxVolume:
      volumeID: "pxvol"
      fsType: "<fs-type>"
```

> **Ghi chú:**
> Hãy chắc chắn bạn đã có sẵn một PortworxVolume tên là `pxvol`
> trước khi dùng nó trong Pod.

#### Di trú CSI cho Portworx (Portworx CSI migration)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]`

Trong Kubernetes v1.36, mọi thao tác đối với volume Portworx in-tree
đều được chuyển hướng mặc định tới Driver Container Storage Interface (CSI) `pxd.portworx.com`.
[Driver CSI Portworx](https://docs.portworx.com/portworx-enterprise/operations/operate-kubernetes/storage-operations/csi)
phải được cài đặt trên cluster.

### projected

Một projected volume ánh xạ nhiều nguồn volume có sẵn vào cùng một
thư mục. Để biết thêm chi tiết, xem [projected volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/).

### secret

Volume `secret` được dùng để truyền thông tin nhạy cảm, chẳng hạn mật khẩu, cho
các Pod. Bạn có thể lưu secret trong Kubernetes API và mount chúng dưới dạng file để
các Pod sử dụng mà không cần gắn kết trực tiếp với Kubernetes. Volume `secret` được
lưu trên tmpfs (một filesystem trên RAM) nên chúng không bao giờ được ghi xuống
lưu trữ bất biến (non-volatile).

> **Ghi chú:**
>
> * Bạn phải tạo một Secret trong Kubernetes API trước khi có thể sử dụng nó.
>
> * Một Secret luôn được mount ở chế độ `readOnly`.
>
> * Container dùng Secret qua kiểu mount volume [`subPath`](#using-subpath) sẽ không
>   nhận được cập nhật của Secret.

Để biết thêm chi tiết, xem [Cấu hình Secret](https://kubernetes.io/docs/concepts/configuration/secret/).

## Sử dụng subPath (Using subPath) {#using-subpath}

Đôi khi việc chia sẻ một volume cho nhiều mục đích sử dụng trong cùng một Pod là hữu ích.
Thuộc tính `volumeMounts[*].subPath` chỉ định một đường dẫn con (sub-path) bên trong volume
được tham chiếu thay vì gốc (root) của nó.

Ví dụ sau cho thấy cách cấu hình một Pod với LAMP stack (Linux, Apache, MySQL, PHP)
dùng một volume chia sẻ duy nhất. Cấu hình `subPath` mẫu này không được khuyến nghị
dùng cho môi trường production.

Code và tài nguyên (asset) của ứng dụng PHP ánh xạ vào thư mục `html` của volume và
cơ sở dữ liệu MySQL được lưu trong thư mục `mysql` của volume. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

### Sử dụng subPath với biến môi trường được mở rộng (Using subPath with expanded environment variables) {#using-subpath-expanded-environment}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.17 [stable]`

Dùng trường `subPathExpr` để xây dựng tên thư mục `subPath` từ
các biến môi trường downward API.
Thuộc tính `subPath` và `subPathExpr` loại trừ lẫn nhau.

Trong ví dụ này, một `Pod` dùng `subPathExpr` để tạo một thư mục `pod1` bên trong
volume `hostPath` `/var/log/pods`.
Volume `hostPath` lấy tên `Pod` từ `downwardAPI`.
Thư mục host `/var/log/pods/pod1` được mount tại `/logs` trong container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: container1
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    image: busybox:1.28
    command: [ "sh", "-c", "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt" ]
    volumeMounts:
    - name: workdir1
      mountPath: /logs
      # Việc mở rộng biến dùng ngoặc tròn (không phải ngoặc nhọn).
      subPathExpr: $(POD_NAME)
  restartPolicy: Never
  volumes:
  - name: workdir1
    hostPath:
      path: /var/log/pods
```

## Tài nguyên (Resources)

Phương tiện lưu trữ (chẳng hạn Disk hoặc SSD) của một volume `emptyDir` được quyết định bởi
phương tiện của filesystem chứa thư mục gốc của kubelet (thường là
`/var/lib/kubelet`). Không có giới hạn về lượng không gian mà một volume `emptyDir` hoặc
`hostPath` có thể sử dụng, và không có sự cô lập (isolation) giữa các container hoặc
giữa các Pod.

Để tìm hiểu về việc yêu cầu không gian bằng một đặc tả tài nguyên (resource specification), xem
[cách quản lý tài nguyên](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/).

## Các plugin volume out-of-tree (Out-of-tree volume plugins)

Các plugin volume out-of-tree bao gồm
Container Storage Interface (CSI), và cả
FlexVolume (đã lỗi thời). Các plugin này cho phép các nhà cung cấp lưu trữ tạo plugin lưu trữ
tùy chỉnh mà không cần thêm mã nguồn plugin của họ vào repository Kubernetes.

Trước đây, mọi plugin volume đều là "in-tree". Các plugin "in-tree" được build, liên kết (link), biên dịch,
và phân phối cùng các binary lõi của Kubernetes. Điều này có nghĩa là việc thêm một hệ thống lưu trữ mới vào
Kubernetes (một plugin volume) đòi hỏi phải đưa code vào repository code lõi của Kubernetes.

Cả CSI và FlexVolume đều cho phép các plugin volume được phát triển độc lập với
code base của Kubernetes, và được triển khai (cài đặt) trên các cluster Kubernetes như
những phần mở rộng (extension).

Với các nhà cung cấp lưu trữ muốn tạo một plugin volume out-of-tree, vui lòng tham khảo
[FAQ về plugin volume](https://github.com/kubernetes/community/blob/main/sig-storage/volume-plugin-faq.md).

### csi

[Container Storage Interface](https://github.com/container-storage-interface/spec/blob/master/spec.md)
(CSI) định nghĩa một giao diện chuẩn cho các hệ thống điều phối container (như
Kubernetes) để phơi bày các hệ thống lưu trữ bất kỳ cho các workload container của chúng.

Vui lòng đọc [đề xuất thiết kế CSI](https://git.k8s.io/design-proposals-archive/storage/container-storage-interface.md)
để biết thêm thông tin.

> **Ghi chú:**
> Hỗ trợ cho các phiên bản CSI spec 0.2 và 0.3 đã bị đánh dấu lỗi thời trong Kubernetes
> v1.13 và sẽ bị loại bỏ trong một bản phát hành tương lai.

> **Ghi chú:**
> Các driver CSI có thể không tương thích với mọi bản phát hành Kubernetes.
> Vui lòng xem tài liệu của từng driver CSI cụ thể để biết các bước triển khai
> được hỗ trợ cho mỗi bản phát hành Kubernetes và ma trận tương thích (compatibility matrix).

Khi một driver volume tương thích CSI được triển khai trên cluster Kubernetes, người dùng
có thể dùng loại volume `csi` để gắn (attach) hoặc mount các volume mà
driver CSI phơi bày.

Một volume `csi` có thể được dùng trong Pod theo ba cách khác nhau:

* thông qua tham chiếu tới một [PersistentVolumeClaim](#persistentvolumeclaim)
* với một [generic ephemeral volume](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/#generic-ephemeral-volumes)
* với một [CSI ephemeral volume](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/#csi-ephemeral-volumes)
  nếu driver hỗ trợ

Các trường sau khả dụng cho quản trị viên lưu trữ để cấu hình một CSI
persistent volume:

* `driver`: Một giá trị chuỗi chỉ định tên của driver volume sẽ sử dụng.
  Giá trị này phải khớp với giá trị được trả về trong `GetPluginInfoResponse`
  bởi driver CSI như định nghĩa trong
  [CSI spec](https://github.com/container-storage-interface/spec/blob/master/spec.md#getplugininfo).
  Nó được Kubernetes dùng để xác định driver CSI nào cần gọi tới, và được
  các thành phần của driver CSI dùng để xác định các đối tượng PV nào thuộc về driver CSI đó.
* `volumeHandle`: Một giá trị chuỗi định danh duy nhất cho volume. Giá trị này
  phải khớp với giá trị được trả về trong trường `volume.id` của
  `CreateVolumeResponse` bởi driver CSI như định nghĩa trong
  [CSI spec](https://github.com/container-storage-interface/spec/blob/master/spec.md#createvolume).
  Giá trị này được truyền dưới dạng `volume_id` trong mọi lời gọi tới driver volume CSI khi
  tham chiếu volume.
* `readOnly`: Một giá trị boolean tùy chọn cho biết volume có được
  "ControllerPublished" (gắn vào) ở chế độ chỉ đọc hay không. Mặc định là false. Giá trị này được truyền
  cho driver CSI qua trường `readonly` trong `ControllerPublishVolumeRequest`.
* `fsType`: Nếu `VolumeMode` của PV là `Filesystem` thì trường này có thể được dùng
  để chỉ định filesystem sẽ dùng để mount volume. Nếu
  volume chưa được định dạng (format) và việc định dạng được hỗ trợ, giá trị này sẽ được
  dùng để định dạng volume.
  Giá trị này được truyền cho driver CSI qua trường `VolumeCapability` của
  `ControllerPublishVolumeRequest`, `NodeStageVolumeRequest`, và
  `NodePublishVolumeRequest`.
* `volumeAttributes`: Một map từ chuỗi sang chuỗi chỉ định các thuộc tính tĩnh
  của volume. Map này phải khớp với map được trả về trong trường
  `volume.attributes` của `CreateVolumeResponse` bởi driver CSI như
  định nghĩa trong [CSI spec](https://github.com/container-storage-interface/spec/blob/master/spec.md#createvolume).
  Map này được truyền cho driver CSI qua trường `volume_context` trong
  `ControllerPublishVolumeRequest`, `NodeStageVolumeRequest`, và
  `NodePublishVolumeRequest`.
* `controllerPublishSecretRef`: Tham chiếu tới đối tượng secret chứa
  thông tin nhạy cảm cần truyền cho driver CSI để hoàn tất các lời gọi CSI
  `ControllerPublishVolume` và `ControllerUnpublishVolume`. Trường này là
  tùy chọn, và có thể để trống nếu không cần secret. Nếu Secret
  chứa nhiều hơn một secret, tất cả secret đều được truyền.
* `nodeExpandSecretRef`: Tham chiếu tới secret chứa thông tin nhạy cảm
  cần truyền cho driver CSI để hoàn tất lời gọi CSI
  `NodeExpandVolume`. Trường này là tùy chọn và có thể để trống nếu không
  cần secret. Nếu đối tượng chứa nhiều hơn một secret, tất cả
  secret đều được truyền. Khi bạn đã cấu hình dữ liệu secret cho việc mở rộng volume
  khởi phát từ node (node-initiated), kubelet truyền dữ liệu đó qua lời gọi `NodeExpandVolume()`
  tới driver CSI. Mọi phiên bản Kubernetes được hỗ trợ đều cung cấp trường
  `nodeExpandSecretRef`, và có sẵn nó theo mặc định. Các bản phát hành Kubernetes
  trước v1.25 không có hỗ trợ này.
* Bật [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates-removed/)
  tên là `CSINodeExpandSecret` cho từng kube-apiserver và cho kubelet trên mọi
  node. Kể từ phiên bản Kubernetes 1.27, tính năng này đã được bật mặc định
  và không cần bật feature gate một cách tường minh.
  Bạn cũng phải dùng một driver CSI hỗ trợ hoặc yêu cầu dữ liệu secret trong
  các thao tác thay đổi kích thước lưu trữ khởi phát từ node.
* `nodePublishSecretRef`: Tham chiếu tới đối tượng secret chứa
  thông tin nhạy cảm cần truyền cho driver CSI để hoàn tất lời gọi CSI
  `NodePublishVolume`. Trường này là tùy chọn và có thể để trống nếu không
  cần secret. Nếu đối tượng secret chứa nhiều hơn một secret, tất cả
  secret đều được truyền.
* `nodeStageSecretRef`: Tham chiếu tới đối tượng secret chứa
  thông tin nhạy cảm cần truyền cho driver CSI để hoàn tất lời gọi CSI
  `NodeStageVolume`. Trường này là tùy chọn và có thể để trống nếu không cần
  secret. Nếu Secret chứa nhiều hơn một secret, tất cả secret
  đều được truyền.

#### Hỗ trợ raw block volume cho CSI (CSI raw block volume support)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.18 [stable]`

Các nhà cung cấp có driver CSI bên ngoài có thể triển khai hỗ trợ raw block volume
trong các workload Kubernetes.

Bạn có thể thiết lập
[PersistentVolume/PersistentVolumeClaim với hỗ trợ raw block volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#raw-block-volume-support)
như bình thường, không cần bất kỳ thay đổi nào riêng cho CSI.

#### CSI ephemeral volume (CSI ephemeral volumes)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

Bạn có thể cấu hình trực tiếp các volume CSI ngay trong đặc tả
Pod. Các volume được chỉ định theo cách này là tạm thời và không
tồn tại qua các lần khởi động lại Pod. Xem
[Ephemeral Volumes](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/#csi-ephemeral-volumes)
để biết thêm thông tin.

Để biết thêm thông tin về cách phát triển một driver CSI, tham khảo
[tài liệu kubernetes-csi](https://kubernetes-csi.github.io/docs/)

#### Windows CSI proxy

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.22 [stable]`

Các plugin node CSI cần thực hiện nhiều thao tác
đặc quyền như quét thiết bị đĩa và mount filesystem. Các thao tác này
khác nhau trên mỗi hệ điều hành host. Với các node worker Linux, các plugin node CSI
đóng gói container thường được triển khai dưới dạng privileged container. Với các node worker Windows,
các thao tác đặc quyền cho plugin node CSI đóng gói container được hỗ trợ thông qua
[csi-proxy](https://github.com/kubernetes-csi/csi-proxy), một binary độc lập
do cộng đồng quản lý, cần được cài đặt sẵn trên mỗi node Windows.

Để biết thêm chi tiết, tham khảo hướng dẫn triển khai của plugin CSI mà bạn muốn triển khai.

#### Di trú từ plugin in-tree sang driver CSI (Migrating to CSI drivers from in-tree plugins)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

Tính năng `CSIMigration` điều hướng các thao tác nhắm vào các plugin in-tree
hiện có sang các plugin CSI tương ứng (được kỳ vọng là đã được cài đặt và cấu hình).
Nhờ đó, người vận hành không phải thực hiện bất kỳ
thay đổi cấu hình nào đối với các Storage Class, PersistentVolume hoặc PersistentVolumeClaim
hiện có (đang tham chiếu tới plugin in-tree) khi chuyển sang một driver CSI thay thế plugin in-tree.

> **Ghi chú:**
> Các PV hiện có được tạo bởi một plugin volume in-tree vẫn có thể được dùng trong tương lai mà không cần thay đổi
> cấu hình nào, kể cả sau khi việc di trú sang CSI hoàn tất cho loại volume đó, và kể cả sau khi bạn nâng cấp lên
> một phiên bản Kubernetes không còn tích hợp sẵn hỗ trợ cho loại lưu trữ đó.
>
> Là một phần của việc di trú đó, bạn — hoặc một quản trị viên cluster khác — **phải** cài đặt và cấu hình
> driver CSI phù hợp cho loại lưu trữ đó. Phần lõi của Kubernetes không cài đặt phần mềm đó cho bạn.
>
> ---
>
> Sau khi di trú, bạn cũng có thể định nghĩa các PVC và PV mới tham chiếu tới các tích hợp
> lưu trữ cũ tích hợp sẵn (built-in).
> Miễn là bạn đã cài đặt và cấu hình driver CSI phù hợp, việc tạo PV vẫn tiếp tục
> hoạt động, kể cả với các volume hoàn toàn mới. Việc quản lý lưu trữ thực tế bây giờ diễn ra thông qua
> driver CSI.

Các thao tác và tính năng được hỗ trợ bao gồm:
cấp phát/xóa (provisioning/delete), gắn/tháo (attach/detach), mount/unmount, và thay đổi kích thước (resizing) volume.

Các plugin in-tree hỗ trợ `CSIMigration` và đã có driver CSI tương ứng được triển khai
được liệt kê trong [Các loại volume](#volume-types).

### flexVolume (đã lỗi thời — deprecated) {#flexvolume}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [deprecated]`

FlexVolume là một giao diện plugin out-of-tree dùng mô hình dựa trên exec để giao tiếp
với các driver lưu trữ. Các binary driver FlexVolume phải được cài đặt tại một đường dẫn
plugin volume định nghĩa trước trên mỗi node, và trong một số trường hợp, cả trên các node control plane.

Pod tương tác với các driver FlexVolume thông qua plugin volume in-tree `flexVolume`.

Các [plugin](https://github.com/Microsoft/K8s-Storage-Plugins/tree/master/flexvolume/windows) FlexVolume sau,
được triển khai dưới dạng các script PowerShell trên host, hỗ trợ node Windows:

* [SMB](https://github.com/microsoft/K8s-Storage-Plugins/tree/master/flexvolume/windows/plugins/microsoft.com~smb.cmd)
* [iSCSI](https://github.com/microsoft/K8s-Storage-Plugins/tree/master/flexvolume/windows/plugins/microsoft.com~iscsi.cmd)

> **Ghi chú:**
> FlexVolume đã lỗi thời. Sử dụng một driver CSI out-of-tree là cách được khuyến nghị để tích hợp lưu trữ bên ngoài với Kubernetes.
>
> Người bảo trì driver FlexVolume nên triển khai một Driver CSI và giúp người dùng các driver FlexVolume di trú sang CSI.
> Người dùng FlexVolume nên chuyển workload của họ sang dùng Driver CSI tương đương.

## Lan truyền mount (Mount propagation)

> **Thận trọng:**
> Lan truyền mount (mount propagation) là một tính năng cấp thấp không hoạt động nhất quán trên mọi
> loại volume. Dự án Kubernetes khuyến nghị chỉ dùng lan truyền mount với volume `hostPath`
> hoặc `emptyDir` dựa trên bộ nhớ. Xem
> [issue #95049 của Kubernetes](https://github.com/kubernetes/kubernetes/issues/95049)
> để biết thêm bối cảnh.

Lan truyền mount cho phép chia sẻ các volume được mount bởi một container cho
các container khác trong cùng Pod, hoặc thậm chí cho các Pod khác trên cùng node.

Lan truyền mount của một volume được kiểm soát bởi trường `mountPropagation`
trong `containers[*].volumeMounts`. Các giá trị của nó là:

* `None` - Mount volume này sẽ không nhận bất kỳ mount tiếp theo nào
  được host mount vào volume này hoặc bất kỳ thư mục con nào của nó.
  Tương tự, các mount do container tạo ra sẽ không hiển thị trên
  host. Đây là chế độ mặc định.

  Chế độ này tương đương với lan truyền mount `rprivate` như mô tả trong
  [`mount(8)`](https://man7.org/linux/man-pages/man8/mount.8.html)

  Tuy nhiên, CRI runtime có thể chọn lan truyền mount `rslave` (tức là
  `HostToContainer`) khi lan truyền `rprivate` không áp dụng được.
  cri-dockerd (Docker) được biết là chọn lan truyền mount `rslave` khi
  nguồn mount chứa thư mục gốc của Docker daemon (`/var/lib/docker`).

* `HostToContainer` - Mount volume này sẽ nhận tất cả các mount tiếp theo
  được mount vào volume này hoặc bất kỳ thư mục con nào của nó.

  Nói cách khác, nếu host mount bất cứ thứ gì bên trong mount volume này,
  container sẽ thấy nó được mount ở đó.

  Tương tự, nếu bất kỳ Pod nào có lan truyền mount `Bidirectional` tới cùng
  volume đó mount bất cứ thứ gì vào đó, container với lan truyền mount
  `HostToContainer` sẽ thấy nó.

  Chế độ này tương đương với lan truyền mount `rslave` như mô tả trong
  [`mount(8)`](https://man7.org/linux/man-pages/man8/mount.8.html)

* `Bidirectional` - Mount volume này hành xử giống như mount `HostToContainer`.
  Ngoài ra, tất cả các mount volume do container tạo ra sẽ được lan truyền
  ngược lại host và tới tất cả các container của mọi Pod dùng cùng volume đó.

  Trường hợp sử dụng điển hình cho chế độ này là một Pod với driver FlexVolume hoặc CSI, hoặc
  một Pod cần mount thứ gì đó trên host bằng volume `hostPath`.

  Chế độ này tương đương với lan truyền mount `rshared` như mô tả trong
  [`mount(8)`](https://man7.org/linux/man-pages/man8/mount.8.html)

  > **Cảnh báo:**
  > Lan truyền mount `Bidirectional` có thể nguy hiểm. Nó có thể làm hỏng
  > hệ điều hành host, và do đó chỉ được phép trong các privileged
  > container. Rất khuyến khích bạn nắm vững hành vi của Linux kernel.
  > Ngoài ra, mọi mount volume do các container trong Pod tạo ra phải được hủy
  > (unmount) bởi chính các container đó khi kết thúc.

## Mount chỉ đọc (Read-only mounts)

Một mount có thể được đặt thành chỉ đọc bằng cách đặt trường
`.spec.containers[*].volumeMounts[*].readOnly` thành `true`.
Điều này không làm bản thân volume trở thành chỉ đọc, nhưng container cụ thể đó sẽ
không thể ghi vào nó.
Các container khác trong Pod có thể mount cùng volume đó ở chế độ đọc-ghi.

Trên Linux, các mount chỉ đọc mặc định không phải là chỉ đọc một cách đệ quy (recursively read-only).
Ví dụ, xét một Pod mount `/mnt` của host như một volume `hostPath`. Nếu
có một filesystem khác được mount đọc-ghi tại `/mnt/<SUBMOUNT>` (chẳng hạn tmpfs,
NFS, hoặc USB storage), volume được mount vào (các) container cũng sẽ có
`/mnt/<SUBMOUNT>` ghi được, ngay cả khi bản thân mount được chỉ định là chỉ đọc.

### Mount chỉ đọc đệ quy (Recursive read-only mounts)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]`

Mount chỉ đọc đệ quy có thể được bật bằng cách đặt
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `RecursiveReadOnlyMounts`
cho kubelet và kube-apiserver, và đặt trường `.spec.containers[*].volumeMounts[*].recursiveReadOnly`
cho một Pod.

Các giá trị được phép là:

* `Disabled` (mặc định): không có tác dụng gì.

* `Enabled`: làm cho mount trở thành chỉ đọc một cách đệ quy.
  Cần tất cả các yêu cầu sau được thỏa mãn:

  * `readOnly` được đặt là `true`
  * `mountPropagation` không được đặt, hoặc được đặt là `None`
  * Host chạy Linux kernel v5.12 trở lên
  * Container runtime [cấp CRI](./44-cri-vi.md) hỗ trợ mount chỉ đọc đệ quy
  * Container runtime cấp OCI hỗ trợ mount chỉ đọc đệ quy.

  Nó sẽ thất bại nếu bất kỳ điều kiện nào trong số này không đúng.

* `IfPossible`: cố gắng áp dụng `Enabled`, và quay về `Disabled`
  nếu tính năng không được hỗ trợ bởi kernel hoặc runtime class.

Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rro
spec:
  volumes:
    - name: mnt
      hostPath:
        # tmpfs được mount tại /mnt/tmpfs
        path: /mnt
  containers:
    - name: busybox
      image: busybox
      args: ["sleep", "infinity"]
      volumeMounts:
        # /mnt-rro/tmpfs không ghi được
        - name: mnt
          mountPath: /mnt-rro
          readOnly: true
          mountPropagation: None
          recursiveReadOnly: Enabled
        # /mnt-ro/tmpfs ghi được
        - name: mnt
          mountPath: /mnt-ro
          readOnly: true
        # /mnt-rw/tmpfs ghi được
        - name: mnt
          mountPath: /mnt-rw
```

Khi thuộc tính này được kubelet và kube-apiserver nhận biết,
trường `.status.containerStatuses[*].volumeMounts[*].recursiveReadOnly` sẽ được đặt thành
`Enabled` hoặc `Disabled`.

#### Các phần triển khai (Implementations) {#implementations-rro}

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

Các container runtime sau được biết là hỗ trợ mount chỉ đọc đệ quy.

Cấp CRI:

- [containerd](https://containerd.io/), từ v2.0
- [CRI-O](https://cri-o.io/), từ v1.30

Cấp OCI:

- [runc](https://runc.io/), từ v1.1
- [crun](https://github.com/containers/crun), từ v1.8.6

## Tiếp theo (What's next)

Xem ví dụ về [triển khai WordPress và MySQL với Persistent Volumes](https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Một container trong Pod bị crash và kubelet khởi động lại nó. Dữ liệu trong `emptyDir` còn
   không? Nếu bạn `kubectl delete pod` rồi Deployment tạo Pod mới thì sao?
2. Bạn đặt `emptyDir.medium: "Memory"` và quên đặt `sizeLimit`. Hai hệ quả nào xảy ra?
3. `hostPath` và `local` cùng lấy đĩa của node. Vì sao bài khuyên tránh `hostPath` và dùng
   `local` thay thế?
4. Cluster lab của bạn (`k8s-master` + 2 worker) **chưa có StorageClass và chưa có provisioner**.
   Trong các loại volume bài này liệt kê, những loại nào bạn mount được ngay hôm nay mà không
   cần cài thêm gì, và vì sao `persistentVolumeClaim` chưa dùng được?
5. Bạn mount một ConfigMap bằng `subPath` để chỉ lấy đúng một file. Điều gì sẽ không còn hoạt
   động nữa?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Crash container: **dữ liệu vẫn còn**. Bài nói rõ container bị crash *không* làm Pod bị gỡ
   khỏi node, nên volume `emptyDir` vẫn an toàn. Xóa Pod: **dữ liệu mất vĩnh viễn**, vì
   `emptyDir` được tạo khi Pod được gán vào node và bị xóa khi Pod bị gỡ khỏi node. Pod mới
   nhận một `emptyDir` rỗng, thậm chí có thể trên node khác.
2. Thứ nhất, volume nằm trên **tmpfs**, nên mọi file bạn ghi vào đó **được tính vào memory
   limit của chính container đã ghi** — rất dễ khiến container bị giết vì hết bộ nhớ chứ không
   phải vì hết đĩa. Thứ hai, do không đặt `sizeLimit`, volume dựa trên bộ nhớ sẽ có **kích
   thước bằng allocatable memory của node**, tức gần như không có trần.
3. Vì `hostPath` mở filesystem của host cho Pod: nó có thể **làm lộ credential đặc quyền của
   hệ thống (chẳng hạn của kubelet) hoặc các API đặc quyền như socket của container runtime**,
   và từ đó bị lợi dụng để thoát khỏi container. Thêm hai điểm nữa: Pod cùng một PodTemplate
   hành xử khác nhau trên các node khác nhau vì file trên mỗi node khác nhau, và dung lượng
   `hostPath` chiếm **không được tính là ephemeral storage** nên bạn phải tự giám sát. `local`
   thì được dùng qua PersistentVolume và hệ thống tự biết ràng buộc node qua `nodeAffinity`,
   nên bền vững và di động mà không phải gán Pod vào node bằng tay.
4. Dùng được ngay: **`emptyDir`, `configMap`, `secret`, `downwardAPI`, `projected`** — chúng
   do kubelet dựng tại chỗ từ dữ liệu đã có trong API, không cần backend lưu trữ nào. `hostPath`
   về kỹ thuật cũng chạy được nhưng kèm đúng những rủi ro ở câu 3. Chưa dùng được:
   `nfs`, `iscsi`, `fc` vì **phải có server lưu trữ riêng đang chạy**, và
   **`persistentVolumeClaim` vì nó chỉ mount một PersistentVolume đã tồn tại** — bài này nói
   nó là cách "yêu cầu" lưu trữ bền vững, còn PV đó ở đâu ra là việc của bài
   [92](92-persistent-volumes-vi.md). Cluster lab chưa có provisioner nên chưa có PV nào để
   claim; Lab 6a mới lấp chỗ trống này.
5. **Container đó sẽ không nhận được cập nhật khi ConfigMap thay đổi.** Bài ghi rõ điều này
   cho cả `configMap`, `secret` và `downwardAPI`: mount qua `subPath` là ảnh chụp một lần, khác
   với mount cả volume vốn được kubelet cập nhật khi nguồn đổi.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

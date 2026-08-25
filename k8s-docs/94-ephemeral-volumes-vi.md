# Volume tạm thời (Ephemeral Volumes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 7/16 · Kiểm chứng ở
[Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md).

Bài này gom lại thành một khái niệm những thứ bạn đã dùng rời rạc từ giai đoạn 3 (`emptyDir`,
`configMap`, `secret`) và thêm một loại mới đáng chú ý: **volume tạm thời tổng quát**, thứ
trông như `emptyDir` nhưng thực chất sinh ra một PVC thật. Đọc kỹ đúng phần đó và phần vòng
đời của nó.

**Phải hiểu ở lần đọc này:**

- Volume tạm thời được khai **inline** trong spec Pod, được tạo và xóa cùng Pod; nhờ vậy Pod
  dừng rồi chạy lại mà không bị buộc vào nơi có sẵn một persistent volume — đoạn mở đầu.
- Ba nhóm và ai cung cấp chúng: `emptyDir`/`configMap`/`downwardAPI`/`secret`/`image` do
  **kubelet** quản lý cục bộ trên từng node; **volume tạm thời CSI** bắt buộc phải có driver
  bên thứ ba viết riêng cho chế độ inline; **volume tạm thời tổng quát** dùng được với **mọi**
  driver hỗ trợ cấp phát động — mục *Các loại volume tạm thời*.
- Volume tạm thời tổng quát cho thêm những gì `emptyDir` không có: lưu trữ gắn qua mạng, kích
  thước cố định mà Pod không vượt được, dữ liệu ban đầu, và các thao tác snapshot / nhân bản /
  thay đổi kích thước / theo dõi dung lượng — mục *Volume tạm thời tổng quát*.
- Cơ chế thật của nó: ephemeral volume controller tạo một **PersistentVolumeClaim thật** trong
  cùng namespace, Pod là chủ sở hữu, và garbage collector xóa PVC khi Pod bị xóa; bài khuyến
  nghị dùng `WaitForFirstConsumer` để scheduler được tự do chọn node — mục *Vòng đời và
  PersistentVolumeClaim*.
- Tên PVC sinh ra là **xác định**: `<tên Pod>-<tên volume>`. Điều đó gây khả năng xung đột;
  PVC đã tồn tại **không bị ghi đè hay sửa đổi**, và Pod sẽ không khởi động được — mục *Đặt tên
  PersistentVolumeClaim*. Hệ quả bảo mật: ai tạo được Pod thì gián tiếp tạo được PVC, dù quota
  PVC theo namespace vẫn áp dụng — mục *Bảo mật*.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Volume tạm thời CSI* | cần driver CSI viết riêng cho chế độ inline, cluster lab chưa có | không cần |
| *Các hạn chế đối với driver CSI* (`volumeLifecycleModes`, admission webhook) | là thao tác hardening | giai đoạn 9 |
| Nhắc tới snapshot, nhân bản, theo dõi dung lượng trong *Volume tạm thời tổng quát* | mới chỉ cần biết chúng khả dụng | bài [99](99-volume-snapshots-vi.md), [101](101-volume-pvc-datasource-vi.md), [103](103-storage-capacity-vi.md) |
| Mục *Tiếp theo* với các link KEP | tài liệu thiết kế | không cần |

---

Tài liệu này mô tả *volume tạm thời* (ephemeral volume) trong Kubernetes. Bạn nên
làm quen trước với [volume](91-volumes-vi.md),
đặc biệt là PersistentVolumeClaim và PersistentVolume.

Một số ứng dụng cần lưu trữ bổ sung nhưng không quan tâm liệu dữ liệu
đó có được lưu bền vững qua các lần khởi động lại hay không. Ví dụ, các dịch vụ
cache thường bị giới hạn bởi kích thước bộ nhớ và có thể chuyển những dữ liệu
ít được dùng vào kho lưu trữ chậm hơn bộ nhớ mà ít ảnh hưởng
đến hiệu năng tổng thể.

Các ứng dụng khác lại kỳ vọng một số dữ liệu đầu vào chỉ đọc có sẵn trong
các file, chẳng hạn dữ liệu cấu hình hoặc các khóa bí mật (secret key).

*Volume tạm thời* (ephemeral volume) được thiết kế cho các trường hợp sử dụng này. Vì volume
đi theo vòng đời của Pod và được tạo cũng như xóa cùng với
Pod, các Pod có thể được dừng và khởi động lại mà không bị ràng buộc vào nơi
có sẵn một persistent volume nào đó.

Volume tạm thời được chỉ định *inline* (nội tuyến) trong spec của Pod, điều này
đơn giản hóa việc triển khai và quản lý ứng dụng.

### Các loại volume tạm thời (Types of ephemeral volumes)

Kubernetes hỗ trợ nhiều loại volume tạm thời khác nhau cho
các mục đích khác nhau:

- [emptyDir](91-volumes-vi.md#emptydir): rỗng khi Pod khởi động,
  với lưu trữ lấy cục bộ từ thư mục gốc của kubelet (thường là
  đĩa root) hoặc từ RAM
- [configMap](91-volumes-vi.md#configmap),
  [downwardAPI](91-volumes-vi.md#downwardapi),
  [secret](91-volumes-vi.md#secret): tiêm (inject) các
  loại dữ liệu Kubernetes khác nhau vào một Pod
- [image](91-volumes-vi.md#image): cho phép mount các file
  của container image hoặc các artifact trực tiếp vào một Pod.
- [Volume tạm thời CSI](#csi-ephemeral-volumes):
  tương tự các loại volume trước, nhưng được cung cấp bởi các driver CSI đặc biệt
  vốn [hỗ trợ riêng tính năng này](https://kubernetes-csi.github.io/docs/ephemeral-local-volumes.html)
- [Volume tạm thời tổng quát (generic ephemeral volume)](#generic-ephemeral-volumes),
  có thể được cung cấp bởi tất cả các driver lưu trữ nào cũng hỗ trợ persistent volume

`emptyDir`, `configMap`, `downwardAPI`, `secret` được cung cấp dưới dạng
[lưu trữ tạm thời cục bộ (local ephemeral storage)](95-ephemeral-storage-vi.md).
Chúng được quản lý bởi kubelet trên mỗi node.

Volume tạm thời CSI *bắt buộc* phải được cung cấp bởi các driver lưu trữ CSI
của bên thứ ba.

Volume tạm thời tổng quát *có thể* được cung cấp bởi các driver lưu trữ CSI
bên thứ ba, nhưng cũng có thể bởi bất kỳ driver lưu trữ nào khác hỗ trợ cấp phát
động (dynamic provisioning). Một số driver CSI được viết riêng cho volume
tạm thời CSI và không hỗ trợ cấp phát động: khi đó chúng
không thể được dùng cho volume tạm thời tổng quát.

Ưu điểm của việc dùng driver bên thứ ba là chúng có thể cung cấp
những chức năng mà bản thân Kubernetes không hỗ trợ, ví dụ
lưu trữ với đặc tính hiệu năng khác với đĩa do
kubelet quản lý, hoặc tiêm các dữ liệu khác nhau.

### Volume tạm thời CSI (CSI ephemeral volumes) {#csi-ephemeral-volumes}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

> **Ghi chú:**
> Volume tạm thời CSI chỉ được hỗ trợ bởi một tập con các driver CSI.
> [Danh sách Driver](https://kubernetes-csi.github.io/docs/drivers.html) CSI của Kubernetes
> cho biết những driver nào hỗ trợ volume tạm thời.

Về mặt khái niệm, volume tạm thời CSI tương tự các loại volume `configMap`,
`downwardAPI` và `secret`: phần lưu trữ được quản lý cục bộ trên mỗi
node và được tạo cùng với các tài nguyên cục bộ khác sau khi một Pod đã được
lập lịch (schedule) lên một node. Ở giai đoạn này, Kubernetes không còn khái niệm
lập lịch lại (reschedule) Pod nữa. Việc tạo volume phải hiếm khi thất bại,
nếu không quá trình khởi động Pod sẽ bị kẹt. Đặc biệt, [lập lịch Pod có nhận biết
dung lượng lưu trữ (storage capacity aware Pod scheduling)](103-storage-capacity-vi.md) *không*
được hỗ trợ cho các volume này. Hiện tại chúng cũng không được tính vào
giới hạn sử dụng tài nguyên lưu trữ của một Pod, vì đó là điều
mà kubelet chỉ có thể áp đặt cho phần lưu trữ do chính nó quản lý.

Dưới đây là một ví dụ manifest cho một Pod dùng lưu trữ tạm thời CSI:

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: my-csi-app
spec:
  containers:
    - name: my-frontend
      image: busybox:1.28
      volumeMounts:
      - mountPath: "/data"
        name: my-csi-inline-vol
      command: [ "sleep", "1000000" ]
  volumes:
    - name: my-csi-inline-vol
      csi:
        driver: inline.storage.kubernetes.io
        volumeAttributes:
          foo: bar
```

`volumeAttributes` xác định volume nào được chuẩn bị bởi
driver. Các thuộc tính này là đặc thù cho từng driver và không được
chuẩn hóa. Xem tài liệu của từng driver CSI để có hướng dẫn
chi tiết hơn.

### Các hạn chế đối với driver CSI (CSI driver restrictions)

Volume tạm thời CSI cho phép người dùng cung cấp `volumeAttributes`
trực tiếp cho driver CSI như một phần của spec Pod. Một driver CSI
cho phép các `volumeAttributes` vốn thường chỉ dành cho
quản trị viên là KHÔNG phù hợp để dùng làm volume tạm thời inline.
Ví dụ, các tham số thường được định nghĩa trong StorageClass
không nên được để lộ cho người dùng thông qua việc sử dụng volume tạm thời inline.

Các quản trị viên cluster cần hạn chế những driver CSI được
phép dùng làm volume inline trong spec của Pod có thể thực hiện bằng cách:

- Loại bỏ `Ephemeral` khỏi `volumeLifecycleModes` trong spec của CSIDriver, điều này ngăn
  driver được dùng làm volume tạm thời inline.
- Dùng một [admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
  để hạn chế cách driver này được sử dụng.

### Volume tạm thời tổng quát (Generic ephemeral volumes) {#generic-ephemeral-volumes}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [stable]`

Volume tạm thời tổng quát tương tự volume `emptyDir` ở chỗ
chúng cung cấp một thư mục cho mỗi Pod để chứa dữ liệu tạm (scratch data), thường
là rỗng sau khi cấp phát. Nhưng chúng cũng có thể có thêm các
tính năng khác:

- Lưu trữ có thể là cục bộ hoặc gắn qua mạng (network-attached).
- Volume có thể có kích thước cố định mà Pod không thể vượt quá.
- Volume có thể có sẵn một số dữ liệu ban đầu, tùy thuộc vào driver và
  các tham số.
- Các thao tác thông thường trên volume đều được hỗ trợ với điều kiện driver
  hỗ trợ chúng, bao gồm
  [snapshot](99-volume-snapshots-vi.md),
  [nhân bản (cloning)](101-volume-pvc-datasource-vi.md),
  [thay đổi kích thước (resizing)](92-persistent-volumes-vi.md#expanding-persistent-volumes-claims),
  và [theo dõi dung lượng lưu trữ (storage capacity tracking)](103-storage-capacity-vi.md).

Ví dụ:

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: my-app
spec:
  containers:
    - name: my-frontend
      image: busybox:1.28
      volumeMounts:
      - mountPath: "/scratch"
        name: scratch-volume
      command: [ "sleep", "1000000" ]
  volumes:
    - name: scratch-volume
      ephemeral:
        volumeClaimTemplate:
          metadata:
            labels:
              type: my-frontend-volume
          spec:
            accessModes: [ "ReadWriteOnce" ]
            storageClassName: "scratch-storage-class"
            resources:
              requests:
                storage: 1Gi
```

### Vòng đời và PersistentVolumeClaim (Lifecycle and PersistentVolumeClaim)

Ý tưởng thiết kế then chốt là
[các tham số cho một volume claim](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#ephemeralvolumesource-v1-core)
được cho phép nằm bên trong một nguồn volume của Pod. Các label, annotation và
toàn bộ tập các trường của một PersistentVolumeClaim đều được hỗ trợ. Khi một Pod như vậy được
tạo, ephemeral volume controller sau đó sẽ tạo một object PersistentVolumeClaim
thực sự trong cùng namespace với Pod và đảm bảo rằng PersistentVolumeClaim đó
được xóa khi Pod bị xóa.

Điều đó kích hoạt việc gắn kết (binding) và/hoặc cấp phát volume, hoặc ngay lập tức nếu
StorageClass dùng chế độ gắn kết volume tức thời (immediate volume binding), hoặc khi Pod được
lập lịch tạm thời lên một node (chế độ gắn kết volume
`WaitForFirstConsumer`). Chế độ sau được khuyến nghị cho volume tạm thời tổng quát,
vì khi đó scheduler được tự do chọn một node phù hợp cho
Pod. Với chế độ gắn kết tức thời, scheduler bị buộc phải chọn một node có
quyền truy cập tới volume ngay khi volume sẵn sàng.

Xét về [quyền sở hữu tài nguyên (resource ownership)](36-garbage-collection-vi.md#owners-dependents),
một Pod có lưu trữ tạm thời tổng quát là chủ sở hữu của (các) PersistentVolumeClaim
cung cấp phần lưu trữ tạm thời đó. Khi Pod bị xóa,
garbage collector của Kubernetes sẽ xóa PVC, và điều này sau đó thường
kích hoạt việc xóa volume vì chính sách thu hồi (reclaim policy) mặc định của
các storage class là xóa volume. Bạn có thể tạo lưu trữ cục bộ bán-tạm-thời (quasi-ephemeral)
bằng một StorageClass với reclaim policy là `retain`: phần lưu trữ sẽ tồn tại lâu hơn Pod,
và trong trường hợp này bạn cần đảm bảo việc dọn dẹp volume được thực hiện riêng.

Trong khi các PVC này tồn tại, chúng có thể được dùng như bất kỳ PVC nào khác. Đặc biệt,
chúng có thể được tham chiếu làm nguồn dữ liệu (data source) trong nhân bản volume hoặc
snapshot. Object PVC cũng giữ trạng thái hiện tại của
volume.

### Đặt tên PersistentVolumeClaim (PersistentVolumeClaim naming)

Việc đặt tên cho các PVC được tạo tự động là xác định (deterministic): tên là
sự kết hợp giữa tên Pod và tên volume, với một dấu gạch nối (`-`) ở
giữa. Trong ví dụ trên, tên PVC sẽ là
`my-app-scratch-volume`. Cách đặt tên xác định này giúp việc
tương tác với PVC dễ dàng hơn vì không cần phải tìm kiếm nó một khi
đã biết tên Pod và tên volume.

Cách đặt tên xác định này cũng dẫn tới khả năng xung đột giữa các
Pod khác nhau (một Pod "pod-a" với volume "scratch" và một Pod khác có tên
"pod" với volume "a-scratch" đều cho ra cùng một tên PVC
"pod-a-scratch") và giữa các Pod với các PVC được tạo thủ công.

Những xung đột như vậy sẽ được phát hiện: một PVC chỉ được dùng cho một volume
tạm thời nếu nó được tạo cho chính Pod đó. Việc kiểm tra này dựa trên
quan hệ sở hữu (ownership). Một PVC đã tồn tại sẽ không bị ghi đè hay
sửa đổi. Nhưng điều này không giải quyết được xung đột, vì thiếu
đúng PVC cần thiết thì Pod không thể khởi động.

> **Thận trọng:**
> Hãy cẩn thận khi đặt tên Pod và volume bên trong
> cùng một namespace, sao cho những xung đột này không thể xảy ra.

### Bảo mật (Security)

Việc dùng volume tạm thời tổng quát cho phép người dùng tạo PVC một cách gián tiếp
nếu họ có quyền tạo Pod, ngay cả khi họ không có quyền tạo PVC trực tiếp.
Các quản trị viên cluster phải nhận thức được điều này. Nếu điều này không phù hợp với mô hình bảo mật của họ,
họ nên dùng một [admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
từ chối các object như Pod có chứa volume tạm thời tổng quát.

[Quota thông thường theo namespace cho PVC](https://kubernetes.io/docs/concepts/policy/resource-quotas#storage-resource-quota)
vẫn được áp dụng, vì vậy ngay cả khi người dùng được phép dùng cơ chế mới này, họ cũng không thể dùng
nó để lách các chính sách khác.

## Tiếp theo (What's next)

### Volume tạm thời do kubelet quản lý (Ephemeral volumes managed by kubelet)

Xem [lưu trữ tạm thời cục bộ (local ephemeral storage)](95-ephemeral-storage-vi.md).

### Volume tạm thời CSI (CSI ephemeral volumes)

- Để biết thêm thông tin về thiết kế, xem
  [KEP Ephemeral Inline CSI volumes](https://github.com/kubernetes/enhancements/blob/ad6021b3d61a49040a3f835e12c8bb5424db2bbb/keps/sig-storage/20190122-csi-inline-volumes.md).
- Để biết thêm thông tin về sự phát triển tiếp theo của tính năng này, xem
  [issue theo dõi cải tiến #596](https://github.com/kubernetes/enhancements/issues/596).

### Volume tạm thời tổng quát (Generic ephemeral volumes)

- Để biết thêm thông tin về thiết kế, xem
  [KEP Generic ephemeral inline volumes](https://github.com/kubernetes/enhancements/blob/master/keps/sig-storage/1698-generic-ephemeral-volumes/README.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. `emptyDir` và volume tạm thời tổng quát đều biến mất cùng Pod. Vậy khác biệt thực chất giữa
   chúng là gì?
2. Cluster lab của bạn (`lab-k8s-master` + 2 worker) **chưa có StorageClass và chưa có provisioner**.
   Trong ba nhóm volume tạm thời mà bài liệt kê, nhóm nào bạn dùng được ngay và nhóm nào phải
   đợi Lab 6a? Vì sao.
3. Pod tên `my-app` có volume tạm thời tổng quát tên `scratch-volume`. PVC sinh ra tên gì, ai
   tạo và ai xóa nó? Nếu trong namespace đã có sẵn một PVC trùng tên do bạn tạo tay thì sao?
4. Bài khuyến nghị dùng `WaitForFirstConsumer` cho volume tạm thời tổng quát. Vì sao, và
   `Immediate` gây ra chuyện gì?
5. Vì sao việc cho phép volume tạm thời tổng quát lại là một quyết định **bảo mật**, không chỉ
   là một lựa chọn tiện lợi?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `emptyDir` chỉ là một thư mục lấy từ **lưu trữ cục bộ do kubelet quản lý** trên chính node
   đang chạy Pod. Volume tạm thời tổng quát **được cấp phát bởi một storage driver hỗ trợ cấp
   phát động**, nên nó có thể là **lưu trữ gắn qua mạng**, có **kích thước cố định mà Pod không
   vượt được**, có thể **có sẵn dữ liệu ban đầu**, và hỗ trợ các thao tác volume thông thường:
   snapshot, nhân bản, thay đổi kích thước, theo dõi dung lượng. Điểm giống nhau duy nhất là
   vòng đời gắn với Pod.
2. Dùng được ngay: **nhóm do kubelet quản lý** — `emptyDir`, `configMap`, `downwardAPI`,
   `secret`, `image`. Chúng được cấp dưới dạng lưu trữ tạm thời cục bộ, không cần bất kỳ driver
   nào. Phải đợi: **volume tạm thời tổng quát**, vì nó tạo ra một PVC thật và PVC đó cần một
   driver hỗ trợ **cấp phát động** mới có volume. **Volume tạm thời CSI** thì còn xa hơn: nó
   *bắt buộc* phải có driver CSI viết riêng cho chế độ inline, và Lab 6a không hứa điều đó.
3. PVC tên **`my-app-scratch-volume`** — tên Pod nối tên volume bằng một dấu gạch nối. **Ephemeral
   volume controller tạo** nó trong cùng namespace với Pod, và vì Pod là chủ sở hữu nên
   **garbage collector của Kubernetes xóa** nó khi Pod bị xóa. Nếu đã có PVC trùng tên do bạn
   tạo tay: **PVC đó không bị ghi đè hay sửa đổi** — việc kiểm tra dựa trên quan hệ sở hữu, một
   PVC chỉ được dùng cho volume tạm thời nếu nó được tạo cho chính Pod đó. Nhưng xung đột không
   vì thế mà được giải quyết: thiếu đúng PVC cần thiết thì **Pod không khởi động được**.
4. Vì với `WaitForFirstConsumer`, việc gắn kết và cấp phát chỉ diễn ra khi Pod đã được lập lịch
   tạm thời lên một node, nên **scheduler được tự do chọn node phù hợp cho Pod**. Với chế độ
   gắn kết tức thời (`Immediate`), volume được tạo trước và **scheduler bị buộc phải chọn một
   node có quyền truy cập tới volume đó** — đúng cái bẫy đã gặp ở bài
   [96](96-storage-classes-vi.md), chỉ khác là ở đây nó xảy ra sau lưng bạn vì PVC do controller
   sinh ra.
5. Vì cơ chế này cho phép **người dùng chỉ có quyền tạo Pod gián tiếp tạo được PVC**, kể cả khi
   họ không có quyền tạo PVC trực tiếp. Quản trị viên phải biết điều đó; nếu nó không hợp với
   mô hình bảo mật thì bài khuyên dùng admission webhook từ chối các Pod chứa volume tạm thời
   tổng quát. Điểm trấn an duy nhất là **quota PVC theo namespace vẫn được áp dụng**, nên không
   ai lách được các chính sách khác bằng đường này.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

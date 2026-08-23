# Thu gom rác (Garbage Collection)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/architecture/garbage-collection/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1c](LO-TRINH-ADMIN.md#1c-vòng-đời-và-cơ-chế-nền-của-object),
bài 3/7 · Kiểm chứng ở [Lab 1c](labs/LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md).

Bài này **dựa hoàn toàn** trên hai bài vừa đọc: [29 — Finalizers](29-finalizers-vi.md) và
[30 — Owner và dependent](30-owners-dependents-vi.md). Đọc lệch thứ tự sẽ không hiểu.

Lưu ý: "thu gom rác" ở đây gộp **hai chuyện hoàn toàn khác nhau** — dọn object trong API, và
dọn container/image trên đĩa node. Đừng lẫn hai phần đó.

**Phải hiểu ở lần đọc này:**

- Ba chính sách xóa: **foreground** (xóa phụ thuộc trước, chủ sở hữu vẫn hiện trong API cho
  tới khi xong), **background** (xóa chủ sở hữu ngay, dọn phụ thuộc ở nền — **mặc định**), và
  **orphan** (bỏ lại phụ thuộc).
- Foreground và orphan được hiện thực bằng **finalizer** đặt lên chính chủ sở hữu — đây là chỗ
  bài 29 và bài 30 khớp vào nhau.
- Trong xóa foreground, chỉ những phụ thuộc có `blockOwnerDeletion=true` **và đang nằm trong
  cache của garbage collector** mới thực sự chặn được.
- Phần thu gom container và image là việc của **kubelet trên từng node**, hoàn toàn tách khỏi
  cơ chế owner reference: image mỗi 5 phút, container mỗi phút.
- Cơ chế ngưỡng đĩa `HighThresholdPercent` / `LowThresholdPercent`: vượt ngưỡng cao thì xóa
  image từ cũ nhất cho tới khi xuống ngưỡng thấp.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Danh sách tài nguyên được dọn ở đầu bài (Job, PV, CSR…) | phần lớn chưa học | giai đoạn 4, 6, 9 |
| `MinAge`, `MaxPerPodContainer`, `MaxContainers` | là tham số tinh chỉnh kubelet | giai đoạn 12 và CP5 |
| `imageMaximumGCAge` | cấu hình theo từng node | giai đoạn 12 |
| Ghi chú owner reference cross-namespace | đã đọc ở bài [30](30-owners-dependents-vi.md) | không cần đọc lại |

---

Thu gom rác (garbage collection) là thuật ngữ chung cho các cơ chế khác nhau mà Kubernetes dùng
để dọn dẹp tài nguyên trong cluster. Điều này cho phép dọn dẹp các tài nguyên như sau:

* [Các pod đã kết thúc](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-garbage-collection)
* [Các Job đã hoàn thành](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
* [Các đối tượng không có owner reference](#owners-dependents)
* [Các container và container image không dùng nữa](#containers-images)
* [Các PersistentVolume được cấp phát động (dynamically provisioned) có reclaim policy của StorageClass là Delete](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#delete)
* [Các CertificateSigningRequest (CSR) cũ hoặc đã hết hạn](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#request-signing-process)
* Các Node bị xóa trong các kịch bản sau:
  * Trên cloud khi cluster dùng [cloud controller manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)
  * Tại chỗ (on-premises) khi cluster dùng một addon tương tự cloud controller manager
* [Các đối tượng Lease của Node](./23-nodes-vi.md#nhịp-tim-của-node-node-heartbeats)

## Chủ sở hữu và đối tượng phụ thuộc (Owners and dependents) {#owners-dependents}

Nhiều đối tượng trong Kubernetes liên kết với nhau thông qua [*owner reference*](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/)
(tham chiếu chủ sở hữu). Owner reference cho control plane biết những đối tượng nào phụ thuộc
vào đối tượng khác. Kubernetes dùng owner reference để trao cho control plane, cũng như các
API client khác, cơ hội dọn dẹp các tài nguyên liên quan trước khi xóa một đối tượng.
Trong hầu hết trường hợp, Kubernetes quản lý owner reference một cách tự động.

Quyền sở hữu (ownership) khác với cơ chế [label và selector](./18-labels-vi.md)
mà một số tài nguyên cũng sử dụng. Ví dụ, xét một Service tạo ra các đối tượng
`EndpointSlice`. Service dùng *label* để cho phép control plane xác định những đối tượng
`EndpointSlice` nào đang được dùng cho Service đó. Bên cạnh label, mỗi `EndpointSlice`
được quản lý thay mặt cho một Service còn có một owner reference. Owner reference giúp
các phần khác nhau của Kubernetes tránh can thiệp vào những đối tượng mà chúng không kiểm soát.

> **Ghi chú:**
> Owner reference giữa các namespace (cross-namespace) bị cấm theo thiết kế.
> Các đối tượng phụ thuộc thuộc phạm vi namespace có thể chỉ định chủ sở hữu thuộc phạm vi cluster
> hoặc phạm vi namespace. Một chủ sở hữu thuộc phạm vi namespace **bắt buộc** phải tồn tại trong
> cùng namespace với đối tượng phụ thuộc. Nếu không, owner reference bị coi như không tồn tại,
> và đối tượng phụ thuộc có thể bị xóa một khi tất cả chủ sở hữu được xác minh là không tồn tại.
>
> Các đối tượng phụ thuộc thuộc phạm vi cluster chỉ có thể chỉ định chủ sở hữu thuộc phạm vi cluster.
> Từ v1.20 trở đi, nếu một đối tượng phụ thuộc thuộc phạm vi cluster chỉ định một kind thuộc phạm vi
> namespace làm chủ sở hữu, nó bị coi là có owner reference không thể phân giải được,
> và không thể bị thu gom rác.
>
> Từ v1.20 trở đi, nếu garbage collector phát hiện một `ownerReference` cross-namespace không hợp lệ,
> hoặc một đối tượng phụ thuộc thuộc phạm vi cluster có `ownerReference` tham chiếu tới một kind
> thuộc phạm vi namespace, một Event cảnh báo với reason là `OwnerRefInvalidNamespace` và
> `involvedObject` là đối tượng phụ thuộc không hợp lệ sẽ được ghi nhận.
> Bạn có thể kiểm tra loại Event đó bằng cách chạy
> `kubectl get events -A --field-selector=reason=OwnerRefInvalidNamespace`.

## Xóa theo tầng (Cascading deletion) {#cascading-deletion}

Kubernetes kiểm tra và xóa những đối tượng không còn owner reference, chẳng hạn các pod
bị bỏ lại khi bạn xóa một ReplicaSet. Khi xóa một đối tượng, bạn có thể kiểm soát việc
Kubernetes có tự động xóa các đối tượng phụ thuộc của nó hay không, trong một quá trình
gọi là *xóa theo tầng* (cascading deletion). Có hai kiểu xóa theo tầng như sau:

* Xóa theo tầng foreground (foreground cascading deletion)
* Xóa theo tầng background (background cascading deletion)

Bạn cũng có thể kiểm soát cách thức và thời điểm thu gom rác xóa những tài nguyên có
owner reference bằng các finalizer của Kubernetes.

### Xóa theo tầng foreground (Foreground cascading deletion) {#foreground-deletion}

Trong xóa theo tầng foreground, đối tượng chủ sở hữu mà bạn đang xóa trước tiên chuyển sang
trạng thái *đang trong quá trình xóa* (deletion in progress). Ở trạng thái này, những điều sau
xảy ra với đối tượng chủ sở hữu:

* API server của Kubernetes đặt trường `metadata.deletionTimestamp` của đối tượng thành
  thời điểm đối tượng được đánh dấu để xóa.
* API server của Kubernetes cũng đặt trường `metadata.finalizers` thành
  `foregroundDeletion`.
* Đối tượng vẫn hiển thị qua Kubernetes API cho tới khi quá trình xóa hoàn tất.

Sau khi đối tượng chủ sở hữu chuyển sang trạng thái *đang trong quá trình xóa*, controller
xóa các đối tượng phụ thuộc mà nó biết. Sau khi xóa hết tất cả các đối tượng phụ thuộc mà nó biết,
controller xóa đối tượng chủ sở hữu. Tại thời điểm này, đối tượng không còn hiển thị trong
Kubernetes API nữa.

Trong quá trình xóa theo tầng foreground, những đối tượng phụ thuộc duy nhất chặn việc xóa
chủ sở hữu là những đối tượng có trường `ownerReference.blockOwnerDeletion=true`
và đang nằm trong cache của controller thu gom rác. Cache của controller thu gom rác
có thể không chứa những đối tượng mà loại tài nguyên của chúng không thể list / watch thành công,
hoặc những đối tượng được tạo đồng thời với việc xóa một đối tượng chủ sở hữu.
Xem [Dùng xóa theo tầng foreground](https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/#use-foreground-cascading-deletion)
để tìm hiểu thêm.

### Xóa theo tầng background (Background cascading deletion) {#background-deletion}

Trong xóa theo tầng background, API server của Kubernetes xóa đối tượng chủ sở hữu ngay lập tức
và controller garbage collector (tùy chỉnh hoặc mặc định) dọn dẹp các đối tượng phụ thuộc
ở chế độ nền. Nếu tồn tại finalizer, nó đảm bảo các đối tượng không bị xóa cho tới khi
mọi tác vụ dọn dẹp cần thiết đã hoàn tất. Theo mặc định, Kubernetes dùng xóa theo tầng background
trừ khi bạn chủ động dùng xóa foreground hoặc chọn bỏ lại (orphan) các đối tượng phụ thuộc.

Xem [Dùng xóa theo tầng background](https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/#use-background-cascading-deletion)
để tìm hiểu thêm.

### Các đối tượng phụ thuộc mồ côi (Orphaned dependents)

Khi Kubernetes xóa một đối tượng chủ sở hữu, các đối tượng phụ thuộc bị bỏ lại được gọi là
các đối tượng *mồ côi* (orphan). Theo mặc định, Kubernetes xóa các đối tượng phụ thuộc. Để tìm hiểu
cách ghi đè hành vi này, xem [Xóa đối tượng chủ sở hữu và bỏ lại các đối tượng phụ thuộc](https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/#set-orphan-deletion-policy).

## Thu gom rác các container và image không dùng nữa (Garbage collection of unused containers and images) {#containers-images}

kubelet thực hiện thu gom rác đối với các image không dùng nữa mỗi năm phút một lần và
đối với các container không dùng nữa mỗi phút một lần. Bạn nên tránh dùng các công cụ
thu gom rác bên ngoài, vì chúng có thể phá vỡ hành vi của kubelet và xóa những container
lẽ ra phải tồn tại.

Để cấu hình các tùy chọn thu gom rác cho container và image không dùng nữa, hãy tinh chỉnh
kubelet bằng một [file cấu hình](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)
và thay đổi các tham số liên quan tới thu gom rác thông qua loại tài nguyên
[`KubeletConfiguration`](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).

### Vòng đời của container image (Container image lifecycle)

Kubernetes quản lý vòng đời của tất cả các image thông qua *trình quản lý image* (image manager),
một phần của kubelet, với sự phối hợp của cadvisor. Khi ra quyết định thu gom rác,
kubelet xem xét các giới hạn sử dụng đĩa sau:

* `HighThresholdPercent`
* `LowThresholdPercent`

Mức sử dụng đĩa vượt trên giá trị `HighThresholdPercent` đã cấu hình sẽ kích hoạt thu gom rác;
quá trình này xóa các image theo thứ tự dựa trên lần cuối chúng được sử dụng,
bắt đầu từ image cũ nhất. kubelet xóa các image cho tới khi mức sử dụng đĩa
xuống tới giá trị `LowThresholdPercent`.

#### Thu gom rác cho các container image không dùng nữa (Garbage collection for unused container images) {#image-maximum-age-gc}

Bạn có thể chỉ định khoảng thời gian tối đa mà một image cục bộ có thể không được sử dụng,
bất kể mức sử dụng đĩa. Đây là một thiết lập của kubelet mà bạn cấu hình cho từng node.

Để cấu hình thiết lập này, bạn cần đặt giá trị cho trường `imageMaximumGCAge`
trong file cấu hình kubelet.

Giá trị được chỉ định dưới dạng duration (khoảng thời gian) của Kubernetes.
Xem [duration](https://kubernetes.io/docs/reference/glossary/?all=true#term-duration)
trong bảng thuật ngữ để biết thêm chi tiết.

Ví dụ, bạn có thể đặt trường cấu hình này thành `12h45m`,
nghĩa là 12 giờ 45 phút.

> **Ghi chú:**
> Tính năng này không theo dõi việc sử dụng image qua các lần khởi động lại kubelet. Nếu kubelet
> được khởi động lại, tuổi image đang được theo dõi sẽ bị đặt lại, khiến kubelet phải chờ đủ
> khoảng thời gian `imageMaximumGCAge` trước khi các image đủ điều kiện được thu gom rác
> dựa trên tuổi của image.

### Thu gom rác container (Container garbage collection) {#container-image-garbage-collection}

kubelet thu gom rác các container không dùng nữa dựa trên các biến sau, mà bạn có thể tự định nghĩa:

* `MinAge`: tuổi tối thiểu để kubelet có thể thu gom rác một container.
  Tắt bằng cách đặt thành `0`.
* `MaxPerPodContainer`: số lượng container "chết" (dead) tối đa mà mỗi Pod
  có thể có. Tắt bằng cách đặt giá trị nhỏ hơn `0`.
* `MaxContainers`: số lượng container chết tối đa mà cluster có thể có.
  Tắt bằng cách đặt giá trị nhỏ hơn `0`.

Ngoài các biến này, kubelet còn thu gom rác các container không xác định được danh tính
và các container đã bị xóa, thường bắt đầu từ container cũ nhất.

`MaxPerPodContainer` và `MaxContainers` có khả năng xung đột với nhau trong những tình huống
mà việc giữ số container tối đa cho mỗi Pod (`MaxPerPodContainer`) sẽ vượt quá
tổng số container chết toàn cục cho phép (`MaxContainers`). Trong tình huống này,
kubelet điều chỉnh `MaxPerPodContainer` để giải quyết xung đột. Kịch bản xấu nhất là
hạ `MaxPerPodContainer` xuống `1` và loại bỏ (evict) các container cũ nhất.
Ngoài ra, các container thuộc về những pod đã bị xóa sẽ bị gỡ bỏ một khi chúng
cũ hơn `MinAge`.

> **Ghi chú:**
> kubelet chỉ thu gom rác những container mà nó quản lý.

## Cấu hình thu gom rác (Configuring garbage collection) {#configuring-gc}

Bạn có thể tinh chỉnh việc thu gom rác các tài nguyên bằng cách cấu hình các tùy chọn
riêng của các controller quản lý những tài nguyên đó. Các trang sau hướng dẫn bạn
cách cấu hình thu gom rác:

* [Cấu hình xóa theo tầng cho các đối tượng Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/)
* [Cấu hình dọn dẹp các Job đã hoàn thành](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)

## Tiếp theo (What's next)

* Tìm hiểu thêm về [quyền sở hữu của các đối tượng Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/).
* Tìm hiểu thêm về [finalizer](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/) trong Kubernetes.
* Tìm hiểu về [TTL controller](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/) dọn dẹp các Job đã hoàn thành.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Ba chính sách xóa theo tầng là gì? Cái nào là mặc định?
2. Bạn xóa một chủ sở hữu bằng foreground. Object đó biến mất khỏi `kubectl get` ngay hay còn
   nhìn thấy? Vì sao?
3. Foreground và orphan được hiện thực bằng cơ chế nào đã học ở bài [29](29-finalizers-vi.md)?
4. Một object phụ thuộc có `blockOwnerDeletion=true` nhưng vẫn không chặn được việc xóa chủ sở
   hữu. Bài nêu lý do gì?
5. Thành phần nào dọn container và image không dùng nữa? Nó có liên quan gì tới owner
   reference không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Foreground**, **background**, **orphan**. Kubernetes dùng **background** mặc định, trừ khi
   bạn chủ động chọn foreground hoặc chọn bỏ lại phụ thuộc.
2. **Vẫn nhìn thấy.** Trong xóa foreground, chủ sở hữu chuyển sang trạng thái đang trong quá
   trình xóa: `metadata.deletionTimestamp` được đặt, `metadata.finalizers` có
   `foregroundDeletion`, và object **vẫn hiển thị qua API** cho tới khi mọi phụ thuộc mà
   controller biết đã bị xóa xong.
3. Bằng **finalizer** đặt trên chính object chủ sở hữu: `foregroundDeletion` cho foreground,
   `orphan` cho orphan. Đây chính là cơ chế "chờ đến khi điều kiện được thỏa mãn" của bài 29.
4. Vì nó **không nằm trong cache của controller thu gom rác**. Cache có thể thiếu những object
   mà loại tài nguyên của chúng không list/watch thành công, hoặc những object được tạo đồng
   thời với lúc chủ sở hữu bị xóa.
5. **Kubelet** trên từng node — image mỗi 5 phút, container mỗi phút. Việc này **không liên
   quan** tới owner reference; nó dựa trên tuổi, số lượng và ngưỡng sử dụng đĩa. Kubelet cũng
   chỉ dọn những container do chính nó quản lý.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

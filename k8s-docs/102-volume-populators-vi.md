# Volume Populator và Nguồn dữ liệu (Volume Populators and Data Sources)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/volume-populators-and-data-sources/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](LO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 13/16 · Kiểm chứng ở
Lab 6b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này khái quát hóa hai thứ bạn vừa học: nhân bản volume và khôi phục từ snapshot chỉ là hai
**nguồn dữ liệu tích hợp sẵn**, còn volume populator là cơ chế mở để cắm thêm nguồn khác. Với
vai trò admin, phần đáng nhớ nhất không phải populator mà là **khác biệt giữa hai trường
`dataSource` và `dataSourceRef`** — đó là thứ bạn sẽ gặp khi đọc manifest của người khác.

**Phải hiểu ở lần đọc này:**

- Bài toán: PVC mới thường được cấp một volume rỗng; *nguồn dữ liệu* cho phép yêu cầu volume
  được điền sẵn, còn *volume populator* là các **controller bên ngoài** thực hiện việc điền đó
  dựa trên một custom resource mà PVC tham chiếu — đoạn mở đầu.
- Hai nguồn dữ liệu **tích hợp sẵn** đã có từ trước: nhân bản một volume có sẵn và khôi phục
  một volume snapshot — đoạn mở đầu.
- Khác biệt giữa hai trường: `dataSource` **chỉ nhận PVC hoặc VolumeSnapshot** và **âm thầm bỏ
  qua giá trị không hợp lệ** như thể trường đó để trống; `dataSourceRef` **nhận mọi kiểu đối
  tượng** trong cùng namespace (trừ đối tượng core không phải PVC) và **báo lỗi** khi giá trị
  không hợp lệ. Cả hai bất biến sau khi tạo, được API server gán cho khớp nhau, và đặt hai giá
  trị khác nhau sẽ gây lỗi xác thực — mục *Tham chiếu nguồn dữ liệu*.
- Điều kiện dùng populator tùy chỉnh: bật feature gate `AnyVolumeDataSource` cho **cả
  kube-apiserver và kube-controller-manager**; vì populator là thành phần bên ngoài nên thiếu
  nó thì việc tạo PVC có thể thất bại, và controller bên ngoài nên sinh Event trên PVC để báo
  trạng thái — mục *Volume populator và nguồn dữ liệu*, *Sử dụng volume populator*.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Nguồn dữ liệu giữa các namespace* và *Sử dụng nguồn dữ liệu volume giữa các namespace* | alpha; cần thêm gate `CrossNamespaceVolumeDataSource` và ReferenceGrant của Gateway API | không cần |
| Controller `volume data source validator` | alpha, là thành phần phụ trợ để cảnh báo | không cần |
| Custom resource và CustomResourceDefinition làm nguồn dữ liệu | chưa học CRD | giai đoạn 14, bài [179](179-custom-resources-vi.md) |

---

Tài liệu này mô tả *volume populator* và *nguồn dữ liệu* (data source) trong Kubernetes.
Bạn nên làm quen trước với [persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

Khi bạn tạo một PersistentVolumeClaim,
volume mà Kubernetes cấp phát (provision) cho nó thường bắt đầu ở trạng thái rỗng. Một *nguồn dữ liệu*
(data source) cho phép bạn thay vào đó yêu cầu volume mới được điền sẵn (pre-populated) dữ liệu có sẵn.
*Volume populator* là các controller thực hiện việc điền dữ liệu đó, dựa trên
nguồn dữ liệu mà PersistentVolumeClaim tham chiếu.

Kubernetes hỗ trợ sẵn (built-in) các nguồn dữ liệu để
[nhân bản một volume có sẵn](https://kubernetes.io/docs/concepts/storage/volume-pvc-datasource/) hoặc
[khôi phục một volume snapshot](https://kubernetes.io/docs/concepts/storage/volume-snapshots/). Các volume
populator tùy chỉnh mở rộng cơ chế này. Nguồn dữ liệu là một tài nguyên tùy chỉnh (custom resource), tức là một đối tượng
có kiểu được định nghĩa bởi một
CustomResourceDefinition.
Một populator controller theo dõi các PersistentVolumeClaim tham chiếu tới tài nguyên như vậy
và điền dữ liệu cho volume mới từ tài nguyên đó.

## Volume populator và nguồn dữ liệu (Volume populators and data sources)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [beta]`

Kubernetes hỗ trợ volume populator tùy chỉnh.
Để sử dụng volume populator tùy chỉnh, bạn phải bật
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `AnyVolumeDataSource` cho
kube-apiserver và kube-controller-manager.

Volume populator tận dụng một trường trong spec của PVC có tên `dataSourceRef`. Khác với
trường `dataSource` vốn chỉ có thể chứa tham chiếu tới một PersistentVolumeClaim khác
hoặc tới một VolumeSnapshot, trường `dataSourceRef` có thể chứa tham chiếu tới bất kỳ đối tượng nào trong
cùng namespace, ngoại trừ các đối tượng core không phải PVC. Với các cluster đã bật
feature gate này, nên ưu tiên dùng `dataSourceRef` thay cho `dataSource`.

## Tham chiếu nguồn dữ liệu (Data source references)

Trường `dataSourceRef` hoạt động gần như giống hệt trường `dataSource`. Nếu một trong hai
được chỉ định còn trường kia thì không, API server sẽ gán cùng một giá trị cho cả hai trường. Cả hai
trường đều không thể thay đổi sau khi tạo, và việc cố gắng chỉ định các giá trị khác nhau cho hai
trường sẽ dẫn đến lỗi xác thực (validation error). Do đó hai trường sẽ luôn có nội dung
giống nhau.

Có hai điểm khác biệt giữa trường `dataSourceRef` và trường `dataSource` mà
người dùng cần lưu ý:

* Trường `dataSource` bỏ qua các giá trị không hợp lệ (như thể trường đó để trống), trong khi
  trường `dataSourceRef` không bao giờ bỏ qua giá trị và sẽ gây lỗi nếu một giá trị không hợp lệ được
  sử dụng. Giá trị không hợp lệ là bất kỳ đối tượng core nào (đối tượng không có apiGroup) ngoại trừ PVC.
* Trường `dataSourceRef` có thể chứa nhiều kiểu đối tượng khác nhau, trong khi trường `dataSource`
  chỉ cho phép PVC và VolumeSnapshot.

Khi tính năng `CrossNamespaceVolumeDataSource` được bật, có thêm các khác biệt sau:

* Trường `dataSource` chỉ cho phép các đối tượng cục bộ (local), trong khi trường `dataSourceRef` cho phép
  đối tượng ở bất kỳ namespace nào.
* Khi namespace được chỉ định, `dataSource` và `dataSourceRef` không được đồng bộ với nhau.

Người dùng nên luôn dùng `dataSourceRef` trên các cluster đã bật feature gate, và
quay về dùng `dataSource` trên các cluster chưa bật. Không cần thiết phải xem cả hai trường
trong bất kỳ trường hợp nào. Các giá trị trùng lặp với ngữ nghĩa hơi khác nhau tồn tại chỉ vì
mục đích tương thích ngược (backwards compatibility). Đặc biệt, các controller cũ và mới có thể
tương tác được với nhau vì hai trường này giống nhau.

### Sử dụng volume populator (Using volume populators)

Volume populator là các controller có thể
tạo các volume không rỗng, trong đó nội dung của volume được xác định bởi một Custom Resource.
Người dùng tạo một volume được điền sẵn dữ liệu bằng cách tham chiếu tới một Custom Resource qua trường `dataSourceRef`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: populated-pvc
spec:
  dataSourceRef:
    name: example-name
    kind: ExampleDataSource
    apiGroup: example.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Vì volume populator là các thành phần bên ngoài, việc tạo một PVC sử dụng chúng
có thể thất bại nếu không phải tất cả các thành phần cần thiết đều được cài đặt. Các controller bên ngoài nên sinh ra
các event trên PVC để phản hồi về trạng thái của quá trình tạo, bao gồm cả cảnh báo nếu
PVC không thể được tạo do thiếu thành phần nào đó.

Bạn có thể cài đặt controller [volume data source validator](https://github.com/kubernetes-csi/volume-data-source-validator)
(đang ở giai đoạn alpha) vào cluster của mình. Controller đó sinh ra các Event cảnh báo trên một PVC trong trường hợp không có populator nào
được đăng ký để xử lý kiểu nguồn dữ liệu đó. Khi một populator phù hợp đã được cài đặt cho một PVC,
populator controller đó có trách nhiệm báo cáo các Event liên quan đến việc tạo volume và các sự cố trong
quá trình này.

## Nguồn dữ liệu giữa các namespace (Cross namespace data sources)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [alpha]`

Kubernetes hỗ trợ nguồn dữ liệu volume giữa các namespace (cross namespace volume data source).
Để sử dụng nguồn dữ liệu volume giữa các namespace, bạn phải bật các
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`AnyVolumeDataSource` và `CrossNamespaceVolumeDataSource` cho
kube-apiserver và kube-controller-manager.
Ngoài ra, bạn phải bật feature gate `CrossNamespaceVolumeDataSource` cho csi-provisioner.

Việc bật feature gate `CrossNamespaceVolumeDataSource` cho phép bạn chỉ định
một namespace trong trường dataSourceRef.

> **Ghi chú:**
> Khi bạn chỉ định một namespace cho nguồn dữ liệu volume, Kubernetes kiểm tra
> ReferenceGrant trong namespace kia trước khi chấp nhận tham chiếu.
> ReferenceGrant là một phần của các API mở rộng `gateway.networking.k8s.io`.
> Xem [ReferenceGrant](https://gateway-api.sigs.k8s.io/api-types/referencegrant/)
> trong tài liệu Gateway API để biết chi tiết.
> Điều này có nghĩa là bạn phải mở rộng cluster Kubernetes của mình với ít nhất ReferenceGrant từ
> Gateway API trước khi có thể sử dụng cơ chế này.

### Sử dụng nguồn dữ liệu volume giữa các namespace (Using a cross-namespace volume data source)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [alpha]`

Tạo một ReferenceGrant để cho phép chủ sở hữu namespace chấp nhận tham chiếu.
Bạn định nghĩa một volume được điền sẵn dữ liệu bằng cách chỉ định một nguồn dữ liệu volume
ở namespace khác qua trường `dataSourceRef`. Bạn phải có sẵn một ReferenceGrant hợp lệ
trong namespace nguồn:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-ns1-pvc
  namespace: default
spec:
  from:
  - group: ""
    kind: PersistentVolumeClaim
    namespace: ns1
  to:
  - group: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: new-snapshot-demo
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: foo-pvc
  namespace: ns1
spec:
  storageClassName: example
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  dataSourceRef:
    apiGroup: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: new-snapshot-demo
    namespace: default
  volumeMode: Filesystem
```

## Tiếp theo (What's next)

* Tìm hiểu về [Persistent Volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).
* Tìm hiểu về [Nhân bản CSI Volume](https://kubernetes.io/docs/concepts/storage/volume-pvc-datasource/).
* Tìm hiểu về [Volume Snapshot](https://kubernetes.io/docs/concepts/storage/volume-snapshots/).
* Đọc về các [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
  được nhắc đến trong trang này.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Bạn đặt `dataSource` trỏ tới một đối tượng không hợp lệ, nhưng PVC vẫn được tạo ra như thể
   trường đó trống. Vì sao? `dataSourceRef` sẽ hành xử khác thế nào?
2. Bạn khai cả `dataSource` lẫn `dataSourceRef` với hai giá trị khác nhau trong cùng một PVC.
   Kết quả là gì? Còn nếu chỉ khai một trong hai?
3. Trước bài này bạn đã dùng hai nguồn dữ liệu mà Kubernetes hỗ trợ sẵn, không cần cài
   populator nào. Đó là hai nguồn nào?
4. Cluster lab của bạn chưa bật feature gate nào ngoài mặc định. Bạn dùng được `dataSourceRef`
   với một custom resource không? Cần bật gate nào và ở những thành phần nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì **`dataSource` bỏ qua các giá trị không hợp lệ**, đối xử với chúng như thể trường để
   trống — bạn không nhận được lỗi nào và chỉ phát hiện ra khi volume hóa ra rỗng. Giá trị
   không hợp lệ ở đây là bất kỳ đối tượng core nào (đối tượng không có apiGroup) ngoại trừ PVC.
   Ngược lại, **`dataSourceRef` không bao giờ bỏ qua giá trị và sẽ gây lỗi** khi giá trị không
   hợp lệ. Đó chính là lý do bài khuyên ưu tiên `dataSourceRef` trên các cluster đã bật feature
   gate.
2. Đặt hai giá trị khác nhau sẽ **gây lỗi xác thực (validation error)**. Nếu chỉ khai một
   trường còn trường kia để trống thì **API server gán cùng một giá trị cho cả hai**, nên trong
   thực tế hai trường luôn có nội dung giống nhau. Cả hai đều **không thay đổi được sau khi
   tạo**. Hai trường trùng lặp với ngữ nghĩa hơi khác nhau tồn tại chỉ vì tương thích ngược,
   để controller cũ và mới làm việc được với nhau — bạn **không cần đọc cả hai trường** bao giờ.
3. **Nhân bản một volume có sẵn** và **khôi phục một volume snapshot**. Bài mở đầu bằng đúng
   hai nguồn dữ liệu tích hợp sẵn này, rồi nói volume populator tùy chỉnh là phần mở rộng của
   chính cơ chế đó.
4. **Không.** Muốn dùng volume populator tùy chỉnh, bạn phải bật feature gate
   **`AnyVolumeDataSource`** cho **kube-apiserver và kube-controller-manager**. Trên cluster
   chưa bật gate, bài khuyên quay về dùng `dataSource` — nhưng `dataSource` chỉ nhận PVC và
   VolumeSnapshot, nên custom resource thì vẫn không dùng được. Ngoài ra populator là thành
   phần bên ngoài: thiếu nó thì việc tạo PVC có thể thất bại, và bạn nên tìm nguyên nhân trong
   các Event trên chính PVC đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

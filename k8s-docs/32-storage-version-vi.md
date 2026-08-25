# Các phiên bản lưu trữ (Storage Versions)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/storage-version/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1c](00-ALO-TRINH-ADMIN.md#1c-vòng-đời-và-cơ-chế-nền-của-object),
bài 5/7 · Kiểm chứng ở [Lab 1c](labs/LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md).

Bài [21](21-kubernetes-api-vi.md) đã nói mọi phiên bản API chỉ là những cách biểu diễn của cùng
một dữ liệu lưu trữ. Bài này trả lời câu hỏi kế tiếp: **dữ liệu đó thực sự nằm ở dạng nào?**

**Phải hiểu ở lần đọc này:**

- **Phiên bản API và phiên bản lưu trữ là hai thứ tách biệt.** Object `v1alpha1` và `v1beta1`
  của cùng một resource có thể được mã hóa **giống hệt nhau** trong etcd.
- Mỗi resource có **đúng một** phiên bản lưu trữ đang hoạt động tại một thời điểm; mọi thao
  tác **ghi** đều lưu ở phiên bản đó.
- **Đọc thì chuyển đổi, ghi thì di chuyển.** Vì vậy object cũ có thể nằm ở phiên bản lưu trữ
  cũ **vô thời hạn** nếu không ai ghi lại nó.
- Hệ quả vận hành: bạn **không gỡ được** một phiên bản API cũ cho tới khi chắc chắn không còn
  object nào lưu ở phiên bản đó — gỡ sớm là không đọc được object nữa.
- Với **encryption at rest**, hệ quả còn nặng hơn: khi xoay khóa, phải giữ cả khóa cũ lẫn khóa
  mới cho tới khi mọi object đã được ghi lại ít nhất một lần.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Toàn bộ mục *Phiên bản lưu trữ cho custom resource* và ví dụ CRD `crontabs` | chưa học CRD | giai đoạn 14 |
| Chi tiết KMS provider và cách bật mã hóa | là tác vụ vận hành riêng | giai đoạn 22 |
| Quy trình storage version migration | tác vụ vận hành | giai đoạn 22 |

Đây là bài **khái niệm nền**. Ở giai đoạn 1 chỉ cần nhận ra vấn đề tồn tại; thao tác thật nằm
ở phần checkpoint tasks.

---

API server của Kubernetes lưu trữ các đối tượng (object), dựa trên một kho lưu trữ
phía sau (backing store) tương thích với etcd (thông thường, kho lưu trữ phía sau
chính là etcd). Mỗi đối tượng được tuần tự hóa (serialize) bằng một phiên bản cụ thể
của loại API đó; ví dụ, biểu diễn v1 của một ConfigMap. Kubernetes dùng thuật ngữ
_phiên bản lưu trữ_ (storage version) để mô tả cách một đối tượng được lưu trữ
trong cluster của bạn.

API của Kubernetes cũng dựa trên cơ chế chuyển đổi (conversion) tự động; ví dụ,
nếu bạn có một HorizontalPodAutoscaler, bạn có thể tương tác với
HorizontalPodAutoscaler đó bằng bất kỳ tổ hợp nào giữa phiên bản v1 và v2 của API
HorizontalPodAutoscaler. Kubernetes chịu trách nhiệm chuyển đổi từng lời gọi API
sao cho client không nhìn thấy phiên bản nào thực sự được tuần tự hóa.

Với người quản trị cluster, phiên bản lưu trữ của đối tượng là một khái niệm
quan trọng cần hiểu, vì nó chính là mắt xích nối giữa biểu diễn API của đối tượng
với cách mã hóa (encoding) thực tế trong backend lưu trữ. Điều này có thể quan trọng
khi cách mã hóa nhị phân bên dưới của đối tượng có ý nghĩa, chẳng hạn với
mã hóa dữ liệu lưu trữ (encryption at rest), hoặc khi loại bỏ dần API (API deprecation).

Cùng một API có thể có nhiều phiên bản lưu trữ mà API Server sau đó có thể
chuyển đổi thành một lược đồ đối tượng (object schema). Một đối tượng đơn lẻ
thuộc tài nguyên đó chỉ được phép có một phiên bản lưu trữ tại bất kỳ thời điểm nào.
Điều này nghĩa là API Server nắm được cách mã hóa nhị phân của các đối tượng và
có khả năng chuyển đổi một cách linh hoạt giữa tất cả các phiên bản đã lưu
sang biểu diễn API của đối tượng.

Phiên bản của một đối tượng hoàn toàn tách biệt với phiên bản lưu trữ. Ví dụ,
một đối tượng API `v1alpha1` và một đối tượng API `v1beta1` của cùng một
tài nguyên (Resource) sẽ được mã hóa giống nhau trong kho lưu trữ, miễn là
phiên bản lưu trữ không bị thay đổi giữa hai đối tượng đó.

## Ánh xạ giữa phiên bản lưu trữ và tài nguyên (Storage version to resource mapping)

Mỗi tài nguyên sẽ có 1 phiên bản lưu trữ đang hoạt động tại bất kỳ thời điểm nào,
nghĩa là mọi thao tác ghi lên một đối tượng sẽ lưu đối tượng đó ở phiên bản lưu trữ
ấy. Tuy nhiên, phiên bản lưu trữ có thể được cập nhật, khiến các đối tượng có thể
được lưu ở những phiên bản khác nhau. Một đối tượng chỉ được lưu ở một phiên bản
lưu trữ duy nhất tại bất kỳ thời điểm nào.

Các thao tác đọc từ API Server sẽ chuyển đổi dữ liệu đã lưu sang biểu diễn API
của đối tượng. Nhờ đó, các phiên bản lưu trữ cũ có thể tồn tại vô thời hạn
miễn là không có cập nhật nào xảy ra với đối tượng. Ngược lại, các thao tác ghi
sẽ chuyển đối tượng đã lưu sang biểu diễn mới khi cập nhật.

## Phiên bản lưu trữ cho custom resource (Storage versions for custom resources) {#CustomResourceDefinition-storage-version}

[Custom resource](179-custom-resources-vi.md#storage)
được định nghĩa một cách động (dynamic), và do đó khác với các loại (type) có sẵn
của Kubernetes ở khía cạnh phiên bản lưu trữ. Các đối tượng có sẵn (builtin)
nói chung có cách mã hóa lưu trữ được định nghĩa tách biệt khỏi các loại API
của chúng, trong đó đối tượng được lưu đóng vai trò như một trục trung tâm (hub)
và phiên bản cụ thể của tài nguyên không quan trọng, ngoài việc là một trường
trong lược đồ đối tượng.

Tuy nhiên, với custom resource, một phiên bản nhất định của tài nguyên phải được
đặt làm phiên bản lưu trữ. Lược đồ được định nghĩa bởi phiên bản cụ thể đó của
custom resource sẽ được dùng làm cách mã hóa của tài nguyên ở tầng lưu trữ. Xem
[bộ tính năng CRD nâng cao](179-custom-resources-vi.md#advanced-features-and-flexibility)
để biết thông tin chi tiết hơn về cách thiết lập API và quản lý phiên bản.

Ví dụ, hãy xem CustomResourceDefinition sau cho _crontabs_:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.example.com
spec:
  group: example.com
  # danh sách các phiên bản được CustomResourceDefinition này hỗ trợ
  versions:
  - name: v1beta1
    # Mỗi phiên bản có thể được bật/tắt bằng cờ Served.
    served: true
    # Một và chỉ một phiên bản phải được đánh dấu là phiên bản lưu trữ.
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          host:
            type: string
          port:
            type: string
  - name: v1
    served: true
    storage: false
    schema:
      openAPIV3Schema:
        type: object
        properties:
          host:
            type: string
          port:
            type: string
          time:
            type: string
  conversion:
    strategy: None
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

Định nghĩa API `v1beta1` được dùng làm phiên bản lưu trữ, nghĩa là mọi thao tác
cập nhật hoặc tạo mới `crontabs` sẽ được lưu với lược đồ đối tượng của API
`v1beta1`. Trong trường hợp này, điều đó thực tế có nghĩa là đối tượng API `v1`
sẽ không bao giờ lưu được trường `time`, vì trường này không thuộc định nghĩa
lưu trữ. Lược đồ này được dùng ở tầng lưu trữ như chính cách mã hóa nhị phân
của đối tượng. Việc cố gắng đặt hai phiên bản làm phiên bản lưu trữ cùng lúc
bị coi là không hợp lệ, vì như vậy đồng nghĩa với việc hai lược đồ dữ liệu
cùng lúc được coi là những cách hợp lệ để lưu các đối tượng.

Khi thay đổi phiên bản được dùng để lưu trữ, phiên bản API đó sẽ được dùng để
lưu mọi CR mới hoặc được cập nhật. Việc watch hoặc get đối tượng vẫn sử dụng
được đối tượng, nhưng chỉ chuyển đổi đối tượng từ phiên bản lưu trữ cũ chứ
không ảnh hưởng tới đối tượng. Chỉ thao tác cập nhật hoặc tạo mới mới có
tác dụng và sử dụng phiên bản lưu trữ mới được định nghĩa.

## Phiên bản lưu trữ liên quan thế nào tới mã hóa dữ liệu lưu trữ (How storage versions are relevant to encryption at rest)

Có các công cụ để [mã hóa kho lưu trữ dữ liệu tĩnh](213-kms-provider-vi.md)
của một cluster, đặc biệt cho các secret của cluster. Điều này bổ sung thêm
một lớp bảo vệ chống rò rỉ dữ liệu (data exfiltration), vì dữ liệu thực sự
được lưu trong cluster đã được mã hóa. Nghĩa là API Server thực tế sẽ giải mã
dữ liệu khi lấy chúng từ kho lưu trữ. API Server phải có khóa (key) cho
phiên bản lưu trữ đó để giải mã đối tượng một cách chính xác.

Trong trường hợp này, phiên bản lưu trữ không chỉ đơn thuần là cách mã hóa
nhị phân của đối tượng. Miễn là những gì được lưu có thể được chuyển đổi
bằng cách nào đó thành đối tượng API, nó có thể được dùng như một phiên bản lưu trữ.

## Di chuyển sang một phiên bản lưu trữ khác (Migrating to a different storage version)

Việc một tài nguyên đơn lẻ có nhiều phiên bản lưu trữ có thể gây ra vấn đề cho
người quản trị cluster. Người quản trị cluster không thể gỡ bỏ các phiên bản cũ
của một API cho CRD — vốn có thể không còn được hỗ trợ — cho đến khi chắc chắn
rằng không còn đối tượng nào đang dùng phiên bản lưu trữ gắn với nó. Với số lượng
đối tượng lớn và một góc nhìn không rõ ràng về việc đối tượng nào là mới, đối tượng
nào vẫn được lưu bằng phiên bản lưu trữ cũ, rất khó để biết khi nào một phiên bản
có thể được gỡ bỏ một cách an toàn. Nếu một phiên bản bị gỡ bỏ quá sớm, điều đó
có thể đồng nghĩa với việc hoàn toàn không thể đọc được đối tượng.

Một vấn đề quan trọng khác là việc sử dụng khóa mã hóa như đã trình bày ở phần
trên. Vì một tài nguyên phải thực sự đang được sử dụng thì phiên bản lưu trữ
mới được cập nhật, nên khi thực hiện xoay vòng khóa (key rotation), cả khóa
mã hóa cũ lẫn khóa mã hóa mới đều phải được duy trì sử dụng cho đến khi
người quản trị chắc chắn rằng tất cả các đối tượng đã được ghi ít nhất một lần.
Điều này gây ra cả rủi ro bảo mật lẫn vấn đề về tính tiện dụng, vì cho tới lúc đó
một khóa chưa thể được loại bỏ hoàn toàn khỏi việc sử dụng.

Xem [di chuyển phiên bản lưu trữ (storage version migration)](323-storage-version-migration-vi.md)
để có các ví dụ về cách chạy một cuộc di chuyển nhằm đảm bảo tất cả các đối tượng
đều đang dùng phiên bản lưu trữ mới hơn mà không cần can thiệp thủ công.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Phiên bản API của một object và phiên bản lưu trữ của nó có phải một không?
2. Một object được tạo từ hai năm trước và chưa ai sửa. Nó đang nằm ở phiên bản lưu trữ nào —
   phiên bản hiện tại hay phiên bản lúc nó được tạo? Vì sao?
3. Bạn muốn gỡ bỏ một phiên bản API cũ khỏi cluster. Vì sao không thể gỡ ngay cả khi không ai
   còn gọi phiên bản đó nữa?
4. Cluster đang bật mã hóa dữ liệu lưu trữ và bạn muốn xoay khóa. Vì sao chưa thể vứt khóa cũ
   đi ngay sau khi đưa khóa mới vào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Chúng hoàn toàn tách biệt. Bài nêu ví dụ: một object API `v1alpha1` và một object
   API `v1beta1` của cùng resource sẽ được mã hóa **giống nhau** trong kho lưu trữ, miễn là
   phiên bản lưu trữ không đổi giữa hai thời điểm đó.
2. **Phiên bản lúc nó được ghi lần cuối.** Chỉ thao tác **ghi** mới chuyển object sang phiên
   bản lưu trữ hiện hành; thao tác **đọc** chỉ chuyển đổi khi trả về cho client. Nên phiên bản
   lưu trữ cũ có thể tồn tại vô thời hạn.
3. Vì có thể vẫn còn **object đang được lưu ở phiên bản đó**. Không ai gọi API không có nghĩa
   là dữ liệu đã được ghi lại. Gỡ sớm đồng nghĩa với việc **hoàn toàn không đọc được** những
   object ấy nữa.
4. Vì API server cần **khóa của phiên bản lưu trữ tương ứng** để giải mã. Một object chỉ được
   mã hóa lại bằng khóa mới khi nó **được ghi lại**. Phải giữ cả hai khóa cho tới khi chắc
   chắn mọi object đã được ghi ít nhất một lần — đó vừa là rủi ro bảo mật vừa là phiền toái
   vận hành, và là lý do tồn tại của storage version migration.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

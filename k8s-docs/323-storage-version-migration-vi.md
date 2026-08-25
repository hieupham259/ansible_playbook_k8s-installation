# Di trú object Kubernetes bằng Storage Version Migration (Migrate Kubernetes Objects Using Storage Version Migration)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/storage-version-migration/>
>
> Trang này hướng dẫn cách dùng Storage Version Migration để bảo đảm mọi object đã lưu trong
> etcd của một resource được ghi lại theo storage version mới hoặc khóa mã hóa mới.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** tài liệu tra cứu thuộc nhánh `/docs/tasks/`
([Checkpoint tiếp nối](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)) — bài này là
bản thực hành cho khái niệm ở bài [32 — Phiên bản lưu trữ](32-storage-version-vi.md), đồng thời
nối tiếp cặp bài mã hóa at rest của giai đoạn 22: [208 — Encrypting Confidential Data at
Rest](208-encrypt-data-vi.md) và [213 — KMS provider](213-kms-provider-vi.md) đã dạy cách
**đổi khóa**, còn bài này dạy cách **ép ghi lại dữ liệu cũ** theo khóa mới.

Tính năng này ở trạng thái beta và cần bật feature gate `StorageVersionMigrator` cùng runtime
config `storagemigration.k8s.io/v1beta1` trên API server — trên cluster lab bạn phải cấu hình
lại control plane (kỹ thuật của bài [196](196-configure-feature-gates-vi.md) và
[220](220-kubeadm-reconfigure-vi.md)) mới thực hành được; lần đọc này ưu tiên hiểu cơ chế.

**Phải hiểu ở lần đọc này:**

- Vì sao phải "ghi lại chủ động" dữ liệu trong etcd: đổi storage schema (v1 → v2) hay đổi khóa
  mã hóa at rest chỉ tác động tới bản ghi **được ghi mới** — object cũ nằm im theo định dạng
  cũ cho tới khi có thứ ghi lại nó.
- StorageVersionMigration là một API object: bạn khai `spec.resource` (group + resource) và
  control plane ghi lại toàn bộ object của resource đó; điều kiện tiên quyết là resource có
  resource version dạng số nguyên (mọi resource Kubernetes và CRD đều có, aggregated API thì
  có thể không).
- Luồng re-encrypt Secret: thêm khóa mới lên **đầu** danh sách provider trong
  EncryptionConfiguration → tạo StorageVersionMigration cho resource `secrets` → chờ condition
  `Succeeded` → xác minh prefix `k8s:enc:aescbc:v1:key2` trong etcd.
- Luồng CRD: thêm v2 với `storage: true` (v1 hạ xuống `storage: false`) kèm conversion
  webhook; CR tạo từ trước **vẫn nằm ở v1 trong etcd** cho tới khi StorageVersionMigration
  chạy xong, sau đó `status.storedVersions` của CRD chỉ còn v2.
- Cách theo dõi migration qua `.status.conditions`, ví dụ
  `kubectl wait --for=condition=Succeeded storageversionmigration...`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Các lệnh `etcdctl get ... \| hexdump -C` đọc thẳng etcd | cần đối số kết nối etcd và thao tác đã học ở giai đoạn 19 | bài [197 — Vận hành etcd](197-configure-upgrade-etcd-vi.md) và mục xác minh của bài [213](213-kms-provider-vi.md) |
| Chi tiết viết conversion webhook cho CRD (`clientConfig`, `caBundle`) | thuộc phần mở rộng API, ngoài lộ trình admin | trang [Versions in CustomResourceDefinitions](377-custom-resource-definition-versioning-vi.md) khi cần |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]`

Kubernetes dựa vào việc dữ liệu API được chủ động ghi lại (re-write) để hỗ trợ một số hoạt
động bảo trì liên quan đến lưu trữ at rest. Hai ví dụ nổi bật là schema có phiên bản của các
resource được lưu trữ (tức là storage schema ưu tiên của một resource đổi từ v1 sang v2) và
mã hóa at rest (tức là ghi lại dữ liệu cũ dựa trên thay đổi trong cách dữ liệu cần được mã
hóa).

Việc chạy storage version migration cho phép bảo đảm rằng mọi object của một Resource đã được
di trú khỏi storage version cũ. Yêu cầu để chạy một storage migration là bảo đảm Resource đó
có resource version dạng số nguyên. Mọi Resource của Kubernetes và các CRD đều được bảo đảm có
thuộc tính này, nhưng migration sẽ thất bại nếu điều kiện đó không được thỏa mãn, chẳng hạn
với các aggregated API.

## Trước khi bạn bắt đầu (Before you begin)

Cài đặt [`kubectl`](185-tools-vi.md).

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.30 hoặc mới hơn. Để kiểm tra phiên bản, nhập
`kubectl version`.

Bảo đảm cluster của bạn đã bật
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#StorageVersionMigrator)
`StorageVersionMigrator`. Bạn cần quyền quản trị control plane để thực hiện thay đổi đó.

Bật REST API của storage version migration bằng cách đặt runtime config
`storagemigration.k8s.io/v1beta1` thành `true` cho API server. Để biết thêm về cách làm điều
đó, hãy đọc [bật hoặc tắt một Kubernetes API](207-enable-disable-api-vi.md).

## Mã hóa lại Secret của Kubernetes bằng storage version migration (Re-encrypt Kubernetes secrets using storage version migration)

- Trước tiên, [cấu hình KMS provider](213-kms-provider-vi.md) để mã hóa dữ liệu at rest trong
  etcd bằng cấu hình mã hóa sau.

  ```yaml
  kind: EncryptionConfiguration
  apiVersion: apiserver.config.k8s.io/v1
  resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: c2VjcmV0IGlzIHNlY3VyZQ==
  ```

  Nhớ bật tự động nạp lại (reload) file cấu hình mã hóa bằng cách đặt
  `--encryption-provider-config-automatic-reload` thành true.

- Tạo một Secret bằng kubectl.

  ```shell
  kubectl create secret generic my-secret --from-literal=key1=supersecret
  ```

- [Xác minh](213-kms-provider-vi.md#xác-minh-dữ-liệu-đã-được-mã-hóa-verifying-that-the-data-is-encrypted)
  rằng dữ liệu đã tuần tự hóa (serialized) của object Secret đó có prefix
  `k8s:enc:aescbc:v1:key1`.

- Cập nhật file cấu hình mã hóa như sau để xoay (rotate) khóa mã hóa.

  ```yaml
  kind: EncryptionConfiguration
  apiVersion: apiserver.config.k8s.io/v1
  resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key2
          secret: c2VjcmV0IGlzIHNlY3VyZSwgaXMgaXQ/
    - aescbc:
        keys:
        - name: key1
          secret: c2VjcmV0IGlzIHNlY3VyZQ==
  ```

- Để bảo đảm secret `my-secret` đã tạo trước đó được mã hóa lại bằng khóa mới `key2`, bạn sẽ
  dùng _Storage Version Migration_.

- Tạo một manifest StorageVersionMigration tên là `migrate-secret.yaml` như sau:

  ```yaml
  kind: StorageVersionMigration
  apiVersion: storagemigration.k8s.io/v1beta1
  metadata:
    name: secrets-migration
  spec:
    resource:
      group: ""
      resource: secrets
  ```

  Tạo object bằng `kubectl` như sau:

  ```shell
  kubectl apply -f migrate-secret.yaml
  ```

- Theo dõi việc di trú các Secret bằng cách kiểm tra `.status` của StorageVersionMigration.
  Một migration thành công sẽ có condition `Succeeded` được đặt thành true. Lấy object
  StorageVersionMigration như sau:

  ```shell
  kubectl wait --for=condition=Succeeded storageversionmigration.storagemigration.k8s.io/secrets-migration
  ```

  Output tương tự như sau:

  ```yaml
  kind: StorageVersionMigration
  apiVersion: storagemigration.k8s.io/v1beta1
  metadata:
    name: secrets-migration
    uid: 628f6922-a9cb-4514-b076-12d3c178967c
    resourceVersion: "90"
    creationTimestamp: "2024-03-12T20:29:45Z"
  spec:
    resource:
      group: ""
      resource: secrets
  status:
    conditions:
    - type: Running
      status: "False"
      lastUpdateTime: "2024-03-12T20:29:46Z"
      reason: StorageVersionMigrationInProgress
    - type: Succeeded
      status: "True"
      lastUpdateTime: "2024-03-12T20:29:46Z"
      reason: StorageVersionMigrationSucceeded
    resourceVersion: "84"
  ```

- [Xác minh](213-kms-provider-vi.md#xác-minh-dữ-liệu-đã-được-mã-hóa-verifying-that-the-data-is-encrypted)
  rằng secret đã lưu giờ đây có prefix `k8s:enc:aescbc:v1:key2`.

## Cập nhật storage schema ưu tiên của một CRD (Update the preferred storage schema of a CRD)

Xét kịch bản: một CustomResourceDefinition (CRD) được tạo để phục vụ các custom resource (CR)
và được đặt làm storage schema ưu tiên. Khi đến lúc giới thiệu v2 của CRD, có thể thêm v2 ở
chế độ chỉ phục vụ (serving only) kèm một conversion webhook. Cách này cho phép chuyển đổi êm
hơn: người dùng có thể tạo CR bằng schema v1 hoặc v2, với webhook đảm nhiệm việc chuyển đổi
schema cần thiết giữa hai phiên bản. Trước khi đặt v2 làm storage schema version ưu tiên, điều
quan trọng là bảo đảm mọi CR hiện có đang lưu ở v1 được di trú sang v2. Việc di trú này có thể
thực hiện qua _Storage Version Migration_ để di trú toàn bộ CR từ v1 sang v2.

- Tạo một manifest cho CRD, tên là `test-crd.yaml`, như sau:

  ```yaml
  apiVersion: apiextensions.k8s.io/v1
  kind: CustomResourceDefinition
  metadata:
    name: selfierequests.example.com
  spec:
    group: example.com
    names:
      plural: selfierequests
      singular: selfierequest
      kind: SelfieRequest
      listKind: SelfieRequestList
    scope: Namespaced
    versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            hostPort:
              type: string
    conversion:
      strategy: Webhook
      webhook:
        clientConfig:
          url: "https://127.0.0.1:9443/crdconvert"
          caBundle: <CABundle info>
      conversionReviewVersions:
      - v1
      - v2
  ```

  Storage version tại thời điểm này phải là `v1`, xác nhận bằng cách chạy:
  ```shell
  kubectl get crd selfierequests.example.com -o jsonpath='{.spec.versions[?(@.storage==true)].name}'
  ```

  Tạo CRD bằng kubectl:

  ```shell
  kubectl apply -f test-crd.yaml
  ```

- Tạo một manifest cho một testcrd ví dụ. Đặt tên manifest là `cr1.yaml` và dùng nội dung sau:

  ```yaml
  apiVersion: example.com/v1
  kind: SelfieRequest
  metadata:
    name: cr1
    namespace: default
  ```

  Tạo CR bằng kubectl:

  ```shell
  kubectl apply -f cr1.yaml
  ```

- Xác minh rằng CR được ghi và lưu ở v1 bằng cách lấy object từ etcd.

  ```shell
  ETCDCTL_API=3 etcdctl get /kubernetes.io/example.com/testcrds/default/cr1 [...] | hexdump -C
  ```

  trong đó `[...]` chứa các đối số bổ sung để kết nối tới etcd server.

- Cập nhật CRD `test-crd.yaml` để thêm phiên bản v2 ở chế độ vừa phục vụ vừa lưu trữ (serving
  và storage), còn v1 chỉ phục vụ (serving only), như sau:

  ```yaml
  apiVersion: apiextensions.k8s.io/v1
  kind: CustomResourceDefinition
  metadata:
  name: selfierequests.example.com
  spec:
    group: example.com
    names:
      plural: selfierequests
      singular: selfierequest
      kind: SelfieRequest
      listKind: SelfieRequestList
    scope: Namespaced
    versions:
      - name: v2
        served: true
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
              hostPort:
                type: string
    conversion:
      strategy: Webhook
      webhook:
        clientConfig:
          url: "https://127.0.0.1:9443/crdconvert"
          caBundle: <CABundle info>
        conversionReviewVersions:
          - v1
          - v2
  ```

  Storage version bây giờ phải là `v2`, xác nhận điều này:
  ```shell
  kubectl get crd selfierequests.example.com -o jsonpath='{.spec.versions[?(@.storage==true)].name}'
  ```

  Cập nhật CRD bằng kubectl:

  ```shell
  kubectl apply -f test-crd.yaml
  ```

- Tạo file resource CR với tên `cr2.yaml` như sau:

  ```yaml
  apiVersion: example.com/v2
  kind: SelfieRequest
  metadata:
    name: cr2
    namespace: default
  ```

- Tạo CR bằng kubectl:

  ```shell
  kubectl apply -f cr2.yaml
  ```

- Xác minh rằng CR được ghi và lưu ở v2 bằng cách lấy object từ etcd.

  ```shell
  ETCDCTL_API=3 etcdctl get /kubernetes.io/example.com/testcrds/default/cr2 [...] | hexdump -C
  ```

  trong đó `[...]` chứa các đối số bổ sung để kết nối tới etcd server.

- Tạo một manifest StorageVersionMigration tên là `migrate-crd.yaml`, với nội dung như sau:

  ```yaml
  kind: StorageVersionMigration
  apiVersion: storagemigration.k8s.io/v1beta1
  metadata:
    name: crdsvm
  spec:
    resource:
      group: example.com
      resource: selfierequests
  ```

  Tạo object bằng _kubectl_ như sau:

  ```shell
  kubectl apply -f migrate-crd.yaml
  ```

- Theo dõi việc di trú bằng status. Migration thành công sẽ có condition `Succeeded` được đặt
  thành "True" trong field status. Lấy migration resource như sau:

  ```shell
  kubectl get storageversionmigration.storagemigration.k8s.io/crdsvm -o yaml
  ```

  Output tương tự như sau:

  ```yaml
  kind: StorageVersionMigration
  apiVersion: storagemigration.k8s.io/v1beta1
  metadata:
    name: crdsvm
    uid: 13062fe4-32d7-47cc-9528-5067fa0c6ac8
    resourceVersion: "111"
    creationTimestamp: "2024-03-12T22:40:01Z"
  spec:
    resource:
      group: example.com
      resource: testcrds
  status:
    conditions:
      - type: Running
        status: "False"
        lastUpdateTime: "2024-03-12T22:40:03Z"
        reason: StorageVersionMigrationInProgress
      - type: Succeeded
        status: "True"
        lastUpdateTime: "2024-03-12T22:40:03Z"
        reason: StorageVersionMigrationSucceeded
    resourceVersion: "106"
  ```

- Xác minh rằng cr1 tạo trước đó giờ đây được ghi và lưu ở v2 bằng cách lấy object từ etcd.

  ```shell
  ETCDCTL_API=3 etcdctl get /kubernetes.io/example.com/testcrds/default/cr1 [...] | hexdump -C
  ```

  trong đó `[...]` chứa các đối số bổ sung để kết nối tới etcd server.

- Đồng thời xác minh rằng status về stored version của CRD giờ đây chỉ còn v2:

  ```shell
  kubectl get crd testcrds.example.com -o yaml
  ```

  Output tương tự như sau:

  ```yaml
  kind: CustomResourceDefinition
  apiVersion: apiextensions.k8s.io/v1
  metadata:
    name: testcrds.example.com
  spec:
    group: example.com
    names:
      kind: TestCRD
      plural: testcrds
    scope: Namespaced
    versions:
      - name: v1
        served: true
        storage: false
      - name: v2
        served: true
        storage: true
  status:
    acceptedNames:
      kind: TestCRD
      plural: testcrds
    conditions:
      - type: Established
        status: "True"
    storedVersions:
      - v2
  ```

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn này.

1. Storage Version Migration giải quyết vấn đề gì, và hai tình huống bảo trì tiêu biểu nào
   cần đến nó?
2. Câu bẫy: bạn thêm `key2` lên đầu danh sách provider trong EncryptionConfiguration và API
   server đã tự nạp lại cấu hình. Secret `my-secret` tạo từ trước giờ đã được mã hóa bằng
   `key2` chưa? Vì sao?
3. Trong kịch bản CRD, sau khi bạn `apply` bản CRD mới có v2 `storage: true` và v1
   `storage: false`, object `cr1` (tạo từ hồi v1) đang nằm trong etcd ở phiên bản nào? Khi
   nào nó mới chuyển sang v2?
4. Muốn biết một StorageVersionMigration đã hoàn tất thành công, bạn nhìn vào đâu và dùng
   lệnh nào để chờ nó?
5. Trước khi thử tính năng này trên cluster lab của bạn, phải thay đổi những gì trên control
   plane, và điều kiện nào của resource khiến migration có thể thất bại?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó bảo đảm **mọi object đã lưu của một resource được chủ động ghi lại**, để không còn
   object nào nằm ở định dạng lưu trữ cũ. Hai tình huống tiêu biểu: **đổi storage schema có
   phiên bản** của resource (v1 → v2) và **mã hóa at rest** (ghi lại dữ liệu cũ theo cách mã
   hóa mới, ví dụ sau khi xoay khóa).
2. **Chưa.** Đổi cấu hình chỉ ảnh hưởng các lần ghi mới; `my-secret` vẫn nằm trong etcd với
   prefix `k8s:enc:aescbc:v1:key1` cho tới khi có thứ ghi lại nó. Đó chính là lý do phải tạo
   một object StorageVersionMigration cho resource `secrets` — sau khi nó `Succeeded`, dữ
   liệu lưu mới mang prefix `k8s:enc:aescbc:v1:key2`. Trực giác "đổi cấu hình xong là dữ liệu
   đổi theo" sai vì mã hóa được áp lúc ghi, không phải một job chạy ngầm tự động.
3. `cr1` **vẫn được lưu ở v1** trong etcd — đặt `storage: true` cho v2 chỉ quyết định phiên
   bản dùng cho các lần ghi từ nay về sau. Nó chỉ chuyển sang v2 **sau khi
   StorageVersionMigration cho resource đó chạy thành công**; lúc ấy
   `status.storedVersions` của CRD mới chỉ còn `v2`.
4. Nhìn vào `.status.conditions` của object StorageVersionMigration: migration thành công có
   condition `Succeeded` với status `"True"`. Lệnh chờ:
   `kubectl wait --for=condition=Succeeded storageversionmigration.storagemigration.k8s.io/<tên>`.
5. Phải **bật feature gate `StorageVersionMigrator`** và **đặt runtime config
   `storagemigration.k8s.io/v1beta1=true` cho API server** — cả hai đều cần quyền quản trị
   control plane. Migration đòi hỏi resource có **resource version dạng số nguyên**: mọi
   resource Kubernetes và CRD đều thỏa mãn, nhưng aggregated API có thể không, và khi đó
   migration sẽ thất bại.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

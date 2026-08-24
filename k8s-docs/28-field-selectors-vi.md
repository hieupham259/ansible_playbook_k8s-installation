# Field selector (Field Selectors)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1b](00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl),
bài 9/9 · Kiểm chứng ở [Lab 1b](labs/LAB-1B-OBJECT-LABEL-KUBECTL-VA-KUBECONFIG.md).

Bài cuối nhóm 1b, và ngắn nhất. Đọc nó như phần bổ sung cho bài [18](18-labels-vi.md): cùng là
lọc, nhưng lọc theo thứ khác.

**Phải hiểu ở lần đọc này:**

- Field selector lọc theo **giá trị field của resource**; label selector lọc theo **label bạn
  tự gắn**. Hai cơ chế độc lập, dùng chung được trong một lệnh.
- Mọi resource đều hỗ trợ `metadata.name` và `metadata.namespace`; ngoài hai cái đó thì **tùy
  từng kind**.
- Chỉ có `=`, `==`, `!=`. **Không** có `in`, `notin`, `exists` — khác hẳn label selector.
- Dấu phẩy là AND, giống label selector.
- Dùng field không được hỗ trợ thì API server **báo lỗi**, chứ không im lặng trả về rỗng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bảng đầy đủ field theo từng kind | là tài liệu tra cứu | tra khi cần |
| Các field của Pod như `status.phase`, `spec.nodeName` | chưa học Pod và lập lịch | giai đoạn 3 và 7 |
| `selectableFields` của CustomResourceDefinition | thuộc chủ đề mở rộng API | giai đoạn 14 |

---

_Field selector_ (bộ chọn trường) cho phép bạn chọn các object Kubernetes dựa trên
giá trị của một hoặc nhiều trường (field) của resource. Dưới đây là một vài ví dụ về truy vấn field selector:

* `metadata.name=my-service`
* `metadata.namespace!=default`
* `status.phase=Pending`

Lệnh `kubectl` sau chọn tất cả các Pod có giá trị của trường [`status.phase`](47-pod-lifecycle-vi.md#pod-phase) là `Running`:

```shell
kubectl get pods --field-selector status.phase=Running
```

> **Ghi chú:** Field selector về bản chất là các *bộ lọc* (filter) resource. Mặc định, không có selector/filter nào được áp dụng, nghĩa là tất cả resource thuộc loại được chỉ định đều được chọn. Điều này khiến hai truy vấn `kubectl get pods` và `kubectl get pods --field-selector ""` tương đương nhau.

## Các trường được hỗ trợ (Supported fields)

Các field selector được hỗ trợ khác nhau tùy theo loại resource của Kubernetes. Mọi loại resource đều hỗ trợ hai trường `metadata.name` và `metadata.namespace`. Dùng field selector không được hỗ trợ sẽ gây ra lỗi. Ví dụ:

```shell
kubectl get ingress --field-selector foo.bar=baz
```
```
Error from server (BadRequest): Unable to find "ingresses" that match label selector "", field selector "foo.bar=baz": "foo.bar" is not a known field selector: only "metadata.name", "metadata.namespace"
```

### Danh sách các trường được hỗ trợ (List of supported fields)

| Kind                      | Các trường                                                                                                                                                                                                                                                      |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pod                       | `spec.nodeName`<br>`spec.restartPolicy`<br>`spec.schedulerName`<br>`spec.serviceAccountName`<br>`spec.hostNetwork`<br>`status.phase`<br>`status.podIP`<br>`status.podIPs`<br>`status.nominatedNodeName`                                                                            |
| Event                     | `involvedObject.kind`<br>`involvedObject.namespace`<br>`involvedObject.name`<br>`involvedObject.uid`<br>`involvedObject.apiVersion`<br>`involvedObject.resourceVersion`<br>`involvedObject.fieldPath`<br>`reason`<br>`reportingComponent`<br>`source`<br>`type` |
| Secret                    | `type`                                                                                                                                                                                                                                                          |
| Service                   | `spec.clusterIP`<br>`spec.type`                                                                                                                                                                                                                                 |
| Namespace                 | `status.phase`                                                                                                                                                                                                                                                  |
| ReplicaSet                | `status.replicas`                                                                                                                                                                                                                                               |
| ReplicationController     | `status.replicas`                                                                                                                                                                                                                                               |
| Job                       | `status.successful`                                                                                                                                                                                                                                             |
| Node                      | `spec.unschedulable`                                                                                                                                                                                                                                            |
| CertificateSigningRequest | `spec.signerName`                                                                                                                                                                                                                                               |

### Các trường của custom resource (Custom resources fields)

Mọi loại custom resource đều hỗ trợ hai trường `metadata.name` và `metadata.namespace`.

Ngoài ra, trường `spec.versions[*].selectableFields` của một CustomResourceDefinition
khai báo những trường nào khác trong một custom resource có thể được dùng trong field selector. Xem [các trường có thể chọn (selectable fields) cho custom resource](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#crd-selectable-fields)
để biết thêm thông tin về cách dùng field selector với CustomResourceDefinition.

## Các toán tử được hỗ trợ (Supported operators)

Bạn có thể dùng các toán tử `=`, `==` và `!=` với field selector (`=` và `==` có ý nghĩa như nhau). Ví dụ, lệnh `kubectl` sau chọn tất cả các Service của Kubernetes không nằm trong namespace `default`:

```shell
kubectl get services  --all-namespaces --field-selector metadata.namespace!=default
```

> **Ghi chú:** [Các toán tử dựa trên tập hợp (set-based operators)](18-labels-vi.md#set-based-requirement)
> (`in`, `notin`, `exists`) không được hỗ trợ cho field selector.

## Nối chuỗi các selector (Chained selectors)

Giống như [label](./18-labels-vi.md) và các loại selector khác, các field selector có thể được nối với nhau thành một danh sách phân tách bằng dấu phẩy. Lệnh `kubectl` sau chọn tất cả các Pod có `status.phase` khác `Running` và có trường `spec.restartPolicy` bằng `Always`:

```shell
kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always
```

## Nhiều loại resource (Multiple resource types)

Bạn có thể dùng field selector trên nhiều loại resource cùng lúc. Lệnh `kubectl` sau chọn tất cả các Statefulset và Service không nằm trong namespace `default`:

```shell
kubectl get statefulsets,services --all-namespaces --field-selector metadata.namespace!=default
```

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Nói bằng một câu: label selector lọc theo cái gì, field selector lọc theo cái gì?
2. `in` và `notin` dùng được với field selector không?
3. Bạn gõ một field selector mà resource đó không hỗ trợ. API server trả về danh sách rỗng hay
   báo lỗi? Vì sao hành vi này khác với việc gõ một label không tồn tại?
4. Hai cách sau khác nhau chỗ nào về mặt tải lên API server: lọc bằng `--field-selector`, so
   với lấy hết rồi lọc bằng `-o jsonpath` ở phía client?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Label selector lọc theo **label do bạn gắn vào metadata**; field selector lọc theo **giá trị
   các field có sẵn của chính resource** (`metadata.name`, `status.phase`, `spec.nodeName`…).
2. **Không.** Field selector chỉ hỗ trợ `=`, `==` và `!=`. Toán tử dựa trên tập hợp là đặc
   quyền của label selector.
3. **Báo lỗi** `BadRequest`, kèm danh sách các field selector hợp lệ. Khác với label: label
   nào cũng là chuỗi tùy ý nên API server không có cách nào biết bạn gõ nhầm, chỉ trả về tập
   rỗng. Field thì có schema, nên gõ sai là lỗi phát hiện được.
4. `--field-selector` lọc **ở phía server**, nên API server chỉ trả về phần đã khớp. Lọc bằng
   `jsonpath` thì toàn bộ danh sách vẫn được truyền về client rồi mới bị cắt bớt — tốn băng
   thông và tốn công API server hơn, đáng kể khi cluster lớn.

</details>

Đây là bài cuối của nhóm 1b. Trả lời được hết các câu trong chín bài của nhóm thì bạn sẵn sàng
vào Lab 1b.

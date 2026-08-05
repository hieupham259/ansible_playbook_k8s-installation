# Field selector (Field Selectors)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/>

_Field selector_ (bộ chọn trường) cho phép bạn chọn các object Kubernetes dựa trên
giá trị của một hoặc nhiều trường (field) của resource. Dưới đây là một vài ví dụ về truy vấn field selector:

* `metadata.name=my-service`
* `metadata.namespace!=default`
* `status.phase=Pending`

Lệnh `kubectl` sau chọn tất cả các Pod có giá trị của trường [`status.phase`](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase) là `Running`:

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

> **Ghi chú:** [Các toán tử dựa trên tập hợp (set-based operators)](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#set-based-requirement)
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

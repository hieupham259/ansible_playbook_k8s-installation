# Truy cập metadata thiết bị DRA (Access DRA Device Metadata)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/access-dra-device-metadata/
>
> Trang này hướng dẫn cách truy cập metadata của thiết bị từ các container sử dụng cấp phát
> tài nguyên động (dynamic resource allocation — DRA), bằng cách đọc các file JSON tại những
> đường dẫn quy ước bên trong container.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13 — Lập lịch và workload nâng cao](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao)
→ dòng **Thực hành**, bài 3/5 · Kiểm chứng ở [Lab 13 — DRA](labs/LAB-13-DRA.md) phần B5.4.

Bài này ở mức **alpha** và đòi Kubernetes server v1.36 trở lên — khác hai bài kế
[270](270-allocate-devices-dra-vi.md) và [271](271-set-up-dra-cluster-vi.md) vốn đã stable. Cluster
lab không có GPU và không có DRA driver, nên phần B5.4 của Lab 13 chỉ kiểm được **vế phủ định**:
container không yêu cầu thiết bị thì không có thư mục metadata. Đọc bài này để nắm quy ước đường
dẫn, chưa phải để chạy.

**Phải hiểu ở lần đọc này:**

- Metadata thiết bị đến với workload qua **file JSON tại đường dẫn quy ước bên trong container**,
  không qua API — ứng dụng chỉ cần `find` và `cat` là đọc được, không cần quyền nào trên apiserver.
- Với một ResourceClaim được tham chiếu trực tiếp, đường dẫn là
  `/var/run/kubernetes.io/dra-device-attributes/resourceclaims/<claimName>/<requestName>/<driverName>-metadata.json`
  — ba mảnh đường dẫn tương ứng tên claim, tên request và tên driver.
- Với ResourceClaimTemplate thì Kubernetes sinh một ResourceClaim cho **mỗi Pod** và **tên claim
  sinh ra không đoán trước được**, nên đường dẫn đổi sang
  `.../resourceclaimtemplates/<podClaimName>/<requestName>/...`; `<podClaimName>` chính là field
  `name` trong `spec.resourceClaims[]` của Pod, và JSON cũng mang field `podClaimName` ghi lại ánh
  xạ đó.
- Điều kiện để có metadata không chỉ là "cluster đã bật DRA": mục *Trước khi bạn bắt đầu* đòi thêm
  rằng **DRA driver phải hỗ trợ metadata thiết bị** — driver bật `EnableDeviceMetadata` và
  `MetadataVersions` khi khởi động kubelet plugin.
- Cách đọc metadata một cách bền vững qua nhiều phiên bản: file JSON có field `apiVersion` cho biết
  phiên bản lược đồ, và ứng dụng viết bằng ngôn ngữ khác Go phải **kiểm field đó trước khi parse**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ba hàm Go của package `devicemetadata` (`ReadResourceClaimMetadata`, `ReadResourceClaimTemplateMetadata`, `ReadResourceClaimMetadataWithDriverName`) | là việc của người viết ứng dụng tiêu thụ thiết bị, không phải của quản trị viên | không cần cho lộ trình; thứ phải nhớ là quy ước đường dẫn ở hai mục *Truy cập metadata thiết bị bằng ResourceClaim* và *…bằng ResourceClaimTemplate* của chính bài này |
| Nội dung thật bên trong file JSON — model, phiên bản driver, UUID thiết bị | cần một driver thật công bố thuộc tính mới có dữ liệu để đọc | bài [149](149-dynamic-resource-allocation-vi.md) mục *Lược đồ metadata*, đã đọc ở mạch chính giai đoạn 13 |
| Hai lệnh `kubectl apply -f https://k8s.io/examples/dra/…` và mục *Dọn dẹp* | trên cluster không có thiết bị, Pod dừng ở pending chứ không chạy tới bước xem log | bài [270](270-allocate-devices-dra-vi.md) giải thích trạng thái pending đó; [Lab 13](labs/LAB-13-DRA.md) phần B4 kiểm chứng |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Trang này hướng dẫn cách truy cập
[metadata thiết bị](149-dynamic-resource-allocation-vi.md#device-metadata)
từ các container sử dụng _cấp phát tài nguyên động (dynamic resource allocation — DRA)_.
Metadata thiết bị cho phép workload khám phá thông tin về các thiết bị đã được cấp phát,
chẳng hạn như các thuộc tính (attribute) của thiết bị hoặc chi tiết network interface — bằng
cách đọc các file JSON tại những đường dẫn quy ước (well-known path) bên trong container.

Trước khi đọc trang này, hãy làm quen với
[Cấp phát tài nguyên động (Dynamic Resource Allocation — DRA)](149-dynamic-resource-allocation-vi.md)
và cách
[cấp phát thiết bị cho workload](270-allocate-devices-dra-vi.md).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.36 hoặc mới hơn. Để kiểm tra phiên bản,
nhập `kubectl version`.

* Hãy chắc chắn rằng quản trị viên cluster của bạn đã thiết lập DRA, gắn thiết bị và cài đặt
  driver. Để biết thêm thông tin, xem
  [Thiết lập DRA trong một cluster](271-set-up-dra-cluster-vi.md).
* Hãy chắc chắn rằng DRA driver được triển khai trong cluster của bạn hỗ trợ metadata thiết bị.
  Các driver dùng [DRA kubelet plugin](https://pkg.go.dev/k8s.io/dynamic-resource-allocation/kubeletplugin)
  bật các tùy chọn `EnableDeviceMetadata` và
  `MetadataVersions` khi khởi động plugin. Hãy xem tài liệu của driver
  để biết chi tiết.

## Truy cập metadata thiết bị bằng ResourceClaim (Access device metadata with a ResourceClaim) {#access-metadata-resourceclaim}

Khi bạn dùng một ResourceClaim được tham chiếu trực tiếp để cấp phát thiết bị, các file
metadata thiết bị xuất hiện bên trong container tại:

```
/var/run/kubernetes.io/dra-device-attributes/resourceclaims/<claimName>/<requestName>/<driverName>-metadata.json
```

1. Xem lại manifest ví dụ sau:

   ```yaml
   apiVersion: resource.k8s.io/v1
   kind: ResourceClaim
   metadata:
     name: gpu-claim
   spec:
     devices:
       requests:
       - name: gpu
         exactly:
           deviceClassName: gpu.example.com
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: gpu-metadata-reader
   spec:
     resourceClaims:
     - name: my-gpu
       resourceClaimName: gpu-claim
     containers:
     - name: workload
       image: ubuntu:24.04
       resources:
         claims:
         - name: my-gpu
           request: gpu
       command:
       - sh
       - -c
       - |
         echo "=== DRA device metadata ==="
         find /var/run/kubernetes.io/dra-device-attributes -name '*-metadata.json' -print -exec cat {} \;
         sleep 3600
     restartPolicy: Never
   ```

   Manifest này tạo một ResourceClaim tên là `gpu-claim` yêu cầu một
   thiết bị từ DeviceClass `gpu.example.com`, và một Pod đọc
   metadata của thiết bị.

1. Tạo ResourceClaim và Pod:

   ```shell
   kubectl apply -f https://k8s.io/examples/dra/dra-device-metadata-pod.yaml
   ```

1. Sau khi Pod chạy, xem log của container để thấy metadata:

   ```shell
   kubectl logs gpu-metadata-reader
   ```

   Output sẽ tương tự như sau:

   ```
   === DRA device metadata ===
   /var/run/kubernetes.io/dra-device-attributes/resourceclaims/gpu-claim/gpu/gpu.example.com-metadata.json
   {
     "kind": "DeviceMetadata",
     "apiVersion": "metadata.resource.k8s.io/v1alpha1",
     ...
   }
   ```

1. Để xem toàn bộ file metadata, hãy exec vào container:

   ```shell
   kubectl exec gpu-metadata-reader -- \
     cat /var/run/kubernetes.io/dra-device-attributes/resourceclaims/gpu-claim/gpu/gpu.example.com-metadata.json
   ```

   Output là một đối tượng JSON chứa các thuộc tính thiết bị như model,
   phiên bản driver và UUID của thiết bị. Xem
   [lược đồ metadata (metadata schema)](149-dynamic-resource-allocation-vi.md#device-metadata-schema)
   để biết chi tiết về cấu trúc JSON.

## Truy cập metadata thiết bị bằng ResourceClaimTemplate (Access device metadata with a ResourceClaimTemplate) {#access-metadata-template}

Khi bạn dùng một ResourceClaimTemplate, Kubernetes sinh ra một ResourceClaim cho
mỗi Pod. Vì tên của claim được sinh ra không thể đoán trước, các file metadata
xuất hiện tại một đường dẫn dùng tên tham chiếu claim trong Pod (Pod's claim
reference name) thay thế:

```
/var/run/kubernetes.io/dra-device-attributes/resourceclaimtemplates/<podClaimName>/<requestName>/<driverName>-metadata.json
```

`<podClaimName>` tương ứng với field `name` trong mục
`spec.resourceClaims[]` của Pod. JSON metadata cũng bao gồm một field
`podClaimName` ghi lại ánh xạ này.

1. Xem lại manifest ví dụ sau:

   ```yaml
   apiVersion: resource.k8s.io/v1
   kind: ResourceClaimTemplate
   metadata:
     name: gpu-claim-template
   spec:
     spec:
       devices:
         requests:
         - name: gpu
           exactly:
             deviceClassName: gpu.example.com
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: gpu-metadata-template-reader
   spec:
     resourceClaims:
     - name: my-gpu
       resourceClaimTemplateName: gpu-claim-template
     containers:
     - name: workload
       image: ubuntu:24.04
       resources:
         claims:
         - name: my-gpu
           request: gpu
       command:
       - sh
       - -c
       - |
         echo "=== DRA device metadata (from template) ==="
         find /var/run/kubernetes.io/dra-device-attributes -name '*-metadata.json' -print -exec cat {} \;
         sleep 3600
     restartPolicy: Never
   ```

   Manifest này tạo một ResourceClaimTemplate và một Pod. Mỗi Pod nhận được
   một ResourceClaim riêng được sinh ra cho nó. Đường dẫn metadata dùng tên tham chiếu
   claim của Pod là `my-gpu`.

1. Tạo ResourceClaimTemplate và Pod:

   ```shell
   kubectl apply -f https://k8s.io/examples/dra/dra-device-metadata-template-pod.yaml
   ```

1. Sau khi Pod chạy, xem metadata:

   ```shell
   kubectl exec gpu-metadata-template-reader -- \
     cat /var/run/kubernetes.io/dra-device-attributes/resourceclaimtemplates/my-gpu/gpu/gpu.example.com-metadata.json
   ```

## Đọc metadata trong ứng dụng của bạn (Read metadata in your application) {#read-metadata-application}

### Ứng dụng Go (Go applications)

Package `k8s.io/dynamic-resource-allocation/devicemetadata` cung cấp các
hàm dựng sẵn để đọc các file metadata. Các hàm này tự động xử lý việc
thương lượng phiên bản (version negotiation), giải mã luồng metadata và chuyển đổi
nó sang các kiểu nội bộ, nhờ đó code của bạn hoạt động được trên nhiều phiên bản lược đồ
(schema version) mà không cần kiểm tra phiên bản thủ công.

Với một ResourceClaim được tham chiếu trực tiếp:

```go
import "k8s.io/dynamic-resource-allocation/devicemetadata"

dm, err := devicemetadata.ReadResourceClaimMetadata("gpu-claim", "gpu")
```

Với một claim được sinh từ template (dùng tên tham chiếu claim trong Pod):

```go
dm, err := devicemetadata.ReadResourceClaimTemplateMetadata("my-gpu", "gpu")
```

Nếu bạn biết tên driver cụ thể, bạn có thể đọc file metadata của một driver
duy nhất:

```go
dm, err := devicemetadata.ReadResourceClaimMetadataWithDriverName("gpu.example.com", "gpu-claim", "gpu")
```

Giá trị trả về `*metadata.DeviceMetadata` chứa metadata của claim, các request,
và các thuộc tính theo từng thiết bị.

Các ứng dụng viết bằng ngôn ngữ khác có thể đọc trực tiếp file JSON và kiểm tra
field `apiVersion` để xác định phiên bản lược đồ trước khi phân tích cú pháp (parse).

## Dọn dẹp (Clean up) {#clean-up}

Xóa các tài nguyên mà bạn đã tạo:

```shell
kubectl delete -f https://k8s.io/examples/dra/dra-device-metadata-pod.yaml
kubectl delete -f https://k8s.io/examples/dra/dra-device-metadata-template-pod.yaml
```

## Tiếp theo (What's next)

* [Tìm hiểu thêm về metadata thiết bị DRA](149-dynamic-resource-allocation-vi.md#device-metadata)
* [Cấp phát thiết bị cho workload bằng DRA](270-allocate-devices-dra-vi.md)
* Để biết thêm thông tin về thiết kế, xem
  [KEP-5304](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/5304-dra-attributes-downward-api).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 13:

1. Ứng dụng trong container lấy metadata thiết bị bằng con đường nào, và vì sao đường dẫn của một
   ResourceClaim tham chiếu trực tiếp lại khác đường dẫn của claim sinh từ ResourceClaimTemplate?
2. **Câu bẫy.** Cluster đã bật DRA, driver đã cài, Pod đã được cấp phát thiết bị. Vậy chắc chắn có
   file metadata trong container chứ?
3. Trên `lab-k8s-worker2`, bạn chạy một Pod bình thường — không có mục `resources.claims` nào —
   rồi `kubectl exec` vào chạy `find /var/run/kubernetes.io/dra-device-attributes`. Bạn kỳ vọng
   thấy gì, và điều đó nói lên quy tắc nào về thời điểm file metadata xuất hiện?
4. Ứng dụng của bạn không viết bằng Go. Bài bảo đọc metadata thế nào, phải kiểm field nào trước
   khi parse, và để làm gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Bằng cách **đọc file JSON tại những đường dẫn quy ước bên trong chính container** — bài dùng
   `find … -name '*-metadata.json' -exec cat {} \;` và `kubectl exec … cat …`, không gọi API nào.
   Hai đường dẫn khác nhau vì với **ResourceClaimTemplate, Kubernetes sinh một ResourceClaim cho
   mỗi Pod và tên claim sinh ra không thể đoán trước**; không đoán được tên thì không đặt được nó
   vào đường dẫn, nên đường dẫn dùng **`<podClaimName>`** — field `name` trong
   `spec.resourceClaims[]` của Pod — thay cho `<claimName>`, và nhánh đầu đường dẫn đổi từ
   `resourceclaims/` sang `resourceclaimtemplates/`.
2. **Không chắc.** Ngoài việc quản trị viên đã thiết lập DRA, gắn thiết bị và cài driver, bài còn
   đòi **DRA driver phải hỗ trợ metadata thiết bị** — tức driver bật `EnableDeviceMetadata` và
   `MetadataVersions` khi khởi động kubelet plugin, và bài bảo phải **xem tài liệu của driver** để
   biết. Cộng thêm điều kiện phiên bản: tính năng ở mức **alpha**, server phải **v1.36 trở lên**.
3. **Không thấy gì** — lệnh `find` không tìm ra file nào. Quy ước đường dẫn của bài luôn đi qua
   một tên claim và một tên request, tức metadata **chỉ xuất hiện cho container thật sự yêu cầu
   thiết bị** qua `resources.claims` trỏ tới một claim đã được cấp phát. Không có claim thì không
   có nhánh đường dẫn nào để tạo ra.
4. **Đọc trực tiếp file JSON** ở đúng đường dẫn quy ước, và **kiểm field `apiVersion`** để xác
   định **phiên bản lược đồ** trước khi phân tích cú pháp. Mục đích: code chạy được trên nhiều
   phiên bản lược đồ — đúng việc mà package Go `devicemetadata` làm sẵn bằng cơ chế thương lượng
   phiên bản, còn ngôn ngữ khác thì phải tự làm.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

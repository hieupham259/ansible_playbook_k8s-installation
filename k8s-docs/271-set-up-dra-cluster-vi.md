# Thiết lập DRA trong một cluster (Set Up DRA in a Cluster)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/set-up-dra-cluster/
>
> Trang này hướng dẫn cách cấu hình cấp phát tài nguyên động (dynamic resource allocation — DRA)
> trong một cluster Kubernetes bằng cách bật các API group và cấu hình các lớp thiết bị
> (classes of devices). Các hướng dẫn này dành cho quản trị viên cluster.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13 — Lập lịch và workload nâng cao](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao)
→ dòng **Thực hành**, **bài 5/5 — bài cuối của nhóm** · Kiểm chứng ở
[Lab 13 — DRA](labs/LAB-13-DRA.md) phần B1.1, B3.2 và B11.

Bài này viết cho **quản trị viên cluster** và là bài duy nhất trong nhóm nói về phía cung: bật API
group, xác minh, cài driver, tạo lớp thiết bị. Mục *Xác minh rằng DRA đã được bật* chính là **phép
kiểm đầu tiên** mà Lab 13 dùng để đo năng lực DRA của cluster rồi rẽ nhánh — lab **cấm** cài
driver, bật feature gate hay tự tạo ResourceSlice để "có dữ liệu".

**Phải hiểu ở lần đọc này:**

- Bốn việc của quản trị viên theo đúng thứ tự bài đặt: (tùy chọn) bật thêm API group → **xác minh**
  DRA → cài driver thiết bị → tạo DeviceClass. Kèm một điều kiện tiên quyết dễ bỏ qua: gắn thiết bị
  trước, nhưng **đợi thiết lập xong DRA cho cluster rồi mới cài driver**, để tránh sự cố với driver.
- Phép xác minh và cách đọc hai output của `kubectl get deviceclasses`: `No resources found` nghĩa
  là **cấu hình đúng**; `error: the server doesn't have a resource type "deviceclasses"` nghĩa là
  **nhóm API `resource.k8s.io` bị tắt**. Phân biệt "không có DeviceClass nào" với "không có kiểu
  DeviceClass" là toàn bộ giá trị của mục này.
- DRA nói chung đã stable, nhưng **một số khía cạnh vẫn alpha/beta và có kind API riêng**: bật
  `resource.k8s.io/v1beta1` và `v1beta2` chỉ khi cần đỡ driver/workload cũ, `resource.k8s.io/v1alpha3`
  cho các tính năng alpha. Nếu `.spec.resourceClaims` bị loại khỏi Pod hoặc Pod được lập lịch mà bỏ
  qua ResourceClaim thì kiểm feature gate `DynamicResourceAllocation` trên bốn thành phần:
  kube-apiserver, kube-controller-manager, kube-scheduler và kubelet.
- **ResourceSlice do driver phát hành**, không do bạn tạo: `kubectl get resourceslices` là phép xác
  minh driver đang hoạt động, và cột `NODE`/`DRIVER`/`POOL` cho biết thiết bị nằm ở đâu. Driver
  không công bố được thì đọc log driver, không sửa ResourceSlice bằng tay.
- DeviceClass là **danh mục do quản trị viên định nghĩa bằng selector CEL** trên thuộc tính mà
  driver đã công bố trong ResourceSlice — ví dụ `device.driver == "driver.example.com"`. Vì thế
  quy trình đúng là **đọc ResourceSlice trước** (`kubectl get resourceslice <tên> -o yaml`) để biết
  có thuộc tính nào, rồi mới viết selector.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Cài đặt driver thiết bị* ở mức làm thật | driver do nhà cung cấp thiết bị phát hành; ba VM của lab không có thiết bị nào để cài driver | [Lab 13](labs/LAB-13-DRA.md) phần B1 đo năng lực rồi rẽ nhánh, và cấm cài driver |
| Cách bật/tắt một API group cho kube-apiserver, và bước khắc phục "cấu hình lại và khởi động lại `kube-apiserver`" | là sửa cấu hình control plane của một cluster đang chạy | [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) — bài [207](207-enable-disable-api-vi.md) và [196](196-configure-feature-gates-vi.md) |
| Nội dung ví dụ của ResourceSlice — `attributes`, `capacity`, `nodeName` | cần một driver thật mới có ResourceSlice để đọc | bài [149](149-dynamic-resource-allocation-vi.md) mục *ResourceSlice*, đã đọc ở mạch chính giai đoạn 13 |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

Trang này hướng dẫn cách cấu hình _cấp phát tài nguyên động (dynamic resource allocation — DRA)_
trong một cluster Kubernetes bằng cách bật các API group và cấu hình các lớp thiết bị
(classes of devices). Các hướng dẫn này dành cho quản trị viên cluster.

## Về DRA (About DRA) {#about-dra}

DRA là một tính năng của Kubernetes cho phép bạn yêu cầu (request) và chia sẻ tài nguyên
giữa các Pod. Các tài nguyên này thường là các thiết bị (device) gắn kèm,
chẳng hạn như bộ tăng tốc phần cứng (hardware accelerator).

Với DRA, các driver thiết bị và quản trị viên cluster định nghĩa các _lớp_ thiết bị
(device class) sẵn sàng để _claim_ (yêu cầu sử dụng) trong các workload. Kubernetes
cấp phát các thiết bị khớp với những claim cụ thể và đặt các Pod tương ứng
lên các node có thể truy cập những thiết bị đã được cấp phát.

Hãy chắc chắn rằng bạn đã quen với cách DRA hoạt động và với các thuật ngữ của DRA như
DeviceClass, ResourceClaim và ResourceClaimTemplate.
Để biết chi tiết, xem
[Cấp phát tài nguyên động (Dynamic Resource Allocation — DRA)](149-dynamic-resource-allocation-vi.md).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.34 hoặc mới hơn. Để kiểm tra phiên bản,
nhập `kubectl version`.

* Gắn thiết bị vào cluster của bạn, trực tiếp hoặc gián tiếp. Để tránh các sự cố tiềm ẩn
  với driver, hãy đợi đến khi bạn thiết lập xong tính năng DRA cho cluster
  rồi mới cài đặt driver.

## Tùy chọn: bật thêm các API group của DRA (Optional: enable additional DRA API groups) {#enable-dra}

DRA nhìn chung là một tính năng ổn định (stable) trong Kubernetes; tuy nhiên, một số khía cạnh
của nó có thể vẫn đang ở giai đoạn alpha hoặc beta.
Nếu bạn muốn dùng bất kỳ khía cạnh nào của DRA chưa ổn định,
và tính năng liên quan dựa trên một kind API riêng,
thì bạn phải bật các API group alpha hoặc beta tương ứng.

Một số DRA driver hoặc workload cũ hơn có thể vẫn cần
API v1beta1 từ Kubernetes 1.30 hoặc v1beta2 từ Kubernetes 1.32.
Chỉ khi và chỉ nếu bạn muốn hỗ trợ chúng, hãy bật các
[API group](https://kubernetes.io/docs/reference/using-api/#api-groups) sau:

* `resource.k8s.io/v1beta1`
* `resource.k8s.io/v1beta2`

Các tính năng alpha có kiểu API riêng cần:

* `resource.k8s.io/v1alpha3`

Để biết thêm thông tin, xem
[Bật hoặc tắt các API group](https://kubernetes.io/docs/reference/using-api/#enabling-or-disabling).

## Xác minh rằng DRA đã được bật (Verify that DRA is enabled) {#verify}

Để xác minh rằng cluster đã được cấu hình đúng, hãy thử liệt kê các DeviceClass:

```shell
kubectl get deviceclasses
```

Nếu cấu hình của các thành phần là chính xác, output sẽ tương tự như sau:

```
No resources found
```

Nếu DRA chưa được cấu hình đúng, output của lệnh trên sẽ tương tự như sau:

```
error: the server doesn't have a resource type "deviceclasses"
```

Ví dụ, điều này có thể xảy ra khi API group resource.k8s.io bị tắt.
Một phép kiểm tra tương tự cũng áp dụng được cho các kiểu cấp cao nhất (top-level type)
ở chất lượng alpha hoặc beta.

Hãy thử các bước khắc phục sự cố sau:

1. Cấu hình lại và khởi động lại thành phần `kube-apiserver`.

1. Nếu toàn bộ field `.spec.resourceClaims` bị loại bỏ khỏi các Pod, hoặc nếu
   các Pod được lập lịch (schedule) mà không xét đến các ResourceClaim, hãy xác minh
   rằng [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
   `DynamicResourceAllocation` không bị tắt
   cho kube-apiserver, kube-controller-manager, kube-scheduler hoặc kubelet.

## Cài đặt driver thiết bị (Install device drivers) {#install-drivers}

Sau khi bạn bật DRA cho cluster, bạn có thể cài đặt các driver cho những thiết bị
đã gắn kèm. Để có hướng dẫn, hãy xem tài liệu của chủ sở hữu thiết bị
hoặc của dự án duy trì các driver thiết bị đó. Các driver mà bạn
cài đặt phải tương thích với DRA.

Để xác minh rằng các driver đã cài đặt đang hoạt động như mong đợi, hãy liệt kê
các ResourceSlice trong cluster của bạn:

```shell
kubectl get resourceslices
```

Output sẽ tương tự như sau:

```
NAME                                                  NODE                DRIVER               POOL                             AGE
00000-driver.example.com-cluster-1-node-1-abcde      cluster-1-node-1    driver.example.com   cluster-1-device-pool-1-r1gc     7s
00000-driver.example.com-cluster-1-node-2-fghij      cluster-1-node-2    driver.example.com   cluster-1-device-pool-2-446z     8s
```

Hãy thử các bước khắc phục sự cố sau:

1. Kiểm tra tình trạng (health) của DRA driver và tìm các thông báo lỗi về việc
   phát hành (publish) ResourceSlice trong log của nó. Nhà cung cấp driver
   có thể có thêm hướng dẫn về cài đặt và khắc phục sự cố.

## Tạo DeviceClass (Create DeviceClasses) {#create-deviceclasses}

Bạn có thể định nghĩa các nhóm thiết bị mà người vận hành ứng dụng có thể
claim trong workload bằng cách tạo các DeviceClass. Một số nhà cung cấp
driver thiết bị cũng có thể hướng dẫn bạn tạo DeviceClass trong quá trình
cài đặt driver.

Các ResourceSlice mà driver của bạn phát hành chứa thông tin về các
thiết bị mà driver đó quản lý, chẳng hạn như dung lượng (capacity), metadata và các thuộc tính
(attribute). Bạn có thể dùng CEL (Common Expression Language) để lọc theo các thuộc tính trong
DeviceClass của mình, điều này giúp người vận hành workload tìm thiết bị
dễ dàng hơn.

1.  Để tìm các thuộc tính thiết bị mà bạn có thể chọn trong DeviceClass bằng
    biểu thức CEL, hãy lấy đặc tả (specification) của một ResourceSlice:

    ```shell
    kubectl get resourceslice <resourceslice-name> -o yaml
    ```

    Output sẽ tương tự như sau:

    ```yaml
    apiVersion: resource.k8s.io/v1
    kind: ResourceSlice
    # các dòng được lược bỏ cho rõ ràng
    spec:
      devices:
      - attributes:
          type:
            string: gpu
        capacity:
          memory:
            value: 64Gi
        name: gpu-0
      - attributes:
          type:
            string: gpu
        capacity:
          memory:
            value: 64Gi
        name: gpu-1
      driver: driver.example.com
      nodeName: cluster-1-node-1
    # các dòng được lược bỏ cho rõ ràng
    ```

    Bạn cũng có thể xem tài liệu của nhà cung cấp driver để biết các thuộc tính
    và giá trị khả dụng.

1.  Xem lại manifest DeviceClass ví dụ sau, manifest này chọn mọi thiết bị
    được quản lý bởi driver thiết bị `driver.example.com`:

    ```yaml
    apiVersion: resource.k8s.io/v1
    kind: DeviceClass
    metadata:
      name: example-device-class
    spec:
      selectors:
      - cel:
          expression: |-
            device.driver == "driver.example.com"
    ```

1.  Tạo DeviceClass trong cluster của bạn:

    ```shell
    kubectl apply -f https://k8s.io/examples/dra/deviceclass.yaml
    ```

## Dọn dẹp (Clean up) {#clean-up}

Để xóa DeviceClass mà bạn đã tạo trong bài thực hành này, chạy lệnh sau:

```shell
kubectl delete -f https://k8s.io/examples/dra/deviceclass.yaml
```

## Tiếp theo (What's next)

* [Tìm hiểu thêm về DRA](149-dynamic-resource-allocation-vi.md)
* [Cấp phát thiết bị cho workload bằng DRA](270-allocate-devices-dra-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 13:

1. **Câu bẫy.** `kubectl get deviceclasses` in ra `No resources found`. DRA của cluster hỏng hay
   chạy? Output nào mới là output của cluster chưa cấu hình đúng, và nó chỉ ra chuyện gì?
2. Trên `lab-k8s-master`, giả sử `kubectl get deviceclasses` chạy được nhưng
   `kubectl get resourceslices` trả về rỗng. Kết luận gì về cluster lab? Nếu bạn tạo thêm một
   DeviceClass thì có thiết bị nào xuất hiện không, và vì sao?
3. Bài đặt các việc theo thứ tự nào, và vì sao nó dặn **đợi thiết lập xong DRA cho cluster rồi mới
   cài driver** thay vì làm ngược lại?
4. Khi nào phải bật `resource.k8s.io/v1beta1` và `v1beta2`, khi nào phải bật
   `resource.k8s.io/v1alpha3`? Nếu `.spec.resourceClaims` bị loại khỏi Pod thì kiểm cái gì, trên
   những thành phần nào?
5. Bạn muốn viết selector CEL cho một DeviceClass mới. Bài bảo lấy tên thuộc tính ở đâu, và vì sao
   không thể tự nghĩ ra tên thuộc tính?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Đang chạy đúng.** `No resources found` là output mà bài nói ra khi "cấu hình của các thành
   phần là chính xác" — nó chỉ nghĩa là **chưa có DeviceClass nào được tạo**. Output của cluster
   chưa cấu hình đúng là `error: the server doesn't have a resource type "deviceclasses"`, tức
   **apiserver không phục vụ kiểu đó** — ví dụ khi nhóm API `resource.k8s.io` bị tắt. Chỗ dễ sai
   là gộp hai thứ làm một: "danh sách rỗng" khác hẳn "không có kiểu này".
2. Kết luận: **nhóm API DRA có được phục vụ, nhưng không có driver nào đang công bố thiết bị** —
   vì **ResourceSlice là thứ do driver phát hành**, và `kubectl get resourceslices` chính là phép
   xác minh driver hoạt động mà bài quy định. Tạo thêm DeviceClass **không làm xuất hiện thiết bị
   nào**: DeviceClass chỉ là **selector CEL lọc trên thuộc tính mà driver đã công bố**; lọc trên
   một tập rỗng thì vẫn rỗng.
3. Thứ tự: **(tùy chọn) bật thêm API group → xác minh DRA đã bật → cài driver thiết bị → tạo
   DeviceClass**. Bài dặn gắn thiết bị vào cluster nhưng **đợi thiết lập xong tính năng DRA rồi mới
   cài driver** để **tránh các sự cố tiềm ẩn với driver** — driver khởi động trên một cluster chưa
   phục vụ nhóm API mà nó cần thì không có chỗ để công bố ResourceSlice.
4. Bật **`v1beta1` và `v1beta2`** *chỉ khi và chỉ nếu* bạn muốn hỗ trợ **các DRA driver hoặc
   workload cũ hơn** còn cần API v1beta1 từ Kubernetes 1.30 hoặc v1beta2 từ 1.32. Bật
   **`v1alpha3`** cho **các tính năng alpha có kiểu API riêng**. Nếu `.spec.resourceClaims` bị loại
   khỏi Pod — hoặc Pod được lập lịch mà không xét ResourceClaim — thì xác minh feature gate
   **`DynamicResourceAllocation` không bị tắt** cho **kube-apiserver, kube-controller-manager,
   kube-scheduler và kubelet**.
5. Lấy từ **chính ResourceSlice mà driver đã phát hành**: `kubectl get resourceslice <tên> -o yaml`
   rồi đọc phần `spec.devices[].attributes` và `capacity`; hoặc xem tài liệu của nhà cung cấp
   driver. Không tự nghĩ ra được vì **thuộc tính là do driver công bố**, không phải do Kubernetes
   quy định — selector viết cho một thuộc tính không tồn tại thì đơn giản là không khớp thiết bị
   nào.

</details>

Hết nhóm Thực hành của giai đoạn 13. Câu nào chưa trả lời được thì quay lại đúng mục tương ứng,
rồi mới mở [Lab 13 — DRA](labs/LAB-13-DRA.md) — lab **tùy chọn**, và vì ba VM của lab không có GPU
nên nó chạy **nhánh đọc-hiểu**: phần B1 đo năng lực DRA thật của cluster bằng năm phép kiểm rồi
chốt nhánh, thay vì cài driver để "có dữ liệu".

# Cấp phát thiết bị cho workload bằng DRA (Allocate Devices to Workloads with DRA)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/allocate-devices-dra/
>
> Trang này hướng dẫn cách cấp phát thiết bị cho các Pod của bạn bằng cấp phát tài nguyên động
> (dynamic resource allocation — DRA). Các hướng dẫn này dành cho người vận hành workload.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13 — Lập lịch và workload nâng cao](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao)
→ dòng **Thực hành**, bài 4/5 · Kiểm chứng ở [Lab 13 — DRA](labs/LAB-13-DRA.md) phần B4.

Bài này viết cho **người vận hành workload**, không phải cho quản trị viên — đó là lý do nó bắt
đầu bằng `kubectl get deviceclasses` chứ không bằng việc tạo ra DeviceClass. Cluster lab không có
thiết bị nào, nên ở phần B4 của Lab 13 bạn chỉ đi tới **đúng chỗ mà bài nói luồng sẽ dừng lại**:
Pod ở trạng thái pending.

**Phải hiểu ở lần đọc này:**

- Chọn giữa hai cách claim theo đúng tiêu chí ở mục *Claim tài nguyên*: **ResourceClaim** tạo thủ
  công khi nhiều Pod cần **chia sẻ** cùng nhóm thiết bị, hoặc khi claim phải sống lâu hơn vòng đời
  một Pod; **ResourceClaimTemplate** khi mỗi Pod cần **thiết bị riêng** nhưng cấu hình giống nhau
  — ví dụ các Pod của một Job chạy song song.
- Yêu cầu thiết bị nằm ở **hai chỗ** trong Pod spec: khai báo tại `pod.spec.resourceClaims`, mỗi
  mục đặt đúng **một** trong `resourceClaimName` hoặc `resourceClaimTemplateName`; rồi container
  gọi tên claim đó tại `resources.claims`. Trong Job ví dụ, `container1` và `container2` cùng gọi
  `shared-gpu-claim` nên **chia sẻ** thiết bị, còn `container0` gọi `separate-gpu-claim` sinh từ
  template.
- Hậu quả khi tham chiếu một ResourceClaim **chưa tồn tại**: Pod **ở trạng thái pending cho đến
  khi ResourceClaim được tạo** — không phải lỗi lúc tạo Pod. Bài cũng khuyến cáo **không** tham
  chiếu một ResourceClaim tự động sinh ra, vì claim đó bị ràng buộc vào vòng đời của Pod đã kích
  hoạt việc sinh ra nó.
- Điểm khởi đầu của người vận hành workload là `kubectl get deviceclasses`; **lỗi quyền ở lệnh này
  không phải lỗi cấu hình của bạn** — bài bảo đi hỏi quản trị viên cluster hoặc nhà cung cấp driver
  về các thuộc tính thiết bị khả dụng.
- Hai bước khắc phục sự cố của bài: đi sâu dần **Job → Pod → ResourceClaim** và `kubectl describe`
  ở từng cấp để tìm field trạng thái hay event giải thích được; còn lỗi `must specify one of:
  resourceClaimName, resourceClaimTemplateName` thì kiểm hai field trên, và nếu chúng đã đúng thì
  nghi một mutating Pod webhook được build trên các API Kubernetes < 1.32.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Biểu thức CEL trong `selectors` — `device.attributes[…]`, `device.capacity[…] == quantity("64Gi")` | phải có ResourceSlice thật mới biết thuộc tính nào tồn tại; không có thiết bị thì mọi selector đều không khớp | bài [271](271-set-up-dra-cluster-vi.md) mục *Tạo DeviceClass*, bài kế của nhóm; [Lab 13](labs/LAB-13-DRA.md) phần B3.2 |
| Cơ chế `completions: 10` và `parallelism: 2` của Job ví dụ | ở đây Job chỉ là cái vỏ để cho thấy hai chỗ khai báo claim | bài [67](67-job-vi.md), đã đọc ở [giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller) |
| Chạy thật Job ví dụ rồi xem thiết bị được cấp phát, và mục *Dọn dẹp* đi kèm | cluster lab không có GPU và không có DRA driver | [Lab 13](labs/LAB-13-DRA.md) — chỉ nhánh A cấp phát thật; nhánh B dừng lại ở trạng thái pending |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

Trang này hướng dẫn cách cấp phát thiết bị cho các Pod của bạn bằng
_cấp phát tài nguyên động (dynamic resource allocation — DRA)_. Các hướng dẫn này dành cho
người vận hành workload (workload operator). Trước khi đọc trang này, hãy làm quen với cách
DRA hoạt động và với các thuật ngữ của DRA như
ResourceClaim và ResourceClaimTemplate.
Để biết thêm thông tin, xem
[Cấp phát tài nguyên động (Dynamic Resource Allocation — DRA)](149-dynamic-resource-allocation-vi.md).

## Về việc cấp phát thiết bị bằng DRA (About device allocation with DRA) {#about-device-allocation-dra}

Với vai trò người vận hành workload, bạn có thể _claim_ (yêu cầu sử dụng) thiết bị cho các
workload của mình bằng cách tạo ResourceClaim hoặc ResourceClaimTemplate. Khi bạn triển khai
workload, Kubernetes và các driver thiết bị sẽ tìm thiết bị khả dụng, cấp phát chúng cho các
Pod của bạn, và đặt các Pod lên những node có thể truy cập các thiết bị đó.

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

* Hãy chắc chắn rằng quản trị viên cluster của bạn đã thiết lập DRA, gắn thiết bị và cài đặt
  driver. Để biết thêm thông tin, xem
  [Thiết lập DRA trong một cluster](271-set-up-dra-cluster-vi.md).

## Nhận diện thiết bị cần claim (Identify devices to claim) {#identify-devices}

Quản trị viên cluster của bạn hoặc các driver thiết bị sẽ tạo các
_DeviceClass_ định nghĩa các nhóm thiết bị. Bạn có thể claim thiết bị bằng cách dùng
CEL (Common Expression Language) để lọc theo các thuộc tính (property) cụ thể của thiết bị.

Lấy danh sách các DeviceClass trong cluster:

```shell
kubectl get deviceclasses
```

Output sẽ tương tự như sau:

```
NAME                 AGE
driver.example.com   16m
```

Nếu bạn gặp lỗi về quyền (permission error), có thể bạn không có quyền lấy danh sách DeviceClass.
Hãy hỏi quản trị viên cluster hoặc nhà cung cấp driver về các thuộc tính thiết bị
khả dụng.

## Claim tài nguyên (Claim resources) {#claim-resources}

Bạn có thể yêu cầu tài nguyên từ một DeviceClass bằng cách dùng
ResourceClaim. Để tạo một ResourceClaim, hãy làm một trong các cách sau:

* Tạo thủ công một ResourceClaim nếu bạn muốn nhiều Pod chia sẻ quyền truy cập
  cùng một nhóm thiết bị, hoặc nếu bạn muốn một claim tồn tại lâu hơn vòng đời của một
  Pod.
* Dùng một
  ResourceClaimTemplate
  để Kubernetes tự sinh và quản lý ResourceClaim riêng cho từng Pod. Hãy tạo
  ResourceClaimTemplate nếu bạn muốn mỗi Pod có quyền truy cập vào các thiết bị riêng biệt
  nhưng có cấu hình tương tự nhau. Ví dụ, bạn có thể muốn truy cập đồng thời
  vào thiết bị cho các Pod trong một Job dùng
  [thực thi song song (parallel execution)](67-job-vi.md#parallel-jobs).

Nếu bạn tham chiếu trực tiếp một ResourceClaim cụ thể trong một Pod, ResourceClaim đó
phải đã tồn tại trong cluster. Nếu ResourceClaim được tham chiếu không tồn tại,
Pod sẽ ở trạng thái pending cho đến khi ResourceClaim được tạo. Bạn có thể
tham chiếu một ResourceClaim được tự động sinh ra trong một Pod, nhưng điều này không được
khuyến nghị vì các ResourceClaim tự động sinh bị ràng buộc vào vòng đời của Pod
đã kích hoạt việc sinh ra chúng.

Để tạo một workload claim tài nguyên, hãy chọn một trong các phương án sau:

#### ResourceClaimTemplate

Xem lại manifest ví dụ sau:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: example-resource-claim-template
spec:
  spec:
    devices:
      requests:
      - name: gpu-claim
        exactly:
          deviceClassName: example-device-class
          selectors:
            - cel:
                expression: |-
                  device.attributes["driver.example.com"].type == "gpu" &&
                  device.capacity["driver.example.com"].memory == quantity("64Gi")
```

Manifest này tạo một ResourceClaimTemplate yêu cầu các thiết bị trong
DeviceClass `example-device-class` khớp cả hai điều kiện sau:

* Thiết bị có thuộc tính `driver.example.com/type` với giá trị
  `gpu`.
* Thiết bị có dung lượng (capacity) `64Gi`.

Để tạo ResourceClaimTemplate, chạy lệnh sau:

```shell
kubectl apply -f https://k8s.io/examples/dra/resourceclaimtemplate.yaml
```

#### ResourceClaim

Xem lại manifest ví dụ sau:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: example-resource-claim
spec:
  devices:
    requests:
    - name: single-gpu-claim
      exactly:
        deviceClassName: example-device-class
        allocationMode: All
        selectors:
        - cel:
            expression: |-
              device.attributes["driver.example.com"].type == "gpu" &&
              device.capacity["driver.example.com"].memory == quantity("64Gi")
```

Manifest này tạo một ResourceClaim yêu cầu các thiết bị trong
DeviceClass `example-device-class` khớp cả hai điều kiện sau:

* Thiết bị có thuộc tính `driver.example.com/type` với giá trị
  `gpu`.
* Thiết bị có dung lượng (capacity) `64Gi`.

Để tạo ResourceClaim, chạy lệnh sau:

```shell
kubectl apply -f https://k8s.io/examples/dra/resourceclaim.yaml
```

## Yêu cầu thiết bị trong workload bằng DRA (Request devices in workloads using DRA) {#request-devices-workloads}

Để yêu cầu cấp phát thiết bị, hãy chỉ định một ResourceClaim hoặc một ResourceClaimTemplate
trong field `resourceClaims` của đặc tả Pod. Sau đó, yêu cầu một
claim cụ thể theo tên trong field `resources.claims` của một container trong Pod đó.
Bạn có thể chỉ định nhiều mục trong field `resourceClaims` và dùng các
claim cụ thể trong các container khác nhau.

1. Xem lại Job ví dụ sau:

   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: example-dra-job
   spec:
     completions: 10
     parallelism: 2
     template:
       spec:
         restartPolicy: Never
         containers:
         - name: container0
           image: ubuntu:24.04
           command: ["sleep", "9999"]
           resources:
             claims:
             - name: separate-gpu-claim
         - name: container1
           image: ubuntu:24.04
           command: ["sleep", "9999"]
           resources:
             claims:
             - name: shared-gpu-claim
         - name: container2
           image: ubuntu:24.04
           command: ["sleep", "9999"]
           resources:
             claims:
             - name: shared-gpu-claim
         resourceClaims:
         - name: separate-gpu-claim
           resourceClaimTemplateName: example-resource-claim-template
         - name: shared-gpu-claim
           resourceClaimName: example-resource-claim
   ```

   Mỗi Pod trong Job này có các đặc điểm sau:

   * Cung cấp cho các container một ResourceClaimTemplate tên là `separate-gpu-claim` và một
     ResourceClaim tên là `shared-gpu-claim`.
   * Chạy các container sau:
       * `container0` yêu cầu các thiết bị từ ResourceClaimTemplate
         `separate-gpu-claim`.
       * `container1` và `container2` chia sẻ quyền truy cập vào các thiết bị từ
         ResourceClaim `shared-gpu-claim`.

1. Tạo Job:

   ```shell
   kubectl apply -f https://k8s.io/examples/dra/dra-example-job.yaml
   ```

Hãy thử các bước khắc phục sự cố sau:

1. Khi workload không khởi động như mong đợi, hãy đi sâu dần từ Job
   xuống các Pod rồi tới các ResourceClaim, và kiểm tra các đối tượng
   ở từng cấp bằng `kubectl describe` để xem có field trạng thái hay
   sự kiện (event) nào giải thích vì sao workload
   không khởi động hay không.
1. Khi việc tạo Pod thất bại với lỗi `must specify one of: resourceClaimName,
   resourceClaimTemplateName`, hãy kiểm tra rằng mọi mục trong `pod.spec.resourceClaims`
   đều có đúng một trong hai field đó được đặt. Nếu chúng đã đúng, thì có khả năng
   cluster đã cài một mutating Pod webhook được build
   dựa trên các API của Kubernetes < 1.32. Hãy làm việc với quản trị viên cluster
   để kiểm tra điều này.

## Dọn dẹp (Clean up) {#clean-up}

Để xóa các đối tượng Kubernetes mà bạn đã tạo trong bài thực hành này, làm theo các
bước sau:

1.  Xóa Job ví dụ:

    ```shell
    kubectl delete -f https://k8s.io/examples/dra/dra-example-job.yaml
    ```

1.  Để xóa các resource claim của bạn, chạy một trong các lệnh sau:

    * Xóa ResourceClaimTemplate:

      ```shell
      kubectl delete -f https://k8s.io/examples/dra/resourceclaimtemplate.yaml
      ```
    * Xóa ResourceClaim:

      ```shell
      kubectl delete -f https://k8s.io/examples/dra/resourceclaim.yaml
      ```

## Tiếp theo (What's next)

* [Tìm hiểu thêm về DRA](149-dynamic-resource-allocation-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 13:

1. Bài cho hai cách claim tài nguyên. Tiêu chí nào quyết định dùng ResourceClaim tạo thủ công, và
   tiêu chí nào quyết định dùng ResourceClaimTemplate?
2. Một Pod yêu cầu thiết bị bằng DRA phải viết vào **hai** chỗ trong spec. Hai chỗ đó là gì, và
   ràng buộc "đúng một trong hai field" áp cho chỗ nào?
3. **Câu bẫy.** Trong Job ví dụ, `container1` và `container2` cùng ghi `shared-gpu-claim`, còn
   `container0` ghi `separate-gpu-claim`. Ba container đó dùng ba bộ thiết bị riêng phải không? Và
   khi Job chạy `parallelism: 2`, hai Pod có dùng chung `separate-gpu-claim` không?
4. Trên cluster lab bạn tạo một Pod tham chiếu `resourceClaimName: example-resource-claim` mà quên
   tạo ResourceClaim đó trước. Pod rơi vào trạng thái nào, nó có bao giờ được đặt lên
   `lab-k8s-worker1` hay `lab-k8s-worker2` không, và bài bảo lần theo trình tự nào để tìm nguyên
   nhân?
5. Bạn chạy `kubectl get deviceclasses` và nhận lỗi về quyền. Theo bài, đó là dấu hiệu của chuyện
   gì và bạn phải làm gì tiếp?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Dùng **ResourceClaim tạo thủ công** khi muốn **nhiều Pod chia sẻ quyền truy cập cùng một nhóm
   thiết bị**, hoặc khi muốn **claim tồn tại lâu hơn vòng đời của một Pod**. Dùng
   **ResourceClaimTemplate** khi muốn **mỗi Pod có thiết bị riêng biệt nhưng cấu hình tương tự
   nhau** — bài lấy ví dụ các Pod của một Job chạy song song; khi đó Kubernetes tự sinh và quản lý
   một ResourceClaim cho từng Pod.
2. Chỗ thứ nhất: **`pod.spec.resourceClaims`** — khai báo claim ở cấp Pod. Chỗ thứ hai:
   **`resources.claims` của container** — gọi lại claim đó theo tên. Ràng buộc "đúng một trong
   `resourceClaimName` hoặc `resourceClaimTemplateName`" áp cho **từng mục của
   `pod.spec.resourceClaims`**; vi phạm sẽ ra lỗi `must specify one of: resourceClaimName,
   resourceClaimTemplateName`.
3. **Không.** `container1` và `container2` **chia sẻ quyền truy cập vào các thiết bị từ
   ResourceClaim `shared-gpu-claim`** — một claim, dùng chung. Chỉ `container0` mới có phần riêng,
   vì `separate-gpu-claim` trỏ tới một **ResourceClaimTemplate**. Vế sau cũng vậy: template nghĩa
   là **Kubernetes sinh một ResourceClaim riêng cho từng Pod**, nên hai Pod chạy song song
   **không** dùng chung — đây đúng là chỗ dễ nhầm, vì trong manifest hai loại nằm cạnh nhau và
   trông giống nhau, chỉ khác đúng tên field `resourceClaimName` hay `resourceClaimTemplateName`.
4. Pod **ở trạng thái pending cho đến khi ResourceClaim được tạo** — tham chiếu tới một
   ResourceClaim chưa tồn tại không làm hỏng việc tạo Pod. Và **không**: pending nghĩa là Pod chưa
   được lập lịch, nên nó **không hề được đặt lên `lab-k8s-worker1` hay `lab-k8s-worker2`** — nó chỉ
   nằm chờ. Trình tự tìm nguyên nhân của bài: đi **sâu dần từ Job xuống các Pod rồi tới các
   ResourceClaim**, dùng `kubectl describe` ở từng cấp để xem có field trạng thái hay event nào
   giải thích vì sao workload không khởi động.
5. Đó là dấu hiệu **bạn không có quyền lấy danh sách DeviceClass**, chứ không phải cluster hỏng.
   Bài bảo **hỏi quản trị viên cluster hoặc nhà cung cấp driver** về các thuộc tính thiết bị khả
   dụng — vì DeviceClass do quản trị viên và driver tạo ra, không phải do người vận hành workload.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

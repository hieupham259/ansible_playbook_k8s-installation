# Lập lịch GPU (Schedule GPUs)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/>
>
> Cấu hình và lập lịch GPU để dùng như một resource của các node trong cluster.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 29 — DaemonSet, Job nâng cao và thiết bị chuyên dụng](00-ALO-TRINH-ADMIN.md#giai-đoạn-29--daemonset-job-nâng-cao-và-thiết-bị-chuyên-dụng),
bài 8/8 · **Không kiểm chứng được trên cluster lab** — bài đọc-hiểu; xem phần cảnh báo ngay dưới.
Bài thuộc nhóm [giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao) và
nối tiếp bài [184](184-device-plugins-vi.md).

**Ba VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) không có GPU.** `lab-k8s-master`,
`lab-k8s-worker1` và `lab-k8s-worker2` là máy ảo với 2 vCPU và 6 GB RAM, không thiết bị chuyên dụng
nào. Đây không phải phỏng đoán: [Lab 13](labs/LAB-13-DRA.md) đã chạy
[năm phép kiểm năng lực](labs/LAB-13-DRA.md#b1-kiểm-tra-năng-lực-dra--chọn-nhánh) và kết luận cluster
này thiếu đúng điều kiện phần cứng đó. Hệ quả cho bài này, nói thẳng:

- **Không có gì để chạy.** Không có GPU thì không cài driver được, không chạy device plugin được, và
  cluster không expose resource nào dạng `nvidia.com/gpu`. Cài driver hay device plugin còn là tải
  phần mềm bên thứ ba về node — nằm ngoài baseline đã khóa của Lab 00.
- **Vẫn đọc được một thứ có thật:** `kubectl describe node lab-k8s-worker1` cho thấy phần `Capacity`
  **không** có dòng resource nào của nhà cung cấp GPU. Đó là bằng chứng cho đúng điều bài nói — tài
  nguyên GPU chỉ xuất hiện **sau khi** device plugin báo cáo nó.

Nên đọc bài ở mức **hiểu ai làm việc gì và cú pháp khai báo ra sao**. Đây cũng là bài cuối của giai
đoạn 29.

**Phải hiểu ở lần đọc này:**

- Trách nhiệm của quản trị viên ở mục *Sử dụng device plugin*: Kubernetes tự nó **không biết** GPU;
  bạn phải cài **driver của nhà cung cấp lên node** *và* chạy **device plugin do nhà cung cấp đó
  phát hành**. Chỉ sau đó cluster mới expose một resource lập lịch được như `amd.com/gpu` hay
  `nvidia.com/gpu`.
- Ba quy tắc khai báo GPU, khác hẳn `cpu`/`memory`: GPU **chỉ được khai trong `limits`**; khai
  `limits` mà bỏ `requests` là hợp lệ vì Kubernetes lấy limit làm request; khai cả hai thì **bắt
  buộc bằng nhau**; và **không** được khai `requests` mà thiếu `limits`.
- Tên resource là **do nhà cung cấp đặt**, dạng miền — manifest ví dụ dùng
  `gpu-vendor.example/example-gpu`. Nó không phải một trường chuẩn của Kubernetes như `cpu`, nên bạn
  phải biết tên mà plugin trên cluster của bạn công bố.
- Cluster có nhiều loại GPU thì phân biệt bằng **node label và node selector** (mục *Quản lý cluster
  có nhiều loại GPU khác nhau*) — khóa `accelerator` trong ví dụ chỉ là một khóa tự đặt, không phải
  quy ước bắt buộc.
- Mục *Tự động gắn label cho node*: Node Feature Discovery tự phát hiện tính năng phần cứng của từng
  node và quảng bá chúng thành node label, và còn có thể thêm **extended resource, annotation và
  node taint** — dùng taint để chỉ Pod nào cần tính năng đó mới lên được node đó. Manifest thứ hai
  cho thấy cách ghép `nodeAffinity` với label của NFD và label riêng của nhà cung cấp.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ba link hướng dẫn cài của AMD, Intel, NVIDIA và mục *Các triển khai của nhà cung cấp GPU* | Không có GPU thì driver không có thiết bị nào để điều khiển; cài chúng cũng là đưa phần mềm bên thứ ba vào node | Ràng buộc phần cứng thật — [Lab 13](labs/LAB-13-DRA.md) đã chứng minh bằng [năm phép kiểm](labs/LAB-13-DRA.md#b1-kiểm-tra-năng-lực-dra--chọn-nhánh) rằng cluster này không có thiết bị chuyên dụng |
| Triển khai Node Feature Discovery và các feature label của nó | NFD là add-on phải cài thêm, và trên VM không GPU nó không có tính năng GPU nào để phát hiện | Ràng buộc phần cứng thật — chỉ cần hiểu vai trò: NFD thay bạn gắn label thay vì gõ `kubectl label` từng node |
| Cú pháp `nodeAffinity` với `operator: Gt` trong manifest thứ hai | Node affinity đã học kỹ rồi; ở đây nó chỉ là chỗ áp dụng | Bài [138](138-assign-pod-node-vi.md) ở [giai đoạn 7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction), thực hành ở [Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md) phần B2 |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

Kubernetes có hỗ trợ **ổn định (stable)** cho việc quản lý GPU (graphical processing unit —
bộ xử lý đồ họa) của AMD và NVIDIA trên các node khác nhau trong cluster của bạn, thông qua
device plugin.

Trang này mô tả cách người dùng sử dụng GPU, đồng thời nêu ra một số giới hạn trong cách
triển khai hiện tại.

## Sử dụng device plugin (Using device plugins)

Kubernetes triển khai device plugin để cho phép các Pod truy cập những tính năng phần cứng
chuyên dụng, chẳng hạn như GPU.

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes
> cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn
> được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc
> [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> trước khi gửi thay đổi.

Với vai trò quản trị viên (administrator), bạn phải cài driver GPU của nhà cung cấp phần cứng
tương ứng lên các node và chạy device plugin tương ứng do nhà cung cấp GPU đó phát hành. Dưới
đây là một số liên kết tới hướng dẫn của các nhà cung cấp:

* [AMD](https://github.com/ROCm/k8s-device-plugin#deployment)
* [Intel](https://intel.github.io/intel-device-plugins-for-kubernetes/cmd/gpu_plugin/README.html)
* [NVIDIA](https://github.com/NVIDIA/k8s-device-plugin#quick-start)

Sau khi bạn đã cài plugin, cluster của bạn sẽ expose một resource tùy chỉnh có thể lập lịch
được, chẳng hạn `amd.com/gpu` hoặc `nvidia.com/gpu`.

Bạn có thể sử dụng những GPU này từ trong container bằng cách yêu cầu resource GPU tùy chỉnh
đó, hoàn toàn giống cách bạn yêu cầu `cpu` hay `memory`. Tuy nhiên, có một vài giới hạn trong
cách bạn khai báo yêu cầu resource cho các thiết bị tùy chỉnh.

GPU chỉ được phép khai báo trong phần `limits`, điều đó có nghĩa là:
* Bạn có thể khai báo `limits` cho GPU mà không cần khai báo `requests`, bởi vì Kubernetes sẽ
  mặc định lấy giá trị limit làm giá trị request.
* Bạn có thể khai báo GPU ở cả `limits` lẫn `requests`, nhưng hai giá trị này bắt buộc phải
  bằng nhau.
* Bạn không thể khai báo `requests` cho GPU mà không khai báo `limits`.

Dưới đây là một manifest ví dụ cho Pod yêu cầu một GPU:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-vector-add
spec:
  restartPolicy: OnFailure
  containers:
    - name: example-vector-add
      image: "registry.example/example-vector-add:v42"
      resources:
        limits:
          gpu-vendor.example/example-gpu: 1 # yêu cầu 1 GPU
```

## Quản lý cluster có nhiều loại GPU khác nhau (Manage clusters with different types of GPUs)

Nếu các node khác nhau trong cluster của bạn có các loại GPU khác nhau, bạn có thể dùng
[Node Label và Node Selector](267-assign-pods-nodes-vi.md) để lập lịch pod lên đúng node phù hợp.

Ví dụ:

```shell
# Gắn label cho các node theo loại accelerator mà chúng có.
kubectl label nodes node1 accelerator=example-gpu-x100
kubectl label nodes node2 accelerator=other-gpu-k915
```

Khóa label `accelerator` ở trên chỉ là một ví dụ; bạn có thể dùng một khóa label khác nếu muốn.

## Tự động gắn label cho node (Automatic node labelling) {#node-labeller}

Với vai trò quản trị viên, bạn có thể tự động phát hiện và gắn label cho tất cả các node có GPU
bằng cách triển khai [Node Feature Discovery](https://github.com/kubernetes-sigs/node-feature-discovery)
(NFD) của Kubernetes. NFD phát hiện các tính năng phần cứng sẵn có trên từng node trong một
cluster Kubernetes. Thông thường, NFD được cấu hình để quảng bá những tính năng đó dưới dạng
node label, nhưng NFD cũng có thể thêm extended resource, annotation và node taint. NFD tương
thích với mọi [phiên bản được hỗ trợ](https://kubernetes.io/releases/version-skew-policy/#supported-versions)
của Kubernetes. Theo mặc định, NFD tạo các
[feature label](https://kubernetes-sigs.github.io/node-feature-discovery/master/usage/features.html)
cho những tính năng mà nó phát hiện được. Quản trị viên có thể tận dụng NFD để đồng thời gắn
taint cho các node có tính năng cụ thể, sao cho chỉ những pod yêu cầu các tính năng đó mới được
lập lịch lên những node ấy.

Bạn cũng cần một plugin cho NFD để thêm các label phù hợp vào node của mình; đó có thể là các
label chung chung hoặc các label riêng của từng nhà cung cấp. Nhà cung cấp GPU của bạn có thể
cung cấp một plugin bên thứ ba cho NFD; hãy xem tài liệu của họ để biết thêm chi tiết.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-vector-add
spec:
  restartPolicy: OnFailure
  # Bạn có thể dùng node affinity của Kubernetes để lập lịch Pod này lên một node
  # cung cấp đúng loại GPU mà container của nó cần để hoạt động
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: "gpu.gpu-vendor.example/installed-memory"
            operator: Gt # (lớn hơn)
            values: ["40535"]
          - key: "feature.node.kubernetes.io/pci-10.present" # Feature label của NFD
            values: ["true"] # (tùy chọn) chỉ lập lịch lên các node có thiết bị PCI 10
  containers:
    - name: example-vector-add
      image: "registry.example/example-vector-add:v42"
      resources:
        limits:
          gpu-vendor.example/example-gpu: 1 # yêu cầu 1 GPU
```

#### Các triển khai của nhà cung cấp GPU (GPU vendor implementations)

- [Intel](https://intel.github.io/intel-device-plugins-for-kubernetes/cmd/gpu_plugin/README.html)
- [NVIDIA](https://github.com/NVIDIA/k8s-device-plugin)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 29:

1. `kubectl describe node lab-k8s-worker2` không có dòng nào kiểu `nvidia.com/gpu` ở phần
   `Capacity`. Kể ra **hai** điều kiện mà cluster lab đang thiếu, và nói rõ thiếu điều kiện nào thì
   dù có điều kiện kia resource đó vẫn không xuất hiện.
2. **Câu bẫy.** Bạn quen khai `requests` cho `cpu` rồi để `limits` trống. Với GPU, manifest chỉ có
   `requests: {gpu-vendor.example/example-gpu: 1}` có hợp lệ không? Còn `limits: 2` kèm
   `requests: 1` thì sao? Nếu chỉ khai `limits`, giá trị request là bao nhiêu?
3. Trong manifest ví dụ, resource được viết là `gpu-vendor.example/example-gpu` chứ không phải
   `gpu`. Ai đặt cái tên đó, và làm sao bạn biết tên đúng cần dùng trên một cluster cụ thể?
4. Cluster có hai node mang hai loại GPU khác nhau, và bạn cần Pod chạy đúng trên loại thứ nhất.
   Bài đề xuất cơ chế nào? Nếu không muốn gắn label bằng tay cho từng node thì dùng gì, và công cụ
   đó thêm được những thứ gì ngoài label?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Thiếu **phần cứng GPU trên node** và thiếu **device plugin của nhà cung cấp đang chạy trên
   cluster** (kèm driver được cài lên node). Thiếu device plugin là đủ để resource không xuất hiện —
   **Kubernetes chỉ biết tới GPU qua báo cáo của device plugin**, nên kể cả khi cắm GPU vào máy mà
   không chạy plugin thì `Capacity` vẫn trống. Chiều ngược lại cũng vậy: không có phần cứng thì
   plugin không có gì để báo cáo.
2. **Không hợp lệ** — bài nói thẳng: bạn **không thể khai `requests` cho GPU mà không khai
   `limits`**. `limits: 2` kèm `requests: 1` cũng **không hợp lệ**, vì khai cả hai thì hai giá trị
   **bắt buộc bằng nhau**. Chỉ khai `limits` thì **request tự động bằng chính giá trị limit**. Chỗ
   dễ nhầm là mang thói quen từ `cpu` sang: với CPU, requests và limits là hai con số độc lập phục
   vụ hai việc khác nhau; GPU là **thiết bị nguyên chiếc, không chia nhỏ và không overcommit**, nên
   Kubernetes bắt hai con số phải là một.
3. **Nhà cung cấp GPU đặt**, qua device plugin của họ — vì thế tên có dạng miền như `amd.com/gpu`
   hay `nvidia.com/gpu`, còn `gpu-vendor.example/example-gpu` chỉ là tên giả trong ví dụ. Nó
   **không phải trường chuẩn của Kubernetes** như `cpu` hay `memory`. Cách biết tên đúng: xem tài
   liệu của plugin đang chạy, và đọc phần `Capacity` của node — resource mà plugin công bố sẽ hiện
   ra ở đó.
4. Dùng **node label kèm node selector**: gắn nhãn cho từng node theo loại accelerator rồi cho Pod
   chọn nhãn đó — khóa `accelerator` trong ví dụ chỉ là một khóa tự đặt. Không muốn gắn tay thì dùng
   **Node Feature Discovery**: nó tự phát hiện tính năng phần cứng của từng node và quảng bá thành
   label, đồng thời còn thêm được **extended resource, annotation và node taint** — taint để chỉ Pod
   yêu cầu đúng tính năng đó mới được lập lịch lên node ấy.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là **bài cuối của giai đoạn 29** —
làm tiếp **Checkpoint** ở cuối
[Giai đoạn 29 — DaemonSet, Job nâng cao và thiết bị chuyên dụng](00-ALO-TRINH-ADMIN.md#giai-đoạn-29--daemonset-job-nâng-cao-và-thiết-bị-chuyên-dụng):
triển khai một DaemonSet, rolling update rồi rollback về revision trước và quan sát `ControllerRevision`;
sau đó chạy một Job có `podFailurePolicy` và chứng minh Job dừng đúng theo exit code đã khai báo thay
vì thử lại tới `backoffLimit`. Hai việc đó nằm ở bài [388](388-update-daemon-set-vi.md),
[387](387-rollback-daemon-set-vi.md) và [383](383-pod-failure-policy-vi.md) — không phải ở bài này.

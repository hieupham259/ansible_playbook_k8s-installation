# Cấu hình Pod và Container (Configure Pods and Containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/>
>
> Thực hiện các tác vụ cấu hình phổ biến cho Pod và container.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3a. Pod và vòng đời](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài thực hành 2/11 ·
[Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) **không kiểm chứng bài này**: mục 1.1 của lab ghi rõ đây
là **trang mục lục**, không có nội dung thực hành riêng, nó chỉ trỏ tới các bài đã nằm trong bảng
ánh xạ của lab.

Đọc trang này như một **bản đồ**, không phải một bài học. Mất năm phút để thấy Kubernetes gom
những gì vào cụm "cấu hình Pod và container", rồi quay lại tra khi cần.

**Phải hiểu ở lần đọc này:**

- Trang gốc **không có nội dung riêng**: nó chỉ là danh sách liên kết của mục tác vụ *Configure
  Pods and Containers*. Giá trị duy nhất của nó là cho bạn thấy toàn bộ phạm vi cùng một lúc.
- Danh sách này **không xếp theo thứ tự học**. Chỉ sáu mục thuộc nhóm 3a — probe
  ([274](274-configure-probes-vi.md)), khởi tạo Pod ([276](276-configure-pod-initialization-vi.md)),
  handler vòng đời ([272](272-attach-handler-lifecycle-event-vi.md)), process namespace
  ([292](292-share-process-namespace-vi.md)), user namespace
  ([295](295-user-namespaces-tasks-vi.md)) và image volume
  ([285](285-image-volumes-vi.md)) — phần còn lại rải khắp các giai đoạn sau. Đừng đọc tuần tự từ
  trên xuống.
- Ba mục dành cho Windows — GMSA, `RunAsUserName`, HostProcess Pod — chỉ có nghĩa khi cluster có
  node Windows.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Các mục về tài nguyên: gán CPU/memory, tài nguyên cấp Pod, resize, extended resource, QoS | đều dựa trên `requests` và `limits`, chưa học | nhóm [3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn), thực hành ở Lab 3c |
| ConfigMap, private registry, projected volume, service account cho Pod | cần ConfigMap và Secret | nhóm [3b](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod), thực hành ở Lab 3b |
| Gán Pod vào Node, gán Pod bằng node affinity, gán thiết bị cho Pod | thuộc lập lịch | [giai đoạn 7](00-ALO-TRINH-ADMIN.md#giai-đoạn-7--lập-lịch-và-chính-sách-tài-nguyên), thực hành ở Lab 7a |
| Security context, ba mục Pod Security Standards, di chuyển khỏi PodSecurityPolicy | thuộc bảo mật | [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), thực hành ở Lab 9b |
| GMSA, `RunAsUserName`, HostProcess Pod | cluster lab chỉ có ba VM Linux | [giai đoạn 15](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows) |
| Tạo static Pod, chuyển file Docker Compose sang tài nguyên Kubernetes | static Pod thuộc 3c; Docker Compose cần Service | [nhóm 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) và [giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) |

---

Đây là trang mục lục của mục tác vụ (task) *Configure Pods and Containers* trong tài liệu
Kubernetes. Trang gốc không có nội dung riêng, chỉ liệt kê các trang hướng dẫn thực hiện những
tác vụ cấu hình phổ biến cho Pod và container:

* [Gán tài nguyên memory cho Container và Pod (Assign Memory Resources to Containers and Pods)](./264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU cho Container và Pod (Assign CPU Resources to Containers and Pods)](./263-assign-cpu-resource-vi.md)

* [Gán thiết bị cho Pod và Container (Assign Devices to Pods and Containers)](268-assign-resources-vi.md)

  Gán tài nguyên hạ tầng cho các workload Kubernetes của bạn.

* [Gán tài nguyên CPU và memory ở cấp Pod (Assign Pod-level CPU and memory resources)](265-assign-pod-level-resources-vi.md)

* [Cấu hình GMSA cho Pod và container Windows (Configure GMSA for Windows Pods and containers)](273-configure-gmsa-vi.md)

* [Thay đổi tài nguyên CPU và memory đã gán cho Container (Resize CPU and Memory Resources assigned to Containers)](289-resize-container-resources-vi.md)

* [Thay đổi tài nguyên CPU và memory đã gán cho Pod (Resize CPU and Memory Resources assigned to Pods)](290-resize-pod-resources-vi.md)

* [Cấu hình RunAsUserName cho pod và container Windows (Configure RunAsUserName for Windows pods and containers)](278-configure-runasusername-vi.md)

* [Tạo một Pod HostProcess Windows (Create a Windows HostProcess Pod)](281-create-hostprocess-pod-vi.md)

* [Cấu hình Quality of Service cho Pod (Configure Quality of Service for Pods)](288-quality-service-pod-vi.md)

* [Gán Extended Resource cho một Container (Assign Extended Resources to a Container)](284-extended-resource-vi.md)

* [Cấu hình Pod dùng Volume để lưu trữ (Configure a Pod to Use a Volume for Storage)](280-configure-volume-storage-vi.md)

* [Cấu hình Pod dùng Projected Volume để lưu trữ (Configure a Pod to Use a Projected Volume for Storage)](277-configure-projected-volume-vi.md)

* [Cấu hình Security Context cho Pod hoặc Container (Configure a Security Context for a Pod or Container)](291-security-context-vi.md)

* [Cấu hình Service Account cho Pod (Configure Service Accounts for Pods)](279-configure-service-account-vi.md)

* [Kéo (pull) image từ một private registry (Pull an Image from a Private Registry)](287-pull-image-private-registry-vi.md)

* [Cấu hình Liveness, Readiness và Startup Probe (Configure Liveness, Readiness and Startup Probes)](274-configure-probes-vi.md)

* [Gán Pod vào Node (Assign Pods to Nodes)](267-assign-pods-nodes-vi.md)

* [Gán Pod vào Node bằng Node Affinity (Assign Pods to Nodes using Node Affinity)](266-assign-pods-nodes-node-affinity-vi.md)

* [Cấu hình khởi tạo Pod (Configure Pod Initialization)](276-configure-pod-initialization-vi.md)

* [Gắn handler vào các sự kiện vòng đời Container (Attach Handlers to Container Lifecycle Events)](272-attach-handler-lifecycle-event-vi.md)

* [Cấu hình Pod dùng ConfigMap (Configure a Pod to Use a ConfigMap)](275-configure-pod-configmap-vi.md)

* [Chia sẻ process namespace giữa các container trong một Pod (Share Process Namespace between Containers in a Pod)](292-share-process-namespace-vi.md)

* [Dùng user namespace với Pod (Use a User Namespace With a Pod)](295-user-namespaces-tasks-vi.md)

* [Dùng image volume với Pod (Use an Image Volume With a Pod)](285-image-volumes-vi.md)

* [Tạo static Pod (Create static Pods)](293-static-pod-tasks-vi.md)

* [Chuyển đổi file Docker Compose sang tài nguyên Kubernetes (Translate a Docker Compose File to Kubernetes Resources)](294-translate-compose-kubernetes-vi.md)

* [Áp dụng Pod Security Standards bằng cách cấu hình Admission Controller tích hợp sẵn (Enforce Pod Security Standards by Configuring the Built-in Admission Controller)](282-enforce-standards-admission-controller-vi.md)

* [Áp dụng Pod Security Standards bằng label của Namespace (Enforce Pod Security Standards with Namespace Labels)](283-enforce-standards-namespace-labels-vi.md)

* [Di chuyển từ PodSecurityPolicy sang Admission Controller PodSecurity tích hợp sẵn (Migrate from PodSecurityPolicy to the Built-In PodSecurity Admission Controller)](286-migrate-from-psp-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Trang này không có nội dung riêng. Vậy dùng nó thế nào cho đúng, và vì sao **không** nên đọc
   tuần tự từ mục đầu tới mục cuối?
2. **Câu bẫy.** Danh sách đặt "Cấu hình Quality of Service cho Pod" và "Gán tài nguyên memory cho
   Container và Pod" ngay cạnh "Cấu hình khởi tạo Pod". Ba mục nằm sát nhau trong danh sách có
   nghĩa là chúng cùng một mức độ ưu tiên ở lần đọc này không?
3. Cluster lab của bạn là ba VM Linux — `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`.
   Những mục nào trong danh sách chắc chắn **không** làm được trên cluster đó, dù bạn đã học đủ
   kiến thức?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Dùng nó như **bản đồ để định vị và tra cứu**: nhìn một lượt để biết Kubernetes gom những tác
   vụ nào vào cụm "cấu hình Pod và container", rồi mở đúng trang cần khi gặp việc. **Không** đọc
   tuần tự, vì danh sách **không xếp theo thứ tự học**: nó trộn lẫn những thứ chỉ cần Pod trần
   với những thứ đòi ConfigMap, Secret, `requests`/`limits`, scheduling và bảo mật — tức kiến
   thức của các giai đoạn sau.
2. **Không.** Vị trí trong danh sách chỉ là thứ tự trang gốc, không mang thông tin gì về lộ
   trình. "Cấu hình khởi tạo Pod" là bài [276](276-configure-pod-initialization-vi.md) thuộc
   đúng nhóm 3a và bạn làm được ngay; còn QoS và gán tài nguyên memory đều dựng trên `requests`
   và `limits` nên thuộc nhóm [3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn). Đây
   chính là lý do trang này chỉ nên dùng để định vị, không dùng làm thứ tự đọc.
3. **Ba mục dành cho Windows**: cấu hình GMSA cho Pod và container Windows, cấu hình
   `RunAsUserName` cho pod và container Windows, và tạo một Pod HostProcess Windows. Chúng cần
   node Windows, mà `lab-k8s-master`, `lab-k8s-worker1` và `lab-k8s-worker2` đều là Linux — nên
   đây là giới hạn của môi trường, không phải giới hạn của kiến thức.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

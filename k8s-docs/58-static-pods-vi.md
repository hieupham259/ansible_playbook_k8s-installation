# Pod tĩnh (Static Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/static-pods/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3b](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod), bài 7/7 ·
Kiểm chứng ở [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md).

Bài ngắn nhất nhóm, nhưng nó giải thích chính **cluster của bạn**: `kube-apiserver`,
`kube-controller-manager`, `kube-scheduler` và `etcd` trên `lab-k8s-master` đều là static Pod. Bạn
đã nhìn thấy chúng dưới dạng file manifest ở
[Lab 1a](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md#b2-kiểm-kê-component) mà chưa được
gọi tên; bài này là cái tên đó. Nó cũng khép lại nhóm 3b bằng cách giải thích vì sao bài
[109](109-secret-vi.md) nói static Pod không dùng được Secret.

**Phải hiểu ở lần đọc này:**

- Static Pod do **kubelet quản lý trực tiếp trên một node cụ thể**, không có API server quan
  sát; kubelet tự theo dõi và khởi động lại nó khi gặp sự cố.
- **Mirror Pod**: kubelet tự tạo một bản phản chiếu trên API server để Pod đó *nhìn thấy được*
  nhưng **không điều khiển được** từ đó; tên có hậu tố là hostname của node. Xóa mirror Pod
  bằng `kubectl` **không** xóa static Pod — kubelet tạo lại mirror Pod. Label được lan truyền
  sang mirror Pod nên selector vẫn dùng được như bình thường.
- Giới hạn: spec static Pod **không tham chiếu được đối tượng API khác** — ServiceAccount,
  ConfigMap, Secret; và không hỗ trợ ephemeral container.
- Lý do tồn tại: static Pod được kubelet khởi động **trước khi API server sẵn sàng**, nên nó
  là cách bootstrap các thành phần control plane — đúng cách kubeadm dựng cluster của bạn.
  DaemonSet thì **yêu cầu một control plane đang chạy**.
- Ranh giới chọn công cụ: static Pod không rollout, rollback hay co giãn được bằng cơ chế tiêu
  chuẩn; workload chạy trên mọi node nên dùng **DaemonSet**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cách tạo static Pod (link *tạo static Pod* ở mục cuối) | là thao tác dựng cluster | giai đoạn 8, bài [02](02-create-cluster-kubeadm-vi.md) |
| DaemonSet như một lựa chọn thay thế | chưa học controller | giai đoạn 4, bài [66](66-daemonset-vi.md) |
| Annotation `kubernetes.io/config.mirror` | chi tiết nhận diện, dùng khi tra cứu | không cần |
| Vì sao static Pod là rủi ro bảo mật (chạy được Pod mà không qua API server) | cần mô hình kiểm soát truy cập | giai đoạn 9, bài [128](128-api-server-bypass-risks-vi.md) |

---

_Static Pod_ (Pod tĩnh) được quản lý trực tiếp bởi kubelet daemon trên một node cụ thể,
mà không có API server quan sát chúng.
Khác với các Pod được quản lý bởi control plane (ví dụ như một Deployment),
kubelet theo dõi từng static Pod và khởi động lại nó nếu nó gặp sự cố.

Static Pod luôn được gắn với một kubelet trên một node cụ thể.

Công dụng chính của static Pod là để chạy một control plane tự lưu trữ (self-hosted):
nói cách khác, dùng kubelet để giám sát từng
[thành phần control plane](./15-components-vi.md) riêng lẻ.
Ví dụ, [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) dùng static Pod để chạy
`kube-apiserver`, `kube-controller-manager`, `kube-scheduler` và `etcd` trên các node control plane.

> **Ghi chú:**
> Nếu cluster của bạn chạy các thành phần control plane dưới dạng Pod, nhiều khả năng
> chúng là static Pod. Bạn có thể nhận ra các mirror Pod của chúng trong namespace
> `kube-system` thông qua annotation `kubernetes.io/config.mirror`.

## Mirror Pod (Mirror Pods) {#mirror-pods}

Kubelet tự động cố gắng tạo một mirror Pod (Pod phản chiếu)
trên Kubernetes API server cho mỗi static Pod.
Điều này có nghĩa là các Pod đang chạy trên một node sẽ được nhìn thấy trên API server,
nhưng không thể bị điều khiển từ đó.
Tên của Pod sẽ có hậu tố là hostname của node, ngăn cách bằng một dấu gạch nối ở đầu.

Kubelet lan truyền (propagate) các label từ static Pod sang mirror Pod. Bạn có thể
dùng các label đó như bình thường thông qua các selector.

Nếu bạn thử dùng `kubectl` để xóa mirror Pod khỏi API server,
kubelet _không_ xóa static Pod. Kubelet sẽ tạo lại
mirror Pod.

## Giới hạn (Limitations) {#limitations}

Spec của một static Pod không thể tham chiếu đến các đối tượng API khác,
chẳng hạn như ServiceAccount, ConfigMap hay Secret.

Static Pod không hỗ trợ [ephemeral container](./52-ephemeral-containers-vi.md).

## Static Pod so với DaemonSet (Static Pods vs DaemonSets) {#static-pods-vs-daemonsets}

Nếu bạn đang vận hành Kubernetes dạng cluster và dùng static Pod để chạy một Pod
trên mọi node, thì có lẽ bạn nên dùng
DaemonSet thay thế.

Static Pod không được control plane quản lý, vì vậy chúng không thể được triển khai
cuốn chiếu (rolled out), hoàn tác (rolled back) hay co giãn (scaled) bằng các cơ chế
tiêu chuẩn của Kubernetes. DaemonSet cung cấp các khả năng này và là cách tiếp cận
được khuyến nghị để chạy các workload ở cấp node.

Static Pod được kubelet khởi động trước khi API server sẵn sàng, điều này
khiến chúng phù hợp cho việc khởi tạo (bootstrap) các thành phần control plane.
DaemonSet yêu cầu một control plane đang chạy.

## Tiếp theo (What's next)

- Tìm hiểu cách [tạo static Pod](293-static-pod-tasks-vi.md).
- Tìm hiểu về [các thành phần Kubernetes](./15-components-vi.md) và cách control plane sử dụng static Pod.
- Tìm hiểu về [DaemonSet](66-daemonset-vi.md) như một lựa chọn thay thế cho static Pod.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Trên `lab-k8s-master`, bạn chạy `kubectl delete pod kube-apiserver-lab-k8s-master -n kube-system`.
   Chuyện gì xảy ra với tiến trình API server đang chạy trên node đó?
2. Các Pod control plane hiện ra trong `kubectl get pods -n kube-system`. Vậy control plane
   có quản lý được chúng qua API không?
3. Vì sao kubeadm chạy `kube-apiserver` bằng static Pod chứ không phải bằng DaemonSet — vốn
   cũng là mô hình "mỗi node một Pod"?
4. Bạn muốn chạy một agent thu thập log trên cả `lab-k8s-worker1` và `lab-k8s-worker2`. Dùng static
   Pod hay DaemonSet, và vì sao?
5. Bạn muốn một static Pod đọc mật khẩu từ một Secret. Có làm được không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không có gì xảy ra với tiến trình thật.** Thứ bạn vừa xóa chỉ là **mirror Pod** trên API
   server. Bài nói rõ: nếu bạn thử dùng `kubectl` để xóa mirror Pod khỏi API server, kubelet
   **không** xóa static Pod, và nó sẽ **tạo lại mirror Pod**. Static Pod gắn với kubelet của
   đúng node đó, và kubelet còn tự khởi động lại nó nếu nó gặp sự cố.
2. **Không.** Đây là điểm dễ nhầm nhất của bài: nhìn thấy trong `kubectl get pods` nên trực
   giác cho rằng chúng là Pod bình thường do control plane quản lý. Mirror Pod tồn tại đúng để
   "các Pod đang chạy trên một node sẽ **được nhìn thấy** trên API server, **nhưng không thể
   bị điều khiển từ đó**". Dấu hiệu nhận ra chúng: tên có hậu tố là hostname của node —
   `kube-apiserver-lab-k8s-master` — và annotation `kubernetes.io/config.mirror`.
3. Vì **DaemonSet yêu cầu một control plane đang chạy**, mà `kube-apiserver` *chính là* control
   plane đó — dùng DaemonSet sẽ tạo vòng lặp phụ thuộc không khởi động được. Static Pod thì
   "được kubelet khởi động **trước khi API server sẵn sàng**", nên nó là thứ duy nhất bootstrap
   được các thành phần control plane. Đó cũng là công dụng chính mà bài nêu ngay ở đầu: dùng
   kubelet để giám sát từng thành phần control plane riêng lẻ.
4. **DaemonSet.** Bài khuyến nghị thẳng: nếu bạn đang vận hành Kubernetes dạng cluster và dùng
   static Pod để chạy một Pod trên mọi node, thì nên dùng DaemonSet thay thế. Lý do là static
   Pod **không được control plane quản lý** nên "không thể được triển khai cuốn chiếu, hoàn tác
   hay co giãn bằng các cơ chế tiêu chuẩn của Kubernetes" — bạn sẽ phải sửa file thủ công trên
   từng node. DaemonSet cung cấp đúng những khả năng đó và là cách tiếp cận được khuyến nghị
   cho workload ở cấp node.
5. **Không.** Mục *Giới hạn* nói dứt khoát: spec của một static Pod **không thể tham chiếu đến
   các đối tượng API khác**, chẳng hạn ServiceAccount, ConfigMap hay Secret. Điều này khớp
   chính xác với câu ở bài [109](109-secret-vi.md): "Bạn không thể dùng ConfigMap hay Secret
   với static Pod." Lý do là nhất quán với toàn bộ bài: static Pod sống được cả khi chưa có API
   server, nên nó không thể phụ thuộc vào thứ chỉ API server mới cấp được.

</details>

Nhóm 3b kết thúc ở đây. Câu nào chưa trả lời được thì quay lại đúng mục tương ứng, rồi chuyển
sang [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) để kiểm chứng cả bảy
bài trên cluster thật.

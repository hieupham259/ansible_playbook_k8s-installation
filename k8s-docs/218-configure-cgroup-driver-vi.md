# Cấu hình cgroup driver (Configuring a cgroup driver)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 20 — Cấu hình lại cluster đang chạy](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), bài 2/6 ·
Giai đoạn 20 **không có lab riêng**: bạn tự chấm bằng **Checkpoint** ghi ở cuối mục giai đoạn đó
trong lộ trình, chạy trên cluster ba VM dựng ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

**Đọc bài này với đúng ranh giới của nó.** *Chọn* cgroup driver nào thì đã xong từ lâu: bài
[00](00-container-runtimes-vi.md) ở giai đoạn 2 giải thích vì sao là `systemd`,
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) đặt containerd `SystemdCgroup=true` khi dựng cluster, và
[Lab 2](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md) phần B2.2 đã chứng minh kubelet và containerd
đang khớp nhau. Cái bài này thêm vào — và chỉ có ở đây — là **quy trình đổi driver trên một cluster
đang chạy**, đúng phần mà Lab 2 phần B2.3 cố ý chỉ đọc chứ không làm và ghi lại là "để đối chiếu ở
giai đoạn 20".

**Phải hiểu ở lần đọc này:**

- Mục *Chuyển sang driver `systemd`* nói quy trình đổi tại chỗ **tương tự như khi nâng cấp kubelet**
  và **phải gồm cả hai bước** bên dưới nó — làm một bước là hỏng. Bài cũng nêu một lựa chọn thay
  thế: **thay node cũ bằng node mới**, khi đó chỉ cần bước đầu.
- Hai bước đó nằm ở hai tầng khác nhau. Bước một sửa **ConfigMap `kubelet-config` trong namespace
  `kube-system`** (`kubectl edit cm kubelet-config -n kube-system`), thêm hoặc sửa `cgroupDriver`
  **dưới mục `kubelet:`** — đây là cấu hình dùng chung cho **mọi** node. Bước hai đi từng node đổi
  cấu hình cục bộ.
- Vòng lặp trên từng node, đúng thứ tự: drain với `--ignore-daemonsets` → `systemctl stop kubelet`
  → dừng container runtime → đổi cgroup driver của runtime → đặt `cgroupDriver: systemd` trong
  `/var/lib/kubelet/config.yaml` → khởi động runtime → `systemctl start kubelet` → uncordon. Bài
  bắt làm **từng node một** để workload có đủ thời gian được lập lịch sang node khác.
- Nguồn của giá trị mặc định: **từ v1.22, không đặt `cgroupDriver` thì kubeadm mặc định `systemd`**.
  Vì vậy muốn giữ `cgroupfs` bạn phải **khai báo tường minh** — nếu không, các phiên bản kubeadm
  sau này sẽ áp `systemd` vào khi `kubeadm upgrade`.
- kubeadm dùng **một** `KubeletConfiguration` cho **tất cả** node, lưu trong ConfigMap ở
  `kube-system`; các lệnh con `init`, `join`, `upgrade` ghi nó ra `/var/lib/kubelet/config.yaml`
  của node. Phần **duy nhất** mang tính từng-node là `containerRuntimeEndpoint`: kubeadm phát hiện
  CRI socket, lưu vào `/var/lib/kubelet/instance-config.yaml` rồi vá giá trị đó vào `config.yaml`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Cấu hình cgroup driver của container runtime* | đã làm và đã kiểm chứng rồi: containerd của cluster lab đặt `SystemdCgroup=true` ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md), đối chiếu ở [Lab 2](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md) phần B2.2 | bài [00](00-container-runtimes-vi.md), đã đọc ở [giai đoạn 2](00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime) |
| Ví dụ `kubeadm-config.yaml` truyền cho `kubeadm init` | chỉ dùng khi **dựng** cluster mới; cluster lab đã dựng xong | [giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) — [Lab 8a](labs/LAB-8A-DUNG-CLUSTER-BANG-KUBEADM.md) |
| Ghi chú về tự động phát hiện cgroup driver, alpha từ v1.28 | là một feature gate alpha, không thuộc baseline của cluster lab | bài [196](196-configure-feature-gates-vi.md), cùng giai đoạn 20 |
| Thực hiện thật việc đổi driver trên cluster lab | [Lab 2](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md) phần B2.3 đã kết luận đổi driver trên node đang chạy có rủi ro lỗi tạo lại Pod sandbox, và cluster lab đang dùng đúng driver rồi | Checkpoint giai đoạn 20 **không** yêu cầu đổi cgroup driver; nó yêu cầu đổi một tham số kubelet khác — bài [224](224-kubelet-config-file-vi.md) |

---

Trang này giải thích cách cấu hình cgroup driver của kubelet sao cho khớp với cgroup driver của
container runtime trong các cluster dựng bằng kubeadm.

## Trước khi bạn bắt đầu (Before you begin)

Bạn nên nắm được các
[yêu cầu về container runtime](00-container-runtimes-vi.md)
của Kubernetes.

## Cấu hình cgroup driver của container runtime (Configuring the container runtime cgroup driver)

Trang [Container runtimes](00-container-runtimes-vi.md)
giải thích rằng driver `systemd` được khuyến nghị cho các hệ thống dựng bằng kubeadm, thay vì
driver `cgroupfs` [mặc định](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1)
của kubelet, bởi vì kubeadm quản lý kubelet như một
[systemd service](04-kubelet-integration-vi.md).

Trang đó cũng cung cấp chi tiết về cách thiết lập một số container runtime khác nhau với driver
`systemd` theo mặc định.

## Cấu hình cgroup driver của kubelet (Configuring the kubelet cgroup driver) {#configuring-the-kubelet-cgroup-driver}

kubeadm cho phép bạn truyền một cấu trúc `KubeletConfiguration` trong lúc chạy `kubeadm init`.
`KubeletConfiguration` này có thể bao gồm trường `cgroupDriver`, trường này điều khiển cgroup
driver của kubelet.

> **Ghi chú:**
>
> Từ v1.22 trở đi, nếu người dùng không đặt trường `cgroupDriver` trong `KubeletConfiguration`,
> kubeadm sẽ mặc định nó là `systemd`.
>
> Trong Kubernetes v1.28, bạn có thể bật tính năng tự động phát hiện cgroup driver dưới dạng
> tính năng alpha. Xem
> [systemd cgroup driver](00-container-runtimes-vi.md#systemd-cgroup-driver)
> để biết thêm chi tiết.

Một ví dụ tối giản về việc cấu hình trường này một cách tường minh:

```yaml
# kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta4
kubernetesVersion: v1.21.0
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```

File cấu hình như vậy sau đó có thể được truyền cho lệnh kubeadm:

```shell
kubeadm init --config kubeadm-config.yaml
```

> **Ghi chú:**
>
> Kubeadm dùng cùng một `KubeletConfiguration` cho tất cả các node trong cluster.
> `KubeletConfiguration` được lưu trong một object
> [ConfigMap](108-configmap-vi.md)
> thuộc namespace `kube-system`.
>
> Việc thực thi các lệnh con `init`, `join` và `upgrade` sẽ khiến kubeadm ghi
> `KubeletConfiguration` ra một file tại `/var/lib/kubelet/config.yaml`
> và truyền nó cho kubelet của node cục bộ.
>
> Trên mỗi node, kubeadm phát hiện CRI socket và lưu thông tin chi tiết của nó vào file
> `/var/lib/kubelet/instance-config.yaml`. Khi thực thi các lệnh con `init`, `join` hoặc
> `upgrade`, kubeadm sẽ vá (patch) giá trị `containerRuntimeEndpoint` từ file cấu hình instance
> này vào `/var/lib/kubelet/config.yaml`.

## Sử dụng driver `cgroupfs` (Using the `cgroupfs` driver)

Để dùng `cgroupfs` và để ngăn `kubeadm upgrade` sửa đổi cgroup driver trong
`KubeletConfiguration` trên các hệ thống hiện có, bạn phải khai báo tường minh giá trị của nó.
Điều này áp dụng cho trường hợp bạn không muốn các phiên bản kubeadm trong tương lai áp dụng
driver `systemd` theo mặc định.

Xem mục "[Sửa ConfigMap của kubelet](#modify-the-kubelet-configmap)" bên dưới để biết chi tiết
về cách khai báo tường minh giá trị này.

Nếu bạn muốn cấu hình container runtime dùng driver `cgroupfs`, bạn phải tham khảo tài liệu của
container runtime mà bạn chọn.

## Chuyển sang driver `systemd` (Migrating to the `systemd` driver)

Để thay đổi cgroup driver của một cluster kubeadm hiện có từ `cgroupfs` sang `systemd` tại chỗ
(in-place), cần một quy trình tương tự như khi nâng cấp kubelet. Quy trình này phải bao gồm cả
hai bước được nêu dưới đây.

> **Ghi chú:**
>
> Một cách khác là thay các node cũ trong cluster bằng các node mới sử dụng driver `systemd`.
> Cách này chỉ yêu cầu thực hiện bước đầu tiên bên dưới trước khi join các node mới, đồng thời
> đảm bảo các workload có thể di chuyển an toàn sang các node mới trước khi xóa các node cũ.

### Sửa ConfigMap của kubelet (Modify the kubelet ConfigMap) {#modify-the-kubelet-configmap}

- Chạy `kubectl edit cm kubelet-config -n kube-system`.
- Sửa giá trị `cgroupDriver` hiện có hoặc thêm một trường mới trông như sau:

  ```yaml
  cgroupDriver: systemd
  ```

  Trường này phải nằm dưới mục `kubelet:` của ConfigMap.

### Cập nhật cgroup driver trên tất cả các node (Update the cgroup driver on all nodes)

Với từng node trong cluster:

- [Drain node](255-safely-drain-node-vi.md) bằng lệnh
  `kubectl drain <node-name> --ignore-daemonsets`
- Dừng kubelet bằng lệnh `systemctl stop kubelet`
- Dừng container runtime
- Sửa cgroup driver của container runtime thành `systemd`
- Đặt `cgroupDriver: systemd` trong `/var/lib/kubelet/config.yaml`
- Khởi động container runtime
- Khởi động kubelet bằng lệnh `systemctl start kubelet`
- [Uncordon node](255-safely-drain-node-vi.md) bằng
  lệnh `kubectl uncordon <node-name>`

Hãy thực hiện các bước này trên từng node một, để đảm bảo các workload có đủ thời gian được lập
lịch (schedule) sang các node khác.

Khi quy trình hoàn tất, hãy đảm bảo tất cả các node và workload đều khỏe mạnh.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 20:

1. Cluster lab của bạn đã chạy `cgroupDriver: systemd` từ Lab 00, và Lab 2 đã chứng minh kubelet
   khớp containerd. Vậy bài này còn dạy thêm điều gì mà hai lab đó chưa dạy?
2. **Câu bẫy.** Bạn chạy `kubectl edit cm kubelet-config -n kube-system`, đổi `cgroupDriver` thành
   `systemd` rồi lưu lại. Ba node đã đổi driver chưa? Vì sao?
3. Bạn phải đổi driver trên `lab-k8s-worker2`. Kể đúng thứ tự các bước từ lúc bắt đầu tới lúc node
   nhận Pod trở lại, và nói vì sao bài bắt làm từng node một thay vì đổi cả ba cùng lúc.
4. Bạn tiếp quản một cluster đang chạy `cgroupfs` và muốn **giữ nguyên** như vậy. Bài bảo phải làm
   gì, và vì sao "không làm gì cả" lại là câu trả lời sai?
5. kubeadm dùng một `KubeletConfiguration` chung cho cả cluster. Vậy phần nào trong cấu hình kubelet
   của một node **không** đến từ ConfigMap chung đó, và kubeadm lấy nó ở đâu?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Dạy **quy trình đổi cgroup driver trên một cluster đang chạy**. Lab 00 chốt giá trị lúc dựng,
   Lab 2 chỉ **đọc và đối chiếu** hai bên có khớp không; còn cách chuyển từ `cgroupfs` sang
   `systemd` tại chỗ — sửa ở đâu, theo thứ tự nào, kèm điều kiện gì — thì chỉ có ở bài này. Nó cũng
   trả lời câu hỏi ngược: muốn **giữ** `cgroupfs` thì phải làm gì.
2. **Chưa.** ConfigMap `kubelet-config` là **nguồn cấu hình chung**, còn kubelet trên mỗi node đọc
   file cục bộ `/var/lib/kubelet/config.yaml`. Bài xếp việc sửa ConfigMap là **bước một** và nói rõ
   quy trình **phải gồm cả hai bước**; bước hai là đi từng node đặt `cgroupDriver: systemd` trong
   `/var/lib/kubelet/config.yaml` cùng với việc đổi driver của container runtime. Sửa ConfigMap rồi
   dừng lại là chỗ dễ tưởng đã xong nhất.
3. Thứ tự: **drain node** bằng `kubectl drain <node-name> --ignore-daemonsets` → `systemctl stop
   kubelet` → **dừng container runtime** → sửa cgroup driver của container runtime thành `systemd`
   → đặt `cgroupDriver: systemd` trong `/var/lib/kubelet/config.yaml` → **khởi động container
   runtime** → `systemctl start kubelet` → `kubectl uncordon <node-name>`. Bài bắt làm **từng node
   một** để **các workload có đủ thời gian được lập lịch sang những node khác**; đổi cả ba cùng lúc
   thì không còn node nào nhận Pod bị evict.
4. Phải **khai báo tường minh** giá trị `cgroupDriver: cgroupfs` trong `KubeletConfiguration` — cụ
   thể là qua mục *Sửa ConfigMap của kubelet*. "Không làm gì cả" sai vì **từ v1.22, nếu không đặt
   trường `cgroupDriver` thì kubeadm mặc định nó là `systemd`**; để trống nghĩa là chấp nhận cho
   `kubeadm upgrade` áp `systemd` vào cluster của bạn ở lần nâng cấp sau.
5. Phần **`containerRuntimeEndpoint`**. kubeadm **phát hiện CRI socket trên từng node** và lưu chi
   tiết vào **`/var/lib/kubelet/instance-config.yaml`**; khi chạy `init`, `join` hoặc `upgrade`, nó
   **vá** giá trị đó vào `/var/lib/kubelet/config.yaml` của node. Mọi trường còn lại đến từ
   `KubeletConfiguration` chung lưu trong ConfigMap ở namespace `kube-system`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau. Cả giai đoạn 20
chấm bằng **Checkpoint** ở cuối mục
[Giai đoạn 20 — Cấu hình lại cluster đang chạy](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy),
làm trên cluster lab chứ không có lab riêng.

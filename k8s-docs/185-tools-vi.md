# Cài đặt công cụ (Install Tools)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/tools/>
>
> Thiết lập các công cụ Kubernetes trên máy tính của bạn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 1 — Mô hình Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-1--mô-hình-kubernetes)
→ nhóm [1b. Làm việc với object và kubectl](00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl),
bài 1/8 của dòng **Thực hành** · Kiểm chứng ở
[Lab 1b](labs/LAB-1B-OBJECT-LABEL-KUBECTL-VA-KUBECONFIG.md) phần B0, nơi bạn xác nhận `kubectl`
và độ lệch phiên bản client/server trên cluster lab.

Đây là **trang mục** của nhóm `tasks/tools/`, không phải bài học. Nó chỉ giới thiệu bốn công cụ
và trỏ sang trang cài của từng cái. Đọc một lượt để biết công cụ nào dùng vào việc gì, rồi đi
tiếp — đừng cài gì theo trang này.

**Phải hiểu ở lần đọc này:**

- Bốn công cụ trên trang chia làm hai loại khác hẳn nhau: `kubectl` là **client điều khiển**
  một cluster đã có, còn `kind`, `minikube` và `kubeadm` là công cụ **dựng cluster**.
- `kubectl` là công cụ duy nhất bạn cần ngay ở giai đoạn 1. Mục *kubectl* trỏ sang ba trang cài
  theo hệ điều hành — chỉ đọc trang khớp máy bạn, [186](186-install-kubectl-linux-vi.md) cho
  Linux.
- Lộ trình này dựng cluster bằng **kubeadm** trên VM thật, không dùng `kind` hay `minikube`.
  Hai mục đó đọc để biết chúng tồn tại, không phải để làm theo.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *kind* và *minikube* | cluster của lộ trình là ba VM thật; hai công cụ này chạy cluster trong container hoặc VM đơn, cho ra môi trường khác hẳn nên [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) cấm dùng | không dùng trong lộ trình |
| *kubeadm* | mới là lời giới thiệu một dòng; quy trình cài và dùng nằm ở hai bài riêng | [Giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm), bài [01](01-install-kubeadm-vi.md) và [02](02-create-cluster-kubeadm-vi.md) |

---

> **Ghi chú:** Xem trang [Môi trường học tập (Learning environment)](https://kubernetes.io/docs/setup/learning-environment/)
> để thiết lập một môi trường thực hành.

## kubectl

Công cụ dòng lệnh của Kubernetes, [kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/),
cho phép bạn chạy các lệnh đối với các cluster Kubernetes.
Bạn có thể dùng kubectl để triển khai ứng dụng, xem xét (inspect) và quản lý tài nguyên của cluster,
và xem log. Để biết thêm thông tin, bao gồm danh sách đầy đủ các thao tác của kubectl, xem
[tài liệu tham khảo `kubectl`](https://kubernetes.io/docs/reference/kubectl/).

kubectl có thể cài đặt được trên nhiều nền tảng Linux khác nhau, macOS và Windows.
Tìm hệ điều hành mà bạn dùng bên dưới.

- [Cài đặt kubectl trên Linux](186-install-kubectl-linux-vi.md)
- [Cài đặt kubectl trên macOS](187-install-kubectl-macos-vi.md)
- [Cài đặt kubectl trên Windows](188-install-kubectl-windows-vi.md)

## kind

[`kind`](https://kind.sigs.k8s.io/) cho phép bạn chạy Kubernetes trên máy tính
cục bộ của mình. Công cụ này yêu cầu bạn đã cài đặt
[Docker](https://www.docker.com/) hoặc [Podman](https://podman.io/).

Trang [Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/) của kind
chỉ cho bạn những gì cần làm để bắt đầu chạy được với kind.

[Xem hướng dẫn Quick Start của kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

## minikube

Giống như `kind`, [`minikube`](https://minikube.sigs.k8s.io/) là một công cụ cho phép bạn chạy Kubernetes
cục bộ. `minikube` chạy một cluster Kubernetes cục bộ tất-cả-trong-một (all-in-one) hoặc nhiều node
trên máy tính cá nhân của bạn (bao gồm PC chạy Windows, macOS và Linux) để bạn có thể dùng thử
Kubernetes, hoặc dùng cho công việc phát triển hằng ngày.

Bạn có thể làm theo hướng dẫn chính thức
[Get Started!](https://minikube.sigs.k8s.io/docs/start/) nếu mối quan tâm của bạn
là cài đặt được công cụ này.

[Xem hướng dẫn Get Started! của minikube](https://minikube.sigs.k8s.io/docs/start/)

Sau khi `minikube` đã hoạt động, bạn có thể dùng nó để
[chạy một ứng dụng mẫu](https://kubernetes.io/docs/tutorials/hello-minikube/).

## kubeadm

Bạn có thể dùng công cụ kubeadm để tạo và quản lý các cluster Kubernetes.
Nó thực hiện các hành động cần thiết để dựng lên một cluster tối thiểu khả dụng (minimum viable) và an toàn,
theo cách thân thiện với người dùng.

[Cài đặt kubeadm](01-install-kubeadm-vi.md) chỉ cho bạn cách cài đặt kubeadm.
Sau khi cài đặt, bạn có thể dùng nó để [tạo một cluster](02-create-cluster-kubeadm-vi.md).

[Xem hướng dẫn cài đặt kubeadm](01-install-kubeadm-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 1:

1. Trang này liệt kê bốn công cụ. Chia chúng thành hai nhóm theo **việc chúng làm**, và nói rõ
   nhóm nào bạn cần trước.
2. **Câu bẫy.** Trang gọi cả `kind`, `minikube` và `kubeadm` là công cụ "chạy Kubernetes".
   Vậy ba công cụ đó thay thế được cho nhau không, và cụ thể cái nào dựng nên cluster ba VM
   `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2` mà bạn đang học trên đó?
3. Trong bốn mục của trang, mục nào là thứ **duy nhất** bạn phải làm ngay ở giai đoạn 1, và
   trang con nào của mục đó áp dụng cho máy bạn?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Một nhóm điều khiển, ba công cụ dựng.** `kubectl` là **client**: nó nói chuyện với một
   cluster **đã tồn tại** để triển khai ứng dụng, xem xét tài nguyên và đọc log. `kind`,
   `minikube`, `kubeadm` đều **tạo ra cluster**. **Bạn cần `kubectl` trước** — không có nó thì
   dù cluster đã chạy bạn cũng không ra lệnh được.
2. **Không thay thế được cho nhau, vì chúng tạo ra ba loại môi trường khác nhau.** `kind` chạy
   cluster **bên trong container**, nên đòi Docker hoặc Podman. `minikube` chạy một cluster
   **cục bộ trên chính máy cá nhân**, all-in-one hoặc nhiều node. `kubeadm` dựng **cluster thật
   trên các máy bạn chỉ định** — đó là cái duy nhất trong ba cái tạo ra được một cluster nhiều
   máy thật. Cluster ba VM của bạn **do `kubeadm` dựng**; chỗ dễ nhầm là cả ba đều được trang
   mô tả bằng cụm "chạy Kubernetes cục bộ", nhưng "cục bộ" của `kind`/`minikube` nghĩa là gói
   gọn trong một máy, còn `kubeadm` không có giới hạn đó.
3. **Mục *kubectl*.** Nó trỏ sang ba trang cài theo hệ điều hành; với cluster lab, mọi lệnh chạy
   trên VM Ubuntu nên trang áp dụng là [186 — Cài đặt kubectl trên Linux](186-install-kubectl-linux-vi.md).
   Hai trang [187](187-install-kubectl-macos-vi.md) và [188](188-install-kubectl-windows-vi.md)
   chỉ cần khi bạn muốn điều khiển cluster từ máy trạm macOS hoặc Windows.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

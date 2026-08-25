# Sử dụng Calico cho NetworkPolicy (Use Calico for NetworkPolicy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/calico-network-policy/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy)
— **trang con của bài 7/14**, mục [Cài đặt một Network Policy
Provider](243-network-policy-provider-vi.md); nó không có dòng riêng trong lộ trình. Phần II không
có lab: thực hành thẳng trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm
bằng **Checkpoint** ở cuối [mục giai đoạn
21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy).

Trong năm trang provider, **đây là trang duy nhất khớp cluster lab**: bạn đã chạy đúng nhánh của
nó ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) phần B4 — lab đổi CNI sang Calico và
trả nợ *NetworkPolicy được thực thi thật*. Đọc lại ở đây là để **đối chiếu việc đã làm với tài
liệu**, không phải để cài lại. Bốn trang provider còn lại chỉ đọc cho biết, không được cài vào
cluster lab vì sẽ phá [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot).

**Phải hiểu ở lần đọc này:**

- Trang chia đôi ngay ở mục *Trước khi bạn bắt đầu*: **cloud (GKE)** hay **cục bộ (kubeadm)** —
  phải chọn nhánh trước khi đọc tiếp. Đây là đặc trưng riêng của trang Calico trong nhóm: nó là
  trang provider duy nhất tách hai môi trường triển khai thành hai mục ngang hàng.
- Nhánh GKE: Calico được bật bằng **một cờ lúc tạo cluster** — `gcloud container clusters create
  [CLUSTER_NAME] --enable-network-policy` — chứ không phải một bước cài đặt riêng sau đó.
- Nhánh *Tạo một cluster Calico cục bộ với kubeadm* là nhánh của cluster lab, và bài chỉ trỏ sang
  *Calico Quickstart* chứ không giữ lệnh. Cách xác minh thì bài giữ lại, và nó dùng được cho cả
  hai nhánh: `kubectl get pods --namespace=kube-system`, các Pod của Calico **có tên bắt đầu bằng
  `calico`**, và mọi Pod phải ở trạng thái `Running`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Toàn bộ nhánh *Tạo một cluster Calico với Google Kubernetes Engine (GKE)*, điều kiện tiên quyết `gcloud` | cluster lab là ba VM tự dựng, **không có nhà cung cấp cloud** nào để chạy `gcloud` | không có trong lộ trình; nhánh áp dụng được là nhánh kubeadm ngay dưới nó, đã chạy ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) phần B4 |
| Chi tiết bên trong *Calico Quickstart* mà nhánh kubeadm trỏ tới | quy trình cài đã chạy rồi, theo đúng phiên bản Calico được khóa cho lộ trình | [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) phần B4; số phiên bản nằm ở [bảng A1.4 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) |

---

Trang này trình bày một vài cách nhanh để tạo một cluster Calico trên Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Hãy quyết định xem bạn muốn triển khai một cluster
[trên cloud](#creating-a-calico-cluster-with-google-kubernetes-engine-gke) hay
[cục bộ](#creating-a-local-calico-cluster-with-kubeadm).

## Tạo một cluster Calico với Google Kubernetes Engine (GKE) (Creating a Calico cluster with Google Kubernetes Engine (GKE)) {#creating-a-calico-cluster-with-google-kubernetes-engine-gke}

**Điều kiện tiên quyết**: [gcloud](https://cloud.google.com/sdk/docs/quickstarts).

1.  Để khởi tạo một cluster GKE với Calico, hãy thêm cờ (flag) `--enable-network-policy`.

    **Cú pháp**
    ```shell
    gcloud container clusters create [CLUSTER_NAME] --enable-network-policy
    ```

    **Ví dụ**
    ```shell
    gcloud container clusters create my-calico-cluster --enable-network-policy
    ```

1.  Để xác minh việc triển khai, hãy dùng lệnh sau.

    ```shell
    kubectl get pods --namespace=kube-system
    ```

    Các pod của Calico bắt đầu bằng `calico`. Hãy kiểm tra để chắc chắn rằng
    mỗi pod đều có trạng thái `Running`.

## Tạo một cluster Calico cục bộ với kubeadm (Creating a local Calico cluster with kubeadm) {#creating-a-local-calico-cluster-with-kubeadm}

Để có một cluster Calico cục bộ trên một máy (single-host) trong mười lăm phút
bằng kubeadm, hãy tham khảo
[Calico Quickstart](https://projectcalico.docs.tigera.io/getting-started/kubernetes/).

## Tiếp theo (What's next)

Khi cluster của bạn đã chạy, bạn có thể làm theo bài
[Khai báo Network Policy](201-declare-network-policy-vi.md) để thử nghiệm
NetworkPolicy của Kubernetes.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Bài bắt bạn quyết định một điều **trước** khi đọc bất cứ lệnh nào. Điều đó là gì, và hai nhánh
   sau đó khác nhau ở chỗ nào về mặt thao tác?
2. **Câu bẫy.** Trên GKE, Calico được bật bằng cờ `--enable-network-policy` của
   `gcloud container clusters create`. Vậy có thể hiểu rằng "cài Calico" luôn là chạy thêm một
   bước sau khi cluster đã có không? Nhánh GKE nói ngược lại điều gì?
3. Trên `lab-k8s-master`, bạn muốn chứng minh Calico đã lên đủ trước khi viết NetworkPolicy cho
   Checkpoint giai đoạn 21. Bài đưa đúng một cách xác minh — lệnh nào, và bạn nhìn vào **hai** dấu
   hiệu nào trong output?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Phải quyết định **triển khai trên cloud hay cục bộ** — mục *Trước khi bạn bắt đầu* chỉ có đúng
   một việc là chọn giữa hai nhánh đó. Khác nhau về thao tác: nhánh **GKE** không cài gì cả, chỉ
   **thêm một cờ khi tạo cluster** rồi kiểm tra Pod; nhánh **cục bộ với kubeadm** thì cluster đã có
   sẵn và bạn chạy quy trình cài Calico theo *Calico Quickstart*.
2. **Không.** Nhánh GKE cho thấy trên nền tảng có quản lý, việc bật Calico là **một tùy chọn ngay
   lúc tạo cluster** — `gcloud container clusters create [CLUSTER_NAME] --enable-network-policy` —
   chứ không phải một bước cài đặt riêng đến sau. Chỗ dễ nhầm là suy từ nhánh kubeadm ra cả trang:
   "đã có cluster rồi mới thêm CNI" đúng cho nhánh cục bộ, **không đúng** cho nhánh GKE.
3. Lệnh **`kubectl get pods --namespace=kube-system`**. Hai dấu hiệu: các Pod của Calico **có tên
   bắt đầu bằng `calico`**, và **mỗi Pod đó phải ở trạng thái `Running`**. Bài không đưa cách xác
   minh nào khác — đây là cách dùng chung cho cả nhánh GKE lẫn nhánh kubeadm.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

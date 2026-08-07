# Cấu hình (Configuration)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/configuration/>
>
> Các tài nguyên mà Kubernetes cung cấp để cấu hình các Pod.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3b](LO-TRINH-ADMIN.md#3b-cấu-hình-và-tài-nguyên), bài 1/7 ·
Kiểm chứng ở Lab 3b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là **trang mục lục**, không phải bài học. Cả trang chỉ có một câu giới thiệu và một danh
sách năm trang con. Đọc mất chưa tới một phút; việc duy nhất cần làm là ghi nhớ phạm vi của
nhóm bài sắp tới.

**Phải hiểu ở lần đọc này:**

- Phạm vi của cả mục, theo đúng câu mở đầu trang: đây là **các tài nguyên dùng để cấu hình
  Pod**, không phải cấu hình của cluster hay của các thành phần control plane.
- Ba mục đầu trong danh sách — ConfigMap, Secret, *Quản lý tài nguyên cho Pod và Container* —
  chính là ba bài kế tiếp của nhóm 3b, và bạn sẽ đọc đúng theo thứ tự đó.
- *Tổ chức truy cập cluster bằng các file kubeconfig* có trong danh sách nhưng đã học rồi;
  đừng đọc lại ở đây.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Tổ chức truy cập cluster bằng các file kubeconfig* | đã học, không thuộc nhóm 3b | giai đoạn 1, nhóm [1b](LO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl) |
| *Quản lý tài nguyên cho các node Windows* | cluster lab không có node Windows | giai đoạn 15 |

---

Đây là trang mục lục của phần khái niệm về Cấu hình (Configuration) trong tài liệu Kubernetes.
Các tài nguyên mà Kubernetes cung cấp để cấu hình các Pod.

---

Các trang trong mục này:

* [ConfigMap (ConfigMaps)](./108-configmap-vi.md)
* [Secret (Secrets)](https://kubernetes.io/docs/concepts/configuration/secret/)
* [Quản lý tài nguyên cho Pod và Container (Resource Management for Pods and Containers)](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
* [Tổ chức truy cập cluster bằng các file kubeconfig (Organizing Cluster Access Using kubeconfig Files)](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
* [Quản lý tài nguyên cho các node Windows (Resource Management for Windows nodes)](https://kubernetes.io/docs/concepts/configuration/windows-resource-management/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Trong năm trang con được liệt kê, ba trang nào là ba bài kế tiếp bạn phải đọc trong nhóm
   3b? Hai trang còn lại vì sao không nằm trong danh sách phải đọc lúc này?
2. Cluster lab của bạn có một control plane `k8s-master` và hai worker chạy Ubuntu 24.04.
   Trang con nào trong mục này chắc chắn không dùng tới, và trang con nào bạn đã dùng từ
   giai đoạn 1?
3. Mục này tên là "Cấu hình". Vậy nó có bao gồm cách cấu hình `kube-apiserver` và `kubelet`
   trên cluster của bạn không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **ConfigMap, Secret, và *Quản lý tài nguyên cho Pod và Container*** — theo đúng thứ tự
   xuất hiện trong danh sách. Hai trang còn lại: *Tổ chức truy cập cluster bằng các file
   kubeconfig* **đã học ở giai đoạn 1**, và *Quản lý tài nguyên cho các node Windows*
   thuộc **giai đoạn 15**.
2. **Không dùng tới:** *Quản lý tài nguyên cho các node Windows*, vì cả ba VM đều là Ubuntu.
   **Đã dùng từ giai đoạn 1:** *Tổ chức truy cập cluster bằng các file kubeconfig* — đó chính
   là file kubeconfig bạn dùng mỗi lần gõ `kubectl` trên `k8s-master`.
3. **Không.** Câu mở đầu trang giới hạn phạm vi rất rõ: đây là "các tài nguyên mà Kubernetes
   cung cấp để **cấu hình các Pod**". Toàn bộ năm trang con đều là tài nguyên API hoặc trường
   trong `spec` của Pod. Cấu hình của chính các thành phần control plane và kubelet là chuyện
   của giai đoạn 8, không phải của mục này.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

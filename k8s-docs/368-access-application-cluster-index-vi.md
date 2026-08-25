# Truy cập ứng dụng trong một cluster (Access Applications in a Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/access-application-cluster/>
>
> Cấu hình cân bằng tải (load balancing), port forwarding, hoặc thiết lập firewall và cấu hình
> DNS để truy cập các ứng dụng trong một cluster.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 30 — Truy cập ứng dụng trong cluster](00-ALO-TRINH-ADMIN.md#giai-đoạn-30--truy-cập-ứng-dụng-trong-cluster),
bài 1/4 · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 30](00-ALO-TRINH-ADMIN.md#giai-đoạn-30--truy-cập-ứng-dụng-trong-cluster).

Đây là **trang mục lục** của nhóm `tasks/access-application-cluster/`, trang gốc không có nội dung
kỹ thuật riêng. Đọc trong vài phút để lấy bản đồ. Lộ trình ghi rõ giai đoạn 30 là **nhóm thực hành
đi kèm [giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng)**: phần lớn
trang con của mục này bạn đã đọc rải rác ở các giai đoạn trước, giai đoạn 30 chỉ lấy nốt ba trang
còn lại.

**Phải hiểu ở lần đọc này:**

- Mục *Danh sách các trang trong mục này* có 11 trang con, nhưng giai đoạn 30 chỉ đọc thêm ba:
  [371](371-web-ui-dashboard-vi.md) (bài 2/4), [370](370-service-access-application-cluster-vi.md)
  (bài 3/4) và [369](369-access-cluster-services-vi.md) (bài 4/4). Tám trang còn lại đã nằm ở dòng
  **Thực hành** của các giai đoạn trước — nhận ra trang nào mình đã đọc rồi là đủ.
- Đoạn mở đầu liệt kê các họ cách tiếp cận một ứng dụng đang chạy trong cluster: truy cập chính
  Kubernetes API và cấu hình `kubeconfig` cho nhiều cluster, port forwarding, dùng Service, kết nối
  frontend với backend, cấp phát một load balancer bên ngoài, và cấu hình DNS ở mức cluster. Ba
  đường mà Checkpoint giai đoạn 30 bắt phân biệt — `port-forward`, apiserver proxy, Service — đều
  nằm trong danh sách này.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nội dung kỹ thuật của từng trang con | trang này chỉ là mục lục | ba trang [371](371-web-ui-dashboard-vi.md), [370](370-service-access-application-cluster-vi.md), [369](369-access-cluster-services-vi.md) ngay sau đây |
| Tám trang con còn lại trong danh sách | đã đọc ở giai đoạn trước, giai đoạn 30 không đọc lại | [361](361-configure-access-multiple-clusters-vi.md) và [365](365-list-running-container-images-vi.md) ở [giai đoạn 1b](00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl); [360](360-containers-shared-volume-vi.md) ở [giai đoạn 3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời); [362](362-configure-dns-cluster-vi.md), [363](363-connecting-frontend-backend-vi.md), [364](364-create-external-load-balancer-vi.md), [366](366-port-forward-vi.md) ở [giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng); [359](359-access-cluster-vi.md) ở [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) |

---

Trang gốc là trang mục lục của phần *Tasks → Access Applications in a Cluster*: nội dung trang
chỉ gồm danh sách các trang con, được liệt kê dưới đây theo đúng thứ tự hiển thị trên trang web.
Các trang này hướng dẫn những cách khác nhau để tiếp cận một ứng dụng đang chạy trong cluster —
từ việc truy cập chính Kubernetes API và cấu hình `kubeconfig` cho nhiều cluster, cho tới port
forwarding, dùng Service, kết nối frontend với backend, cấp phát (provision) một bộ cân bằng tải
(load balancer) bên ngoài, cũng như cấu hình DNS ở mức cluster.

## Danh sách các trang trong mục này (Pages in this section)

- [Triển khai và truy cập Kubernetes Dashboard (Deploy and Access the Kubernetes Dashboard)](371-web-ui-dashboard-vi.md) — *Triển khai giao diện web (Kubernetes Dashboard) và truy cập nó.*
- [Truy cập các cluster (Accessing Clusters)](359-access-cluster-vi.md)
- [Cấu hình truy cập tới nhiều cluster (Configure Access to Multiple Clusters)](361-configure-access-multiple-clusters-vi.md)
- [Dùng port forwarding để truy cập ứng dụng trong một cluster (Use Port Forwarding to Access Applications in a Cluster)](366-port-forward-vi.md)
- [Dùng Service để truy cập một ứng dụng trong cluster (Use a Service to Access an Application in a Cluster)](370-service-access-application-cluster-vi.md)
- [Kết nối frontend với backend bằng Service (Connect a Frontend to a Backend Using Services)](363-connecting-frontend-backend-vi.md)
- [Tạo một bộ cân bằng tải bên ngoài (Create an External Load Balancer)](364-create-external-load-balancer-vi.md)
- [Liệt kê toàn bộ container image đang chạy trong một cluster (List All Container Images Running in a Cluster)](365-list-running-container-images-vi.md)
- [Giao tiếp giữa các container trong cùng một Pod bằng volume dùng chung (Communicate Between Containers in the Same Pod Using a Shared Volume)](360-containers-shared-volume-vi.md)
- [Cấu hình DNS cho một cluster (Configure DNS for a Cluster)](362-configure-dns-cluster-vi.md)
- [Truy cập các Service đang chạy trên cluster (Access Services Running on Clusters)](369-access-cluster-services-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 30:

1. Mục này liệt kê 11 trang con. Giai đoạn 30 đọc thêm mấy trang trong số đó, là những trang nào,
   và vì sao số còn lại không phải đọc lại?
2. **Câu bẫy.** Mục này nằm ở giai đoạn cuối lộ trình. Vậy có phải tới bây giờ bạn mới bắt đầu học
   cách đưa ứng dụng trong cluster ra ngoài không?
3. Checkpoint giai đoạn 30 bắt bạn expose một Deployment bằng NodePort rồi gọi nó từ máy host, và
   gọi một Pod không có Service. Trong danh sách trang con, trang nào dạy đường thứ nhất, và hai
   đường còn lại đi qua thành phần nào trên `lab-k8s-master` chứ không mở thêm port nào trên
   `lab-k8s-worker1` hay `lab-k8s-worker2`?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Ba trang**: [371](371-web-ui-dashboard-vi.md) — Dashboard,
   [370](370-service-access-application-cluster-vi.md) — dùng Service để truy cập ứng dụng, và
   [369](369-access-cluster-services-vi.md) — truy cập các Service đang chạy trên cluster. Tám
   trang còn lại **đã nằm ở dòng Thực hành của các giai đoạn trước** — giai đoạn 1b, 3a, 5 và 9 —
   nên lộ trình không xếp chúng lại vào giai đoạn 30.
2. **Không.** Lộ trình ghi giai đoạn 30 là **nhóm thực hành đi kèm giai đoạn 5 — Mạng nền tảng**.
   Service, DNS, `port-forward` và load balancer bên ngoài bạn đã đọc và đã kiểm chứng từ giai
   đoạn 5; giai đoạn 30 chỉ gom nốt phần còn thiếu của cùng nhóm `tasks/`. Số thứ tự file lớn
   **không** có nghĩa là kiến thức mới.
3. Đường NodePort là trang [370](370-service-access-application-cluster-vi.md). Hai đường còn lại —
   `port-forward` của trang [366](366-port-forward-vi.md) và **apiserver proxy** của trang
   [369](369-access-cluster-services-vi.md) — đều đi qua **Kubernetes API server** trên
   `lab-k8s-master`, tức qua kubeconfig sẵn có, nên **không** cần mở thêm port nào trên hai worker.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

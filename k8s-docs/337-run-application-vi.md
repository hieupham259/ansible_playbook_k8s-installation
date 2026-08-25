# Chạy ứng dụng (Run Applications)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/run-application/>
>
> Chạy và quản lý cả ứng dụng không trạng thái (stateless) lẫn ứng dụng có trạng thái (stateful).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 4 → nhóm [4a](00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout), bài 4/7 ·
[Lab 4a](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) **không có phần thực hành cho bài này** —
lab ghi rõ lý do ở bảng cuối mục 1.1: đây là trang mục lục, toàn bộ nội dung là danh sách link nên
không có thao tác nào để kiểm chứng.

Đọc trang này trong vài phút, coi như một tấm bản đồ. Nó **không dạy gì mới**.

**Phải hiểu ở lần đọc này:**

- Đây là trang mục của nhánh `/docs/tasks/run-application/`, gom cả ứng dụng không trạng thái lẫn
  ứng dụng có trạng thái vào một chỗ. Giá trị của nó là cho bạn biết nhánh này có những bài nào,
  để sau này biết đường quay lại tra.
- Trong danh sách, **chỉ ba bài thuộc nhóm 4a**: [345](345-run-stateless-application-vi.md),
  [346](346-scale-deployment-vi.md) và [348](348-update-deployment-rolling-vi.md). Ở giai đoạn này
  bạn đọc đúng ba bài đó, theo thứ tự lộ trình chứ không theo thứ tự trên trang.
- Các link còn lại thuộc StatefulSet, HorizontalPodAutoscaler, PodDisruptionBudget và truy cập API
  từ Pod — mỗi nhóm rơi vào một giai đoạn khác nhau, xem bảng dưới. Biết chúng nằm ở đây là đủ,
  đừng mở ra đọc bây giờ.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Link [344](344-run-single-instance-stateful-vi.md) và [343](343-run-replicated-stateful-application-vi.md) | ứng dụng có trạng thái cần volume và StorageClass, chưa học | [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) |
| Link [347](347-scale-stateful-set-vi.md) và [341](341-force-delete-stateful-set-pod-vi.md) | StatefulSet là nhóm bài ngay kế tiếp | nhóm [4b](00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling), thực hành ở [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) |
| Link [340](340-delete-stateful-set-vi.md) | phần xóa StatefulSet gắn với Service headless quản trị | phần Thực hành của [giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) |
| Link [342](342-hpa-walkthrough-vi.md) | HPA đọc lý thuyết ở nhóm [4b](00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling) nhưng cần metrics-server mới thực hành được — đây là **nợ #1** của lộ trình | phần Thực hành của [giai đoạn 11](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability), lab tương ứng là [Lab 11b](labs/LAB-11B-HPA-VA-VPA.md) |
| Link [338](338-access-api-from-pod-vi.md) | truy cập API từ trong Pod cần ServiceAccount và phân quyền | phần Thực hành của [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) |
| Link [339](339-configure-pdb-vi.md) | đã đọc rồi, không phải bài mới | phần Thực hành của nhóm [3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |

---

Mục này bao gồm các trang sau:

* [Chạy một ứng dụng không trạng thái bằng Deployment (Run a Stateless Application Using a Deployment)](345-run-stateless-application-vi.md)
* [Scale thủ công theo chiều ngang cho một Deployment (Horizontal Manual Scaling for a Deployment)](346-scale-deployment-vi.md)
* [Cập nhật một Deployment mà không có downtime (Update a Deployment Without Downtime)](348-update-deployment-rolling-vi.md)
* [Chạy một ứng dụng có trạng thái đơn instance (Run a Single-Instance Stateful Application)](344-run-single-instance-stateful-vi.md)
* [Chạy một ứng dụng có trạng thái được nhân bản (Run a Replicated Stateful Application)](343-run-replicated-stateful-application-vi.md)
* [Scale một StatefulSet (Scale a StatefulSet)](347-scale-stateful-set-vi.md)
* [Xóa một StatefulSet (Delete a StatefulSet)](340-delete-stateful-set-vi.md)
* [Buộc xóa các Pod của StatefulSet (Force Delete StatefulSet Pods)](341-force-delete-stateful-set-pod-vi.md)
* [Hướng dẫn từng bước về HorizontalPodAutoscaler (HorizontalPodAutoscaler Walkthrough)](342-hpa-walkthrough-vi.md)
* [Chỉ định Disruption Budget cho ứng dụng của bạn (Specifying a Disruption Budget for your Application)](339-configure-pdb-vi.md)
* [Truy cập Kubernetes API từ một Pod (Accessing the Kubernetes API from a Pod)](338-access-api-from-pod-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4:

1. Trang này liệt kê 11 bài. Ở giai đoạn 4, ba bài nào trong danh sách đó thuộc nhóm bài bạn đang
   đọc, và bạn dừng lại ở đâu?
2. **Câu bẫy.** Trang gom "stateless" và "stateful" cạnh nhau nên dễ đọc liền một mạch. Vì sao bạn
   **không nên** mở [344](344-run-single-instance-stateful-vi.md) hay
   [343](343-run-replicated-stateful-application-vi.md) ngay bây giờ, dù chúng nằm cùng một trang
   mục với bài bạn vừa đọc?
3. Bạn đang đứng trên `lab-k8s-master` và muốn tự tay nâng số replica của một Deployment lên rồi
   thực hiện rolling update mà không gián đoạn. Trong danh sách của trang này, hai bài nào trả lời
   đúng hai việc đó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Ba bài đó là [345](345-run-stateless-application-vi.md) (chạy ứng dụng không trạng thái bằng
   Deployment), [346](346-scale-deployment-vi.md) (scale thủ công theo chiều ngang) và
   [348](348-update-deployment-rolling-vi.md) (cập nhật Deployment mà không gây gián đoạn). **Đọc
   hết ba bài đó rồi dừng**; tám link còn lại thuộc các giai đoạn khác. Trang này chỉ là mục lục,
   không có nội dung riêng để học.
2. Vì **thứ tự đọc lấy từ lộ trình, không lấy từ trang mục lục**. Hai bài đó nói về ứng dụng có
   trạng thái, cần volume và StorageClass — những thứ thuộc
   [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) và chưa được dạy. Đọc sớm thì bạn gặp
   một loạt khái niệm chưa có nền, đúng thứ mà khối hướng dẫn đọc này sinh ra để tránh. Sự gần nhau
   trên trang mục lục là do cách kubernetes.io tổ chức nhánh tài liệu, không phải do độ khó tăng dần.
3. Nâng số replica: [346](346-scale-deployment-vi.md). Cập nhật không gián đoạn:
   [348](348-update-deployment-rolling-vi.md). Cả hai đều thuộc nhóm 4a và là bài 6/7 và 7/7 của
   nhóm, tức đọc ngay sau [345](345-run-stateless-application-vi.md).

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

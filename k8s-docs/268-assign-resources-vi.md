# Gán thiết bị cho Pod và Container (Assign Devices to Pods and Containers)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/
>
> Gán tài nguyên hạ tầng cho các workload Kubernetes của bạn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13 — Lập lịch và workload nâng cao](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao)
→ dòng **Thực hành**, bài 2/5 · Bảng ánh xạ của [Lab 13 — DRA](labs/LAB-13-DRA.md) ghi rõ trang này
**không có thao tác riêng**: ba trang con phủ toàn bộ nội dung và được kiểm ở các phần B1, B3, B4
và B5 của lab.

Đây là **trang mục lục**, đọc trong vài phút. Giá trị duy nhất của nó là cho biết ba bài kế tiếp
chia việc theo vai trò nào.

**Phải hiểu ở lần đọc này:**

- Ba trang con chia theo **vai trò**, không theo độ khó:
  [271](271-set-up-dra-cluster-vi.md) là việc của **quản trị viên cluster** — bật API group và cấu
  hình các lớp thiết bị; [270](270-allocate-devices-dra-vi.md) là việc của **người vận hành
  workload** — cấp phát thiết bị cho Pod của mình; [269](269-access-dra-device-metadata-vi.md) là
  việc của **chính container** — đọc file JSON metadata tại các đường dẫn quy ước bên trong nó.
- Thứ tự phụ thuộc đi đúng theo ba vai trò đó: chưa có phần việc của quản trị viên ở 271 thì 270
  không có lớp thiết bị nào để claim, và chưa cấp phát được thiết bị ở 270 thì 269 không có file
  metadata nào để đọc.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mọi chi tiết thao tác | trang gốc chỉ gồm tiêu đề, đoạn mô tả và ba liên kết — không có nội dung riêng nào để hiểu sâu | chính ba bài [271](271-set-up-dra-cluster-vi.md), [270](270-allocate-devices-dra-vi.md) và [269](269-access-dra-device-metadata-vi.md), ba bài kế của nhóm Thực hành này |

---

Đây là trang mục lục (section index). Trang gốc chỉ gồm tiêu đề, phần mô tả ở trên và danh
sách các trang con dưới đây; nội dung chi tiết nằm trong từng trang con:

* [Thiết lập DRA trong một cluster (Set Up DRA in a Cluster)](271-set-up-dra-cluster-vi.md) —
  Trang này hướng dẫn cách cấu hình cấp phát tài nguyên động (dynamic resource allocation — DRA)
  trong một cluster Kubernetes bằng cách bật các API group và cấu hình các lớp thiết bị
  (classes of devices). Các hướng dẫn này dành cho quản trị viên cluster.
* [Cấp phát thiết bị cho workload bằng DRA (Allocate Devices to Workloads with DRA)](270-allocate-devices-dra-vi.md) —
  Trang này hướng dẫn cách cấp phát thiết bị cho các Pod của bạn bằng cấp phát tài nguyên động
  (DRA). Các hướng dẫn này dành cho người vận hành workload.
* [Truy cập metadata thiết bị DRA (Access DRA Device Metadata)](269-access-dra-device-metadata-vi.md) —
  Trang này hướng dẫn cách truy cập metadata của thiết bị từ các container sử dụng cấp phát
  tài nguyên động (DRA), bằng cách đọc các file JSON tại những đường dẫn quy ước bên trong
  container.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 13:

1. Ba trang con phục vụ ba vai trò nào? Trang nào là việc của quản trị viên cluster, trang nào là
   việc của người vận hành workload?
2. **Câu bẫy.** Tên trang là "Gán thiết bị cho Pod và Container" và dòng mô tả là "gán tài nguyên
   hạ tầng cho các workload Kubernetes của bạn". Vậy việc đặt `requests` và `limits` cho CPU và
   memory của container có nằm trong nhóm ba trang này không?
3. Cluster lab ba VM của bạn — `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2` — chưa có lớp
   thiết bị nào và chưa cài DRA driver nào. Theo chính mô tả trên trang mục lục, bạn phải bắt đầu
   từ trang con nào, và vì sao không bắt đầu được từ hai trang kia?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **[271](271-set-up-dra-cluster-vi.md) — quản trị viên cluster**: cấu hình DRA bằng cách bật các
   API group và cấu hình các lớp thiết bị. **[270](270-allocate-devices-dra-vi.md) — người vận
   hành workload**: cấp phát thiết bị cho các Pod của mình bằng DRA.
   **[269](269-access-dra-device-metadata-vi.md) — phía container**: đọc metadata thiết bị từ các
   file JSON tại những đường dẫn quy ước bên trong container.
2. **Không.** Cả ba trang con đều nói về **thiết bị (device) qua DRA** — mô tả của từng trang con
   đều gắn với cấp phát tài nguyên động. "Tài nguyên hạ tầng" ở đây là **thiết bị gắn kèm**, không
   phải `requests`/`limits` của CPU và memory. Đây là chỗ dễ nhầm vì tên nhóm nghe rất giống chủ
   đề tài nguyên thông thường của Pod.
3. Bắt đầu từ **[271](271-set-up-dra-cluster-vi.md)**, và đó là **việc của quản trị viên cluster**.
   Hai trang kia không đứng một mình được: mô tả của 270 là *cấp phát thiết bị cho các Pod*, tức
   cần lớp thiết bị mà 271 cấu hình ra; còn 269 chỉ có file metadata để đọc khi thiết bị đã được
   cấp phát ở 270.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

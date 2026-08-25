# Đưa dữ liệu vào ứng dụng (Inject Data Into Applications)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/inject-data-application/>
>
> Chỉ định cấu hình và các dữ liệu khác cho các Pod chạy workload của bạn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3b. Cấu hình ứng dụng](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod),
bài 7/12 · Kiểm chứng ở [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) phần B4 (`command`, `args`,
biến môi trường), B5 (biến lấy từ file) và B8 (Pod tiêu thụ Secret).

Đây là **trang mục lục**, đọc trong vài phút. Nó chỉ liệt kê các trang con của mục *Inject Data
Into Applications*.

**Phải hiểu ở lần đọc này:**

- Trang này không dạy cơ chế nào: nội dung duy nhất là danh sách **bảy trang con**. Nhìn tên chúng
  là thấy ba đường đưa dữ liệu vào một Pod mà mục này gom lại — qua **command và argument**
  ([330](330-define-command-argument-vi.md)), qua **biến môi trường**
  ([331](331-define-environment-variable-vi.md), [332](332-define-env-via-file-vi.md),
  [333](333-interdependent-env-variables-vi.md), [336](336-env-variable-expose-pod-info-vi.md)), và
  qua **file** ([335](335-downward-api-volume-vi.md)) — cộng một trang riêng về đưa thông tin xác
  thực vào Pod bằng Secret ([334](334-distribute-credentials-secure-vi.md)).
- Thứ tự liệt kê ở đây **không phải thứ tự đọc**. Lộ trình xếp 330, 331, 332, 333 và 334 vào nhóm
  3b, còn hai trang Downward API — [335](335-downward-api-volume-vi.md) và
  [336](336-env-variable-expose-pod-info-vi.md) — thuộc nhóm
  [3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời). Mục lục gộp chung vì kubernetes.io tổ chức theo
  chủ đề, không theo trình tự học.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hai trang con [336](336-env-variable-expose-pod-info-vi.md) và [335](335-downward-api-volume-vi.md) | thuộc nhóm 3a của lộ trình, đi cùng bài Downward API chứ không cùng ConfigMap/Secret | nhóm [3a. Pod và vòng đời](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), cùng giai đoạn 3 |

---

Mục này bao gồm các trang sau:

* [Định nghĩa command và argument cho container (Define a Command and Arguments for a Container)](330-define-command-argument-vi.md)
* [Định nghĩa các biến môi trường phụ thuộc lẫn nhau (Define Dependent Environment Variables)](333-interdependent-env-variables-vi.md)
* [Định nghĩa biến môi trường cho container (Define Environment Variables for a Container)](331-define-environment-variable-vi.md)
* [Định nghĩa giá trị biến môi trường bằng init container (Define Environment Variable Values Using An Init Container)](332-define-env-via-file-vi.md)
* [Cung cấp thông tin Pod cho container thông qua biến môi trường (Expose Pod Information to Containers Through Environment Variables)](336-env-variable-expose-pod-info-vi.md)
* [Cung cấp thông tin Pod cho container thông qua file (Expose Pod Information to Containers Through Files)](335-downward-api-volume-vi.md)
* [Phân phối thông tin xác thực một cách an toàn bằng Secret (Distribute Credentials Securely Using Secrets)](334-distribute-credentials-secure-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Nhìn tên bảy trang con, kể ra ba đường mà mục này dùng để "đưa dữ liệu vào ứng dụng".
2. **Câu bẫy.** Thứ tự liệt kê trên trang này có phải thứ tự nên đọc không, và bảy trang đó có
   thuộc cùng một nhóm bài của lộ trình không?
3. Trong bảy trang, những trang nào bạn thực hành ở [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md), và
   hai trang còn lại được thực hành ở nhóm nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Qua `command` và `args`** của container (trang *Định nghĩa command và argument*); **qua biến
   môi trường** (bốn trang: định nghĩa biến, định nghĩa biến bằng init container, biến phụ thuộc
   lẫn nhau, và biến chứa thông tin Pod); và **qua file** đưa vào container (trang thông tin Pod
   dạng file). Trang thứ bảy — phân phối thông tin xác thực bằng Secret — đứng riêng vì nó nói về
   **nguồn dữ liệu**, không phải về đường đưa vào.
2. **Không, cả hai câu.** Đây là mục lục theo chủ đề của kubernetes.io: thứ tự liệt kê là thứ tự
   hiển thị của trang web, không phải thứ tự học. Và bảy trang **không cùng một nhóm bài**: năm
   trang 330–334 nằm ở nhóm 3b, còn hai trang Downward API (335 và 336) lộ trình xếp vào
   [nhóm 3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời). Lấy mục lục làm lộ trình là cách chắc chắn
   để đọc nhầm thứ tự.
3. Ở Lab 3b bạn thực hành **năm trang thuộc nhóm 3b**: 330 (`command`/`args`), 331 (biến môi
   trường), 332 (biến lấy từ file do init container ghi), 333 (biến phụ thuộc), 334 (credential
   bằng Secret). **Hai trang Downward API — 335 và 336 — thuộc nhóm 3a**, thực hành cùng lab của
   nhóm đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

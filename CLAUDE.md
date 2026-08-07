# Hướng dẫn cho Claude Code trong repo này

## Bản đồ repo

| Thư mục | Nội dung |
| --- | --- |
| `k8s-docs/` | 185 bài dịch tiếng Việt của kubernetes.io, tên file `<số>-<slug>-vi.md` |
| `k8s-docs/LO-TRINH-ADMIN.md` | Giáo trình 15 giai đoạn; **thứ tự đọc** nằm ở đây, không phải ở số thứ tự file |
| `k8s-docs/README.md` | Mục lục tra cứu theo chủ đề |
| `k8s-docs/labs/` | Runbook thực hành đi kèm lộ trình |
| `ansible/` | Playbook cài đặt Kubernetes |
| `course-website/` | Ứng dụng demo |

Số trong tên file `k8s-docs/` là mã định danh bám theo cấu trúc mục của kubernetes.io, **không
phải thứ tự đọc**. Đừng suy ra thứ tự học từ con số.

---

## Viết lab trong `k8s-docs/labs/`

Phần này là bắt buộc với mọi phiên làm việc tạo hoặc sửa file trong `k8s-docs/labs/`.

### Trước khi viết

Đọc theo đúng thứ tự sau, không bỏ bước:

1. [`k8s-docs/labs/README.md`](k8s-docs/labs/README.md) — bản đồ lab, chuỗi snapshot, sổ nợ lab.
2. [`k8s-docs/labs/LAB-00-MOI-TRUONG.md`](k8s-docs/labs/LAB-00-MOI-TRUONG.md) — baseline phiên bản và gate môi trường.
3. [`k8s-docs/LO-TRINH-ADMIN.md`](k8s-docs/LO-TRINH-ADMIN.md) — xác định chính xác nhóm bài mà lab phải phủ.
4. **Toàn bộ** file `.md` của các bài trong nhóm đó. Không viết lab từ kiến thức chung về
   Kubernetes; nội dung lab phải bám đúng những gì bài dịch trình bày.
5. [`k8s-docs/labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md`](k8s-docs/labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md)
   — lab mẫu, dùng làm chuẩn về cấu trúc và giọng văn.

### Nguyên tắc chia lab

**Một lab = một nhóm bài trong lộ trình**, không phải một giai đoạn. Tách lab khi vi phạm một
trong ba điều:

1. Trạng thái cluster đổi (cần CNI khác, StorageClass, ingress controller, metrics-server,
   topology HA) — phải là lab riêng vì nó tạo snapshot mới.
2. Thời lượng vượt 2–4 giờ.
3. Checkpoint vượt khoảng 8–12 gạch đầu dòng vấn đáp.

Giai đoạn nhỏ và đồng nhất thì một lab là đủ. Không tự ý gộp nhiều nhóm bài vào một lab để
"cho gọn", cũng không cắt nhỏ hơn nhóm bài.

### Nguyên tắc không nhảy cóc

Lab **không được** dùng khái niệm chưa dạy trong lộ trình để làm cho bài chạy được, và
**không được** cài trước hạ tầng của giai đoạn sau. Khi một bài cần thứ thuộc giai đoạn sau:

1. Thực hành phần đạt được bằng kiến thức đã có.
2. Ghi phần còn lại vào [sổ nợ lab](k8s-docs/labs/README.md#5-sổ-nợ-lab), kèm lab sẽ trả nợ.
3. Chỉ khi cả hai cách trên bất khả thi mới đề xuất đổi thứ tự lộ trình — và phải hỏi người
   dùng trước, không tự đổi.

Ví dụ đã quyết: HPA/VPA đọc lý thuyết ở giai đoạn 4 nhưng thực hành ở Lab 11b, vì cần
metrics-server của giai đoạn 11. Không cài metrics-server sớm.

Lab phát sinh nợ phải nói rõ trong checkpoint rằng nợ chưa trả. Lab trả nợ phải nhắc người
học đọc lại bài gốc trước khi làm.

### Cấu trúc bắt buộc của một file lab

Đặt tên `LAB-<mã>-<TIÊU-ĐỀ-VIẾT-HOA-KHÔNG-DẤU>.md`, ví dụ `LAB-1B-OBJECT-LABEL-VA-KUBECTL.md`.

Thứ tự các mục:

1. Tiêu đề `# Lab <mã> — <tên>`.
2. Blockquote khai báo **điểm bắt đầu** (snapshot nào) và **điểm kết thúc** (trả về snapshot
   cũ hay tạo snapshot mới), kèm ngày đối chiếu phiên bản.
3. Câu dẫn link tới đúng mục trong `LO-TRINH-ADMIN.md` và tới quy trình mở đầu ở
   [A5.5 của Lab 00](k8s-docs/labs/LAB-00-MOI-TRUONG.md#a55-quy-trình-mở-đầu-mỗi-lab).
4. `## 1. Kết quả phải đạt` — viết ở dạng "chứng minh và giải thích được…", không phải danh
   sách lệnh.
5. `### 1.1. Ánh xạ tài liệu sang bài thực hành` — bảng hai cột. **Mỗi bài trong nhóm phải
   xuất hiện ít nhất một lần.** Nếu một bài không có cách kiểm chứng thực tế, ghi rõ lý do
   thay vì bỏ trống.
6. `### 1.2. Thời lượng`.
7. `## 2. Quy ước và an toàn`.
8. `# Phần B — Thực hành …` với các mục `B0`, `B1`, … theo đúng thứ tự bảng ánh xạ. Giữ tiền
   tố `B` để đồng bộ với lab đã có; `Phần A` chỉ tồn tại trong Lab 00.
9. Mục cuối của phần B là **cleanup và gate cuối**.
10. `## 3. Checkpoint <mã>` — 8–12 checkbox vấn đáp không nhìn tài liệu, cộng một "Bài giải
    thích cuối cùng" dạng kể lại luồng trong vài phút.
11. `## 4. Troubleshooting của lab này` — **chỉ** sự cố phát sinh trong nội dung bài học. Sự
    cố dựng môi trường link về mục 4 của Lab 00.
12. `## 5. Nguồn chính thức` — link kubernetes.io tương ứng.

### Quy tắc nội dung

- **Không lặp lại phần môi trường.** Không chép hướng dẫn cài OS, containerd, kubeadm hay CNI
  vào lab; link về Lab 00.
- **Số phiên bản chỉ tồn tại ở** [bảng A1.3 của Lab 00](k8s-docs/labs/LAB-00-MOI-TRUONG.md#a13-phiên-bản-được-khóa).
  Lab khác cần nói tới phiên bản thì link về đó, tuyệt đối không chép lại con số.
- **Gate `PASS:`** sau mỗi bước có thể sai: một dòng bắt đầu bằng `PASS:` mô tả điều kiện phải
  đạt. Bước nào không kiểm chứng được thì không đưa vào lab.
- **Bằng chứng** ghi vào `~/lab-evidence/<mã lab>/`, ví dụ `~/lab-evidence/1b/`.
- **Fault injection** chỉ chạy trên `k8s-worker2`.
- **Snapshot**: lab không tạo snapshot mới thì gate cuối phải chứng minh cluster đã về đúng
  trạng thái đầu vào.
- Không dùng minikube, kind hay cluster dùng chung. Mọi lab chạy trên cluster VM của Lab 00.
- Lệnh chạy trên node nào phải nói rõ; mặc định là `k8s-master`.

### Độ chính xác kỹ thuật

- Mọi lệnh, flag, field và đường dẫn phải đúng với phiên bản baseline trong Lab 00. Khi không
  chắc một field hay flag có tồn tại ở version đó, **kiểm tra tài liệu chính thức của đúng
  version rồi mới viết** — không đoán, không suy từ version khác.
- Không viết con số thời gian như một cam kết (timeout của controller, thời gian hội tụ). Nếu
  phải nêu, ghi kèm "phụ thuộc cấu hình" như lab 1a đang làm.
- Ưu tiên lệnh có thể verify được kết quả bằng `test`, `grep -q` hoặc so sánh giá trị, thay vì
  bảo người học "quan sát thấy".

### Sau khi viết xong một lab

Cập nhật đủ ba chỗ, nếu thiếu thì coi như chưa xong:

1. [`k8s-docs/labs/README.md`](k8s-docs/labs/README.md) — đổi trạng thái `⬜ chưa viết` thành
   `✅ đã viết` và trỏ link; bổ sung dòng vào bảng chuỗi snapshot nếu lab tạo snapshot mới;
   bổ sung sổ nợ nếu lab phát sinh hoặc trả nợ.
2. [`k8s-docs/LO-TRINH-ADMIN.md`](k8s-docs/LO-TRINH-ADMIN.md) — đổi mục `🧪` tương ứng từ
   "chưa viết" thành link tới file lab.
3. Kiểm tra mọi link tương đối trong file mới thực sự trỏ đúng file và đúng anchor.

### Văn phong

- Tiếng Việt, giọng runbook: câu ngắn, chỉ dẫn trực tiếp, không văn hoa.
- Giữ nguyên tiếng Anh các thuật ngữ Kubernetes và tên đối tượng: Pod, Node, Deployment,
  control plane, `spec`, `status`, label, selector, namespace.
- Xuống dòng ở khoảng 100 ký tự trong file lab. `LO-TRINH-ADMIN.md` giữ dòng dài, không wrap.
- Dùng backtick cho tên lệnh, file, field và giá trị.

---

## Dịch tài liệu kubernetes.io

Khi được yêu cầu dịch một trang kubernetes.io sang tiếng Việt, dùng agent `k8s-docs-translator`.
Quy ước: file đặt tên `<số>-<slug>-vi.md` trong `k8s-docs/`, giữ nguyên cấu trúc và thứ tự mục
của trang gốc, có link trang nguồn ở đầu file. Sau khi thêm bài mới, cập nhật cả
`k8s-docs/README.md` và `k8s-docs/LO-TRINH-ADMIN.md`.

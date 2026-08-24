# Các Container (Containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/containers/>
>
> Công nghệ đóng gói một ứng dụng cùng với các dependency lúc chạy (runtime dependency) của nó.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 2](00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime), bài 1/8 ·
Kiểm chứng ở Lab 2 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài mở đầu giai đoạn, rất ngắn và chỉ đặt vấn đề. Giá trị nằm ở **một nguyên tắc** mà mọi thứ
sau đó dựa vào.

**Phải hiểu ở lần đọc này:**

- Container được thiết kế **bất biến**: muốn đổi thì build image mới rồi tạo lại container,
  **không sửa container đang chạy**. Đây là nguyên tắc chi phối toàn bộ cách vận hành workload
  ở giai đoạn 3 và 4.
- Container image là gói chứa mã, runtime, thư viện và giá trị mặc định — tức là thứ làm cho
  container "chạy đâu cũng như nhau".
- Kubernetes không tự chạy container; nó nhờ **container runtime**, và runtime nào cũng được
  miễn hiện thực CRI.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nhắc tới Pod và việc container được lập lịch cùng nhau | chưa học Pod | giai đoạn 3 |
| RuntimeClass | có bài riêng ở cuối giai đoạn này | bài [43](43-runtime-class-vi.md) |

---

Trang này sẽ thảo luận về container và container image, cũng như việc sử dụng chúng
trong vận hành (operations) và phát triển giải pháp (solution development).

Từ _container_ là một thuật ngữ mang nhiều nghĩa (overloaded term). Bất cứ khi nào bạn
dùng từ này, hãy kiểm tra xem người nghe có đang dùng cùng một định nghĩa hay không.

Mỗi container mà bạn chạy đều có thể lặp lại được (repeatable); việc chuẩn hóa nhờ đã
bao gồm sẵn các dependency đồng nghĩa với việc bạn nhận được cùng một hành vi ở bất kỳ
đâu bạn chạy nó.

Container tách rời (decouple) ứng dụng khỏi hạ tầng host bên dưới. Điều này giúp việc
triển khai dễ dàng hơn trên các môi trường cloud hoặc hệ điều hành khác nhau.

Mỗi node trong một cluster Kubernetes chạy các container tạo nên các
[Pod](46-pods-vi.md) được gán cho node đó.
Các container trong một Pod được đặt cùng chỗ (co-located) và được lập lịch cùng nhau
(co-scheduled) để chạy trên cùng một node.

## Container image (Container images)

Một [container image](./40-images-vi.md) là một gói phần mềm sẵn sàng để chạy, chứa
mọi thứ cần thiết để chạy một ứng dụng: mã nguồn và mọi runtime mà nó cần, các thư viện
của ứng dụng và của hệ thống, cùng các giá trị mặc định cho mọi thiết lập thiết yếu.

Container được thiết kế để không lưu trạng thái (stateless) và
[bất biến (immutable)](https://glossary.cncf.io/immutable-infrastructure/):
bạn không nên thay đổi mã của một container đang chạy. Nếu bạn có một ứng dụng đã được
container hóa và muốn thực hiện thay đổi, quy trình đúng là build một image mới bao gồm
thay đổi đó, rồi tạo lại container để khởi động từ image đã được cập nhật.

## Container runtime (Container runtimes) {#container-runtimes}

Một thành phần nền tảng giúp Kubernetes chạy các container một cách hiệu quả.
Nó chịu trách nhiệm quản lý việc thực thi và vòng đời (lifecycle) của các container
trong môi trường Kubernetes.

Kubernetes hỗ trợ các container runtime như containerd, CRI-O, và bất kỳ hiện thực
(implementation) nào khác của
[Kubernetes CRI (Container Runtime Interface)](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-node/container-runtime-interface.md).

Thông thường, bạn có thể để cluster tự chọn container runtime mặc định cho một Pod.
Nếu bạn cần dùng nhiều hơn một container runtime trong cluster, bạn có thể chỉ định
[RuntimeClass](43-runtime-class-vi.md)
cho một Pod để bảo đảm rằng Kubernetes chạy các container đó bằng một container runtime
cụ thể.

Bạn cũng có thể dùng RuntimeClass để chạy các Pod khác nhau với cùng một container
runtime nhưng với các thiết lập khác nhau.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Ứng dụng trong container cần vá một dòng cấu hình. Quy trình đúng theo bài này là gì, và vì
   sao không phải là `exec` vào container rồi sửa?
2. Kubernetes có tự chạy container không? Nếu không thì ai chạy, và Kubernetes ràng buộc thứ
   đó bằng cái gì?
3. Bài mở đầu bằng câu "container là một thuật ngữ mang nhiều nghĩa". Kể hai nghĩa khác nhau
   mà từ này có thể mang.

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Build một image mới** chứa thay đổi, rồi **tạo lại container** từ image đã cập nhật. Sửa
   trực tiếp phá vỡ tính bất biến: container đang chạy không còn khớp với image sinh ra nó, nên
   lần tạo lại tiếp theo — do restart, do rollout, do node chết — thay đổi của bạn biến mất mà
   không có dấu vết.
2. **Không.** Container runtime chạy — containerd, CRI-O hoặc bất kỳ hiện thực nào khác.
   Kubernetes ràng buộc chúng bằng **CRI (Container Runtime Interface)**; runtime nào hiện thực
   đúng giao diện đó thì dùng được.
3. Ví dụ: **image** (gói phần mềm nằm yên trên đĩa) so với **tiến trình đang chạy** sinh ra từ
   image đó; hoặc **container theo nghĩa Linux** (namespace + cgroup) so với **container trong
   spec của một Pod**. Bài dặn mỗi lần dùng từ này nên kiểm tra người nghe đang hiểu theo nghĩa
   nào.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

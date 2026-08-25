# Môi trường Container (Container Environment)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/containers/container-environment/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 2](00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime), bài 3/8 ·
Kiểm chứng ở [Lab 2](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md).

Bài rất ngắn, trả lời đúng một câu hỏi: **tiến trình bên trong container nhìn thấy gì?**

**Phải hiểu ở lần đọc này:**

- Filesystem mà container thấy là **image cộng với các volume được mount** — không phải chỉ
  image.
- **Hostname của container là tên Pod**, không phải tên container. Đây là chi tiết hay gây bất
  ngờ khi debug.
- Kubernetes bơm sẵn vào container: biến môi trường của **Service** cùng namespace, cộng với
  các biến bạn tự khai báo và biến tĩnh trong image.
- Điểm bẫy quan trọng: biến môi trường Service chỉ chứa **những Service đã tồn tại lúc
  container được tạo**. Service tạo sau sẽ không xuất hiện — và đó chính là lý do người ta dùng
  DNS thay vì biến môi trường.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Định dạng `FOO_SERVICE_HOST` / `FOO_SERVICE_PORT` | chưa học Service | giai đoạn 5 |
| Truy cập Service qua DNS | cần DNS trong cluster | giai đoạn 5 |
| Downward API | có bài riêng | giai đoạn 3, bài [56](56-downward-api-vi.md) |
| Volume | có nhóm bài riêng | giai đoạn 6 |

---

Trang này mô tả các tài nguyên khả dụng cho các Container trong môi trường Container.

## Môi trường container (Container environment)

Môi trường Container của Kubernetes cung cấp một số tài nguyên quan trọng cho các Container:

* Một hệ thống tệp (filesystem), là sự kết hợp của một [image](./40-images-vi.md)
  và một hoặc nhiều [volume](91-volumes-vi.md).
* Thông tin về chính Container đó.
* Thông tin về các đối tượng (object) khác trong cluster.

### Thông tin về Container (Container information)

*Hostname* của một Container là tên của Pod mà Container đó đang chạy bên trong.
Nó có thể được lấy thông qua lệnh `hostname` hoặc lời gọi hàm
[`gethostname`](https://man7.org/linux/man-pages/man2/gethostname.2.html)
trong libc.

Tên Pod và namespace được cung cấp dưới dạng các biến môi trường (environment variable)
thông qua [downward API](335-downward-api-volume-vi.md).

Các biến môi trường do người dùng định nghĩa trong định nghĩa Pod cũng khả dụng
cho Container, cùng với mọi biến môi trường được chỉ định tĩnh trong container image.

### Thông tin về cluster (Cluster information)

Danh sách tất cả các service đang chạy tại thời điểm một Container được tạo sẽ khả dụng
cho Container đó dưới dạng các biến môi trường.
Danh sách này giới hạn trong các service thuộc cùng namespace với Pod của Container mới
và các service của control plane Kubernetes.

Với một service tên là *foo* expose một tập các Pod, mỗi Pod chạy một container tên là *bar*,
các biến sau được định nghĩa:

```shell
FOO_SERVICE_HOST=<the host the service is running on>
FOO_SERVICE_PORT=<the port the service is running on>
```

Các Service có địa chỉ IP riêng và khả dụng cho Container thông qua DNS,
nếu [DNS addon](https://releases.k8s.io/v1.36.0/cluster/addons/dns/) được bật.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [các hook vòng đời Container (Container lifecycle hooks)](42-container-lifecycle-hooks-vi.md).
* Thực hành thực tế việc
  [gắn handler vào các sự kiện trong vòng đời Container](272-attach-handler-lifecycle-event-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Bạn `exec` vào một container và chạy `hostname`. Kết quả là tên container hay tên Pod?
2. Filesystem mà container nhìn thấy được ghép từ những nguồn nào?
3. Một Service được tạo **sau** khi container đã khởi động. Biến môi trường tương ứng có xuất
   hiện trong container đó không? Hệ quả thực tế là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Tên Pod.** Bài nói rõ hostname của một container là tên của Pod mà container đó chạy bên
   trong. Một Pod nhiều container thì mọi container đều có cùng hostname.
2. **Image cộng với một hoặc nhiều volume** được mount vào. Image cung cấp lớp nền, volume phủ
   thêm dữ liệu lên các đường dẫn cụ thể.
3. **Không.** Danh sách biến môi trường Service chỉ gồm các Service **đang chạy tại thời điểm
   container được tạo**. Hệ quả: không thể dựa vào biến môi trường để tìm Service tạo sau — đó
   là lý do cách tìm Service đáng tin cậy là **DNS**, thứ tra cứu lúc chạy chứ không đông cứng
   lúc khởi động.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

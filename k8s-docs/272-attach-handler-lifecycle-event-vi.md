# Gắn handler vào các sự kiện vòng đời của Container (Attach Handlers to Container Lifecycle Events)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3a. Pod và vòng đời](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài thực hành 3/11 ·
Kiểm chứng ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B7.1 và B7.3.

Bài ngắn, nhưng mục *Thảo luận* ở cuối mới là phần đáng đọc kỹ: nó nói về **thời điểm** và **thứ
tự** — thứ mà manifest ví dụ không cho thấy.

**Phải hiểu ở lần đọc này:**

- Có đúng **hai sự kiện**: `postStart` gửi ngay sau khi Container được khởi động, `preStop` gửi
  ngay trước khi Container bị chấm dứt. **Mỗi Container chỉ chỉ định được một handler cho mỗi sự
  kiện.**
- `postStart` **không** được đảm bảo gọi trước entrypoint của Container — nó chạy **bất đồng bộ**
  với mã của Container. Nhưng việc quản lý container của Kubernetes **bị chặn** cho tới khi
  handler xong, và trạng thái Container **không chuyển sang RUNNING** trước lúc đó.
- `preStop` cũng **chặn** việc quản lý container cho tới khi xong, **trừ khi khoảng ân hạn (grace
  period) của Pod hết hạn** — đây là mối nối trực tiếp sang `terminationGracePeriodSeconds` của
  bài [47](47-pod-lifecycle-vi.md).
- Việc `preStop` làm trong ví dụ có ý nghĩa thực tế: `nginx -s quit` rồi vòng lặp
  `while killall -0 nginx; do sleep 1; done` — tức **tắt êm ứng dụng và chờ nó chết hẳn** trước
  khi container biến mất.
- Ranh giới của `preStop`: Kubernetes chỉ gửi sự kiện này khi Pod hoặc container bị **chấm dứt**
  (terminated). Hook **không được gọi khi Pod hoàn thành** (completed).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* gợi ý minikube và các playground | lộ trình không dùng minikube; cluster lab đã dựng sẵn | [Lab 00 — Môi trường](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Chi tiết đầy đủ của hạn chế "hook không chạy khi container hoàn thành" | ở đây chỉ cần biết ranh giới, chưa cần biết vì sao | bài [42](42-container-lifecycle-hooks-vi.md#container-hooks) đã đọc ở [giai đoạn 2](00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime) |
| Mục *Tham khảo* trỏ tới API reference `Lifecycle`, `Container`, `PodSpec` | là chỗ tra field, không phải bài đọc | tra khi viết manifest ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B7 |

---

Trang này hướng dẫn cách gắn handler vào các sự kiện vòng đời (lifecycle events) của Container.
Kubernetes hỗ trợ hai sự kiện postStart và preStop. Kubernetes gửi sự kiện postStart ngay sau
khi một Container được khởi động, và gửi sự kiện preStop ngay trước khi Container bị chấm dứt
(terminated). Mỗi Container có thể chỉ định một handler cho mỗi sự kiện.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy nhập `kubectl version`.

## Định nghĩa handler postStart và preStop (Define postStart and preStop handlers)

Trong bài thực hành này, bạn sẽ tạo một Pod có một Container. Container này có handler cho
các sự kiện postStart và preStop.

Đây là file cấu hình cho Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```

Trong file cấu hình, bạn có thể thấy lệnh postStart ghi một file `message` vào thư mục
`/usr/share` của Container. Lệnh preStop tắt nginx một cách êm thấm (gracefully). Điều này
hữu ích khi Container bị chấm dứt do một sự cố.

Tạo Pod:

    kubectl apply -f https://k8s.io/examples/pods/lifecycle-events.yaml

Xác nhận rằng Container trong Pod đang chạy:

    kubectl get pod lifecycle-demo

Mở một shell vào Container đang chạy trong Pod của bạn:

    kubectl exec -it lifecycle-demo -- /bin/bash

Trong shell, xác nhận rằng handler `postStart` đã tạo file `message`:

    root@lifecycle-demo:/# cat /usr/share/message

Kết quả hiển thị đoạn văn bản do handler postStart ghi ra:

    Hello from the postStart handler

## Thảo luận (Discussion)

Kubernetes gửi sự kiện postStart ngay sau khi Container được tạo. Tuy nhiên, không có gì đảm
bảo rằng handler postStart được gọi trước khi entrypoint của Container được gọi. Handler
postStart chạy bất đồng bộ (asynchronously) so với mã của Container, nhưng việc quản lý
container của Kubernetes sẽ bị chặn (block) cho đến khi handler postStart hoàn thành. Trạng
thái của Container không được đặt thành RUNNING cho đến khi handler postStart hoàn thành.

Kubernetes gửi sự kiện preStop ngay trước khi Container bị chấm dứt. Việc quản lý Container
của Kubernetes sẽ bị chặn cho đến khi handler preStop hoàn thành, trừ khi khoảng thời gian ân
hạn (grace period) của Pod hết hạn. Để biết thêm chi tiết, xem
[Vòng đời của Pod](47-pod-lifecycle-vi.md).

> **Ghi chú:**
>
> Kubernetes chỉ gửi sự kiện preStop khi một Pod hoặc một container trong Pod bị *chấm dứt*
> (terminated). Điều này có nghĩa là hook preStop không được gọi khi Pod *hoàn thành*
> (completed). Về hạn chế này, vui lòng xem chi tiết tại
> [Container hooks](42-container-lifecycle-hooks-vi.md#container-hooks).

## Tiếp theo (What's next)

* Tìm hiểu thêm về [các hook vòng đời của Container](42-container-lifecycle-hooks-vi.md).
* Tìm hiểu thêm về [vòng đời của một Pod](47-pod-lifecycle-vi.md).

### Tham khảo (Reference)

* [Lifecycle](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#lifecycle-v1-core)
* [Container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#container-v1-core)
* Xem `terminationGracePeriodSeconds` trong [PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podspec-v1-core)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. **Câu bẫy.** Tên `postStart` nghe như "chạy trước, rồi mới tới ứng dụng". Bài khẳng định điều
   gì về thứ tự giữa handler `postStart` và entrypoint của Container? Vậy Kubernetes vẫn *chặn*
   cái gì trong lúc handler chạy, và trạng thái Container lúc đó là gì?
2. Bạn ghim một Pod có handler `preStop` là vòng lặp chờ rất lâu vào `lab-k8s-worker2` rồi xóa
   Pod. Theo bài, cái gì giới hạn thời gian handler đó thực sự được chạy?
3. Handler `preStop` trong ví dụ chạy `nginx -s quit` rồi lặp `killall -0 nginx` cho tới khi
   nginx biến mất. Vì sao không dừng ở lệnh đầu tiên?
4. Bạn viết một container chạy xong một việc rồi tự thoát, và gắn cho nó handler `preStop` để
   dọn dẹp. Handler có chạy không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Bài nói thẳng: **không có gì đảm bảo handler `postStart` được gọi trước khi entrypoint của
   Container được gọi** — handler chạy **bất đồng bộ** so với mã của Container. Đây là chỗ dễ
   sai: cái tên gợi ý một thứ tự mà Kubernetes không hứa. Thứ Kubernetes có hứa là **việc quản lý
   container bị chặn cho tới khi handler `postStart` hoàn thành**, và **trạng thái Container không
   được đặt thành RUNNING** cho tới lúc đó.
2. **Khoảng thời gian ân hạn (grace period) của Pod.** Bài nói việc quản lý Container bị chặn cho
   tới khi handler `preStop` hoàn thành, **trừ khi grace period hết hạn** — hết hạn thì Kubernetes
   không chờ nữa. Chi tiết nằm ở [Vòng đời của Pod](47-pod-lifecycle-vi.md).
3. Vì `nginx -s quit` chỉ **ra lệnh** tắt êm, nó trả về ngay chứ không đợi nginx thực sự kết
   thúc. Vòng lặp `killall -0 nginx; sleep 1` giữ cho handler còn chạy **cho tới khi không còn
   tiến trình nginx nào**. Vì Kubernetes bị chặn trong lúc handler chạy, giữ handler sống đồng
   nghĩa với giữ container sống — đó là cách bài bảo đảm nginx tắt hẳn trước khi container bị lấy
   đi. Bài nêu công dụng: hữu ích khi Container bị chấm dứt do một sự cố.
4. **Không.** Ghi chú của bài nói rõ: Kubernetes **chỉ** gửi sự kiện `preStop` khi một Pod hoặc
   một container trong Pod bị **chấm dứt** (terminated); hook **không được gọi khi Pod hoàn
   thành** (completed). Container tự chạy xong rồi thoát rơi đúng vào trường hợp thứ hai, nên
   phần dọn dẹp phải nằm trong chính lệnh của container. Chi tiết ở
   [Container hooks](42-container-lifecycle-hooks-vi.md#container-hooks).

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

# Phát triển và debug service cục bộ bằng telepresence (Developing and debugging services locally using telepresence)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-cluster/local-debugging/>
>
> Tài liệu này mô tả cách dùng `telepresence` để phát triển và debug cục bộ các service đang
> chạy trên một cluster Kubernetes từ xa.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** tài liệu tra cứu thuộc nhánh `/docs/tasks/`
([Checkpoint tiếp nối](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)) — bài này
không thuộc CP nào của lộ trình vì hướng tới developer hơn là admin; nó là lời giải "hạng
nặng" cho vấn đề mà bài [304 — Lấy shell vào container đang chạy](304-get-shell-running-container-vi.md)
giải quyết thủ công.

Toàn bộ bài nói về một công cụ bên thứ ba (Telepresence). Với vai trò admin, bạn chỉ cần nắm
**ý tưởng** và **dấu vết nó để lại trên cluster**; không cần cài hay thao tác thành thạo.

**Phải hiểu ở lần đọc này:**

- Vấn đề gốc: ứng dụng gồm nhiều service chạy trong container trên cluster từ xa, nên vòng
  lặp sửa code – debug rất chậm, kể cả khi pipeline deploy nhanh — mỗi lần sửa vẫn phải deploy
  lại mới thấy kết quả.
- `telepresence connect` khởi động Daemon và nối máy trạm cục bộ vào cluster, cho phép gọi
  các service từ máy cục bộ bằng chính cú pháp DNS của Kubernetes
  (ví dụ `curl https://kubernetes.default`).
- `telepresence intercept` chuyển hướng traffic đang đi vào service trên cluster về process
  chạy trên máy cục bộ của bạn: sửa code cục bộ, lưu lại là thấy hiệu lực ngay, và có thể gắn
  debugger hoặc IDE — trong khi service vẫn truy cập được ConfigMap, secret và các service
  khác trên cluster.
- Cơ chế bên dưới: Telepresence cài một sidecar **traffic-agent** cạnh container của ứng dụng
  trong cluster để bắt traffic vào Pod và định tuyến về môi trường phát triển cục bộ — tức là
  nó **có sửa workload trên cluster**, không phải công cụ chỉ đọc.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Phân biệt chi tiết global intercept và personal intercept | thuộc tài liệu riêng của Telepresence, ngoài phạm vi kubernetes.io | tra cứu qua link trong bài khi cần |
| Tutorial Guestbook trên Google Kubernetes Engine ở mục Tiếp theo | cần môi trường GKE, không thuộc lộ trình admin | không bắt buộc |

---

> **Ghi chú:** Trang này đề cập đến các sản phẩm hoặc dự án bên thứ ba cung cấp chức năng mà
> Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những sản phẩm
> hoặc dự án đó. Để thêm một sản phẩm hoặc dự án vào danh sách này, hãy đọc
> [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> trước khi gửi thay đổi.

Các ứng dụng Kubernetes thường bao gồm nhiều service riêng biệt, mỗi service chạy trong
container của riêng nó. Việc phát triển và debug các service này trên một cluster Kubernetes
từ xa có thể rất phiền phức, đòi hỏi bạn phải
[lấy một shell vào container đang chạy](304-get-shell-running-container-vi.md) để chạy các
công cụ debug.

`telepresence` là công cụ giúp đơn giản hóa quá trình phát triển và debug service cục bộ,
trong khi vẫn proxy service đó tới một cluster Kubernetes từ xa. Dùng `telepresence` cho phép
bạn sử dụng các công cụ tùy ý, chẳng hạn debugger và IDE, cho một service cục bộ, đồng thời
cung cấp cho service đó quyền truy cập đầy đủ vào ConfigMap, secret và các service đang chạy
trên cluster từ xa.

Tài liệu này mô tả cách dùng `telepresence` để phát triển và debug cục bộ các service đang
chạy trên một cluster từ xa.

## Trước khi bạn bắt đầu (Before you begin)

* Đã cài đặt một cluster Kubernetes
* `kubectl` đã được cấu hình để giao tiếp với cluster
* Đã cài đặt [Telepresence](https://www.telepresence.io/docs/latest/quick-start/)

## Kết nối máy cục bộ của bạn với một cluster Kubernetes từ xa (Connecting your local machine to a remote Kubernetes cluster)

Sau khi cài đặt `telepresence`, chạy `telepresence connect` để khởi động Daemon của nó và kết
nối máy trạm cục bộ của bạn với cluster.

```
$ telepresence connect
 
Launching Telepresence Daemon
...
Connected to context default (https://<cluster public IP>)
```

Bạn có thể curl các service bằng cú pháp của Kubernetes, ví dụ
`curl -ik https://kubernetes.default`

## Phát triển hoặc debug một service có sẵn (Developing or debugging an existing service)

Khi phát triển một ứng dụng trên Kubernetes, bạn thường lập trình hoặc debug một service duy
nhất. Service đó có thể cần truy cập các service khác để kiểm thử và debug. Một lựa chọn là
dùng pipeline triển khai liên tục (continuous deployment), nhưng ngay cả pipeline triển khai
nhanh nhất cũng gây ra độ trễ trong chu trình lập trình hoặc debug.

Dùng lệnh `telepresence intercept $SERVICE_NAME --port $LOCAL_PORT:$REMOTE_PORT` để tạo một
"intercept" (chặn và chuyển hướng) cho việc định tuyến lại traffic của service từ xa.

Trong đó:

- `$SERVICE_NAME` là tên service cục bộ của bạn
- `$LOCAL_PORT` là port mà service của bạn đang chạy trên máy trạm cục bộ
- Và `$REMOTE_PORT` là port mà service của bạn lắng nghe trong cluster

Chạy lệnh này sẽ báo cho Telepresence gửi traffic từ xa đến service cục bộ của bạn, thay vì
đến service trong cluster Kubernetes từ xa. Hãy sửa mã nguồn của service ngay trên máy cục bộ,
lưu lại, và thấy các thay đổi tương ứng có hiệu lực ngay lập tức khi truy cập ứng dụng từ xa.
Bạn cũng có thể chạy service cục bộ bằng debugger hoặc bất kỳ công cụ phát triển cục bộ nào
khác.

## Telepresence hoạt động như thế nào? (How does Telepresence work?)

Telepresence cài một sidecar traffic-agent bên cạnh container của ứng dụng sẵn có đang chạy
trong cluster từ xa. Sau đó nó bắt (capture) toàn bộ các request đi vào Pod, và thay vì
chuyển tiếp chúng đến ứng dụng trong cluster từ xa, nó định tuyến toàn bộ traffic (khi bạn
tạo một [global intercept](https://www.getambassador.io/docs/telepresence/latest/concepts/intercepts/#global-intercept))
hoặc một phần traffic (khi bạn tạo một
[personal intercept](https://www.getambassador.io/docs/telepresence/latest/concepts/intercepts/#personal-intercept))
về môi trường phát triển cục bộ của bạn.

## Tiếp theo (What's next)

Nếu bạn quan tâm đến một hướng dẫn thực hành, hãy xem
[tutorial này](https://cloud.google.com/community/tutorials/developing-services-with-k8s),
trong đó trình bày cách phát triển cục bộ ứng dụng Guestbook trên Google Kubernetes Engine.

Để đọc thêm, hãy truy cập [website của Telepresence](https://www.telepresence.io).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn này.

1. Vì sao bài nói rằng ngay cả pipeline triển khai nhanh nhất cũng không đủ cho việc debug
   một service, và Telepresence rút ngắn chu trình đó ở khâu nào?
2. Trong lệnh `telepresence intercept $SERVICE_NAME --port $LOCAL_PORT:$REMOTE_PORT`, port
   nào là port trên máy trạm của bạn và port nào là port service lắng nghe trong cluster?
3. Câu bẫy: khi intercept đang hoạt động, code bạn vừa sửa chạy **ở đâu** — bên trong Pod
   trên cluster hay trên máy của bạn? Vậy service đó lấy ConfigMap và secret từ đâu?
4. Giả sử bạn thử Telepresence với cluster lab của mình: nó để lại dấu vết gì bên trong Pod
   của ứng dụng trên cluster, và điều đó có ý nghĩa gì với nguyên tắc giữ cluster lab sạch?
5. Sau khi chạy `telepresence connect` thành công, lệnh `curl -ik https://kubernetes.default`
   từ máy cục bộ chứng minh được điều gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì với pipeline, **mỗi lần sửa code vẫn phải deploy lại lên cluster rồi mới thấy kết quả**,
   nên luôn có độ trễ trong chu trình sửa – kiểm tra. Telepresence bỏ hẳn bước deploy: traffic
   từ cluster được **chuyển thẳng về service đang chạy trên máy cục bộ**, sửa code lưu lại là
   thấy hiệu lực ngay.
2. **`$LOCAL_PORT` là port service đang chạy trên máy trạm cục bộ; `$REMOTE_PORT` là port
   service lắng nghe trong cluster.** Thứ tự trong cú pháp là `local:remote`.
3. **Trên máy của bạn.** Intercept không đưa code vào cluster; nó chỉ định tuyến lại traffic
   của service từ xa về process cục bộ — nhờ vậy bạn gắn được debugger/IDE. Nhưng service vẫn
   được cấp **quyền truy cập đầy đủ vào ConfigMap, secret và các service trên cluster từ xa**,
   nên nó hành xử như đang chạy trong cluster. Trực giác "muốn debug bằng traffic thật thì
   code phải nằm trong cluster" là sai — đó chính là điều Telepresence đảo ngược.
4. Telepresence **cài một sidecar traffic-agent bên cạnh container của ứng dụng trong
   cluster** để bắt request đi vào Pod. Tức là workload trên cluster đã bị sửa đổi — cluster
   không còn ở trạng thái ban đầu, nên nếu thử trên cluster lab thì phải gỡ sạch hoặc khôi
   phục snapshot trước khi làm lab tiếp theo.
5. Nó chứng minh máy trạm cục bộ **đã được nối vào mạng của cluster và phân giải được tên
   service bằng cú pháp DNS của Kubernetes** — `kubernetes.default` là tên Service nội bộ mà
   bình thường chỉ gọi được từ bên trong cluster.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

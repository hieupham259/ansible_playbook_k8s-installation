# Tầng tổng hợp API của Kubernetes (Kubernetes API Aggregation Layer)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/>
>
> Tầng tổng hợp (aggregation layer) cho phép mở rộng Kubernetes bằng các API bổ sung, vượt ra ngoài những gì các API lõi của Kubernetes cung cấp.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), bài 4/7 ·
Kiểm chứng ở Lab 14 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Giai đoạn này lộ trình ghi rõ là **dành cho platform administrator / người phát triển operator**.

**Đây là món nợ từ giai đoạn 1.** Ở bài [21 — Kubernetes API](21-kubernetes-api-vi.md), bảng
"Đọc lướt" đã xếp *custom resource và aggregation layer* vào loại hoãn lại, kèm câu "Lộ trình đã
ghi rõ phần API aggregation chỉ đọc lướt ở đây và quay lại ở giai đoạn 14." Đây chính là chỗ
quay lại. Bài chỉ dài hơn 20 dòng — toàn bộ khái niệm nằm trong mục *Tầng tổng hợp*.

**Phải hiểu ở lần đọc này:**

- Tầng tổng hợp **chạy trong cùng tiến trình (in-process) với kube-apiserver**, không phải một
  thành phần riêng bạn phải cài. Và cho tới khi có một extension resource được đăng ký, nó
  **không làm gì cả**.
- Cơ chế đăng ký là đối tượng **APIService**: nó "chiếm giữ" một đường dẫn URL trong Kubernetes
  API (ví dụ `/apis/myextension.mycompany.io/v1/…`), và từ đó tầng tổng hợp **proxy mọi thứ**
  gửi tới đường dẫn đó sang APIService đã đăng ký.
- Cách hiện thực phổ biến nhất: chạy một **extension API server trong các Pod bên trong cluster**,
  thường đi kèm **một hoặc nhiều controller**.
- Ranh giới với CRD, nêu ngay ở câu thứ hai của bài: CRD là cách **giúp kube-apiserver nhận biết
  các kind đối tượng mới**; tầng tổng hợp thì **chuyển tiếp request sang một server khác**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Độ trễ phản hồi* — round-trip discovery trong năm giây | là ràng buộc vận hành, chỉ chạm tới khi đã có extension API server thật | Lab 14 |
| Thư viện apiserver-builder và bộ khung extension API server | là công việc lập trình, không phải khái niệm | Lab 14 |
| *Cấu hình tầng tổng hợp*, *thiết lập một extension api-server* (mục Tiếp theo) | là thao tác dựng, cần cluster có extension thật | Lab 14 |
| *Khái niệm Declarative Validation* | là cơ chế nội bộ, bài nói rõ nó mới hỗ trợ validation cho extension API server **trong tương lai** | không cần |

---

Tầng tổng hợp (aggregation layer) cho phép mở rộng Kubernetes bằng các API bổ sung, vượt ra ngoài những gì các API lõi của Kubernetes cung cấp. Các API bổ sung này có thể là những giải pháp có sẵn như [metrics server](https://github.com/kubernetes-sigs/metrics-server), hoặc là các API do chính bạn phát triển.

Tầng tổng hợp khác với [CustomResourceDefinition](179-custom-resources-vi.md) — vốn là cách để giúp kube-apiserver nhận biết các loại (kind) đối tượng mới.

## Tầng tổng hợp (Aggregation layer)

Tầng tổng hợp chạy trong cùng tiến trình (in-process) với kube-apiserver. Cho tới khi có một extension resource được đăng ký, tầng tổng hợp sẽ không làm gì cả. Để đăng ký một API, bạn thêm một đối tượng _APIService_ — đối tượng này "chiếm giữ" (claim) một đường dẫn URL trong Kubernetes API. Kể từ thời điểm đó, tầng tổng hợp sẽ chuyển tiếp (proxy) mọi thứ được gửi tới đường dẫn API đó (ví dụ `/apis/myextension.mycompany.io/v1/…`) tới APIService đã đăng ký.

Cách phổ biến nhất để hiện thực một APIService là chạy một *extension API server* trong các Pod chạy bên trong cluster của bạn. Nếu bạn dùng extension API server để quản lý các resource trong cluster, thì extension API server (còn được viết là "extension-apiserver") thường đi kèm với một hoặc nhiều controller. Thư viện apiserver-builder cung cấp bộ khung (skeleton) cho cả extension API server lẫn các controller đi kèm.

### Độ trễ phản hồi (Response latency)

Extension API server nên có kết nối mạng độ trễ thấp tới và từ kube-apiserver. Các request discovery bắt buộc phải hoàn tất một vòng khứ hồi (round-trip) từ kube-apiserver trong vòng năm giây hoặc ít hơn.

Nếu extension API server của bạn không đạt được yêu cầu độ trễ đó, hãy cân nhắc thực hiện các thay đổi để đáp ứng được yêu cầu này.

## Tiếp theo (What's next)

* Để bộ tổng hợp (aggregator) hoạt động trong môi trường của bạn, hãy [cấu hình tầng tổng hợp](374-configure-aggregation-layer-vi.md).
* Sau đó, [thiết lập một extension api-server](380-setup-extension-api-server-vi.md) để làm việc với tầng tổng hợp.
* Đọc về [APIService](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/api-service-v1/) trong tài liệu tham chiếu API.
* Tìm hiểu về [Khái niệm Declarative Validation](https://kubernetes.io/docs/reference/using-api/declarative-validation/), một cơ chế nội bộ để định nghĩa các luật kiểm tra hợp lệ (validation) mà trong tương lai sẽ hỗ trợ việc kiểm tra hợp lệ cho quá trình phát triển extension API server.

Ngoài ra: tìm hiểu cách mở rộng Kubernetes API bằng [CustomResourceDefinition](378-custom-resource-definitions-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 14:

1. Tầng tổng hợp chạy ở đâu — một tiến trình riêng bạn phải cài, hay bên trong chính
   kube-apiserver? Trên một cluster chưa đăng ký extension nào, nó tiêu tốn gì?
2. Ở [Lab 1a](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) phần B5.3, bạn đã chạy
   `kubectl get apiservices` trên cluster v1.35.6 và lab ghi "APIService là điểm quan sát
   aggregation layer". Theo bài này, một APIService phải có thêm điều gì thì kube-apiserver mới
   thực sự chuyển request đi nơi khác?
3. Câu bẫy: CRD và tầng tổng hợp cùng "mở rộng Kubernetes API". Bài phân biệt hai thứ đó bằng
   đúng một câu ngay đầu — câu đó nói gì?
4. metrics-server mà bạn sẽ cài ở Lab 11a được bài nhắc tới như ví dụ cho loại nào? Vì sao nó
   không thể là một CRD?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Trong cùng tiến trình (in-process) với kube-apiserver** — không có gì để cài thêm, nó có
   sẵn. Và **nó không tiêu tốn gì**: "Cho tới khi có một extension resource được đăng ký, tầng
   tổng hợp sẽ không làm gì cả." Đây là lý do bạn không "bật" aggregation layer; bạn chỉ đăng ký
   thứ cho nó chuyển tiếp.
2. Phải có **một extension API server thật đứng sau đường dẫn mà APIService chiếm giữ**. Cơ chế
   bài mô tả gồm hai vế: APIService "chiếm giữ một đường dẫn URL trong Kubernetes API", và
   **kể từ thời điểm đó** tầng tổng hợp proxy mọi thứ gửi tới đường dẫn ấy sang APIService đã
   đăng ký — mà cách hiện thực phổ biến nhất là chạy extension API server trong các Pod bên
   trong cluster. Chừng nào chưa có extension resource nào được đăng ký, danh sách bạn thấy
   không tạo ra đường proxy nào.
3. "Tầng tổng hợp **khác với CustomResourceDefinition** — vốn là cách để giúp **kube-apiserver
   nhận biết các loại (kind) đối tượng mới**." Nói gọn: CRD làm kube-apiserver **tự phục vụ**
   thêm kind mới; tầng tổng hợp làm kube-apiserver **chuyển tiếp** sang một server khác phục vụ.
   Trực giác "cả hai đều là mở rộng API nên chắc giống nhau" sai ở chỗ đó: khác nhau ở **ai
   phục vụ và lưu trữ**, không phải ở cú pháp người dùng nhìn thấy.
4. metrics-server là ví dụ bài nêu cho **API bổ sung có sẵn chạy qua tầng tổng hợp** ("có thể là
   những giải pháp có sẵn như metrics server"). Nó không thể là CRD vì CRD chỉ giúp
   kube-apiserver **nhận biết và lưu trữ** kind mới, còn metric là dữ liệu được **tính ra tại
   thời điểm hỏi**, do một server riêng phục vụ — đúng trường hợp cần một extension API server
   đứng sau APIService.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

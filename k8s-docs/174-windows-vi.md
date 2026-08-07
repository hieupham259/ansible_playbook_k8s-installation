# Windows trong Kubernetes (Windows in Kubernetes)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/windows/
>
> Kubernetes hỗ trợ các node chạy Microsoft Windows.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 15](LO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows),
bài 1/7 · Kiểm chứng ở Lab 15 (tùy chọn, chưa viết, xem
[bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Lộ trình ghi rõ: bỏ qua hoàn toàn giai đoạn 15 nếu cluster của bạn chỉ có Linux.** Cluster lab
ba VM Ubuntu của bạn **không có node Windows**, và Lab 15 cần thêm một VM Windows Server nên
được đánh dấu tùy chọn. Bảy bài này chỉ đáng đọc khi môi trường thật của bạn có, hoặc sắp có,
worker node Windows.

Bài 1 là **trang mục lục** của nhánh Windows, dài chưa tới một màn hình. Đọc trong hai phút để
biết có những nhóm chủ đề nào, rồi quyết định đọc tiếp hay dừng.

**Phải hiểu ở lần đọc này:**

- Kubernetes hỗ trợ Windows **ở vai trò worker node**, không phải control plane. Bạn thêm Windows
  server làm worker node vào một cluster Kubernetes.
- **Cài `kubectl` trên Windows và có node Windows trong cluster là hai chuyện hoàn toàn khác
  nhau.** Bài nói bạn cài kubectl trên Windows được "bất kể bạn sử dụng hệ điều hành nào bên
  trong cluster".
- Nếu môi trường thực sự có node Windows, bài chỉ ra sáu nhóm chủ đề phải đọc: **mạng, lưu trữ,
  quản lý tài nguyên, bảo mật**, cùng các tác vụ **RunAsUserName / HostProcess Pod / GMSA** và
  **gỡ lỗi**. Bốn nhóm đầu chính là bài 4–7 của giai đoạn này.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Các link tác vụ: RunAsUserName, HostProcess Pod, GMSA, mẹo gỡ lỗi trên Windows | là thao tác trên node Windows thật | khi môi trường thực sự có node Windows |
| Ghi chú về CNCF và cách tiếp cận trung lập với nhà cung cấp | là tuyên bố chính sách nội dung, không phải kỹ thuật | không cần |

---

Kubernetes hỗ trợ các worker node chạy Linux hoặc Microsoft Windows.

> **Ghi chú:** Nội dung trên trang này đề cập đến một sản phẩm hoặc dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những sản phẩm hoặc dự án bên thứ ba đó. Xem [nguyên tắc website của CNCF](https://github.com/cncf/foundation/blob/main/website-guidelines.md) để biết thêm chi tiết.

CNCF và tổ chức mẹ của nó là Linux Foundation có cách tiếp cận trung lập với nhà cung cấp (vendor-neutral) đối với khả năng tương thích. Bạn hoàn toàn có thể thêm [Windows server](https://www.microsoft.com/en-us/windows-server) của mình làm worker node vào một cluster Kubernetes.

Bạn có thể [cài đặt và thiết lập kubectl trên Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/) bất kể bạn sử dụng hệ điều hành nào bên trong cluster.

Nếu bạn đang sử dụng node Windows, bạn có thể đọc:

* [Mạng trên Windows](89-windows-networking-vi.md)
* [Lưu trữ Windows trong Kubernetes](106-windows-storage-vi.md)
* [Quản lý tài nguyên cho node Windows](112-windows-resource-management-vi.md)
* [Cấu hình RunAsUserName cho Pod và container Windows](https://kubernetes.io/docs/tasks/configure-pod-container/configure-runasusername/)
* [Tạo một Windows HostProcess Pod](https://kubernetes.io/docs/tasks/configure-pod-container/create-hostprocess-pod/)
* [Cấu hình Group Managed Service Accounts cho Pod và container Windows](https://kubernetes.io/docs/tasks/configure-pod-container/configure-gmsa/)
* [Bảo mật cho node Windows](131-windows-security-vi.md)
* [Mẹo gỡ lỗi trên Windows](https://kubernetes.io/docs/tasks/debug/debug-cluster/windows/)
* [Hướng dẫn lập lịch Windows container trong Kubernetes](https://kubernetes.io/docs/concepts/windows/user-guide)

hoặc, để có cái nhìn tổng quan, hãy đọc:

* [Windows containers trong Kubernetes](175-windows-intro-vi.md)
* [Hướng dẫn lập lịch Windows container trong Kubernetes](https://kubernetes.io/docs/concepts/windows/user-guide/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 15:

1. Câu bẫy: bạn ngồi trên laptop Windows và chạy `kubectl` từ đó vào cluster lab ba VM Ubuntu.
   Như vậy cluster của bạn đã là cluster "có Windows" chưa? Bài phân biệt hai chuyện đó ở câu
   nào?
2. Bài nói Kubernetes hỗ trợ Windows ở vai trò nào trong cluster? Vai trò nào **không** được nêu?
3. Nếu ngày mai công ty thêm một Windows Server vào cluster, theo trang mục lục này bạn phải đọc
   thêm những nhóm chủ đề nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Chưa.** Bài tách bạch hai chuyện bằng câu: "Bạn có thể **cài đặt và thiết lập kubectl trên
   Windows** bất kể bạn sử dụng hệ điều hành nào **bên trong cluster**." `kubectl` chỉ là client
   nói chuyện với API server qua HTTPS; hệ điều hành của máy chạy client không liên quan gì tới
   hệ điều hành của các node. Cluster chỉ "có Windows" khi có **worker node chạy Windows Server**
   đã join vào.
2. Bài nói Kubernetes hỗ trợ các **worker node** chạy Linux hoặc Microsoft Windows, và bạn "thêm
   Windows server của mình **làm worker node** vào một cluster Kubernetes". Vai trò **control
   plane trên Windows không được nêu** — cả trang chỉ nói về worker node.
3. Bốn nhóm khái niệm có bản dịch trong thư mục: [Mạng trên Windows](89-windows-networking-vi.md),
   [Lưu trữ Windows](106-windows-storage-vi.md),
   [Quản lý tài nguyên cho node Windows](112-windows-resource-management-vi.md) và
   [Bảo mật cho node Windows](131-windows-security-vi.md); cộng thêm ba nhóm tác vụ — cấu hình
   `RunAsUserName`, tạo HostProcess Pod, cấu hình GMSA — và trang mẹo gỡ lỗi. Muốn có cái nhìn
   tổng quan trước thì đọc [Windows containers trong Kubernetes](175-windows-intro-vi.md), đúng
   là bài kế tiếp của giai đoạn này.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

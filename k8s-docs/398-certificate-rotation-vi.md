# Cấu hình xoay vòng certificate cho kubelet (Configure Certificate Rotation for the Kubelet)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/tls/certificate-rotation/>
>
> Trang này hướng dẫn cách bật và cấu hình việc xoay vòng certificate (certificate rotation)
> cho kubelet.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Checkpoint tiếp nối — nhánh `/docs/tasks/`](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 18 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ), bài 3/7 · Kiểm chứng
trên cluster lab: chạy `kubectl get csr` trên `k8s-master` và đọc được trạng thái CSR của kubelet ba
node.

Bài rất ngắn nhưng giải thích đúng cơ chế khiến cluster lab của bạn **không chết vì certificate
hết hạn** sau một năm. Nối tiếp bài [219](219-kubeadm-certs-vi.md) và [23](23-nodes-vi.md).

**Phải hiểu ở lần đọc này:**

- Certificate của kubelet mặc định có thời hạn **một năm**, và Kubernetes có cơ chế tự sinh key
  mới rồi xin certificate mới khi sắp hết hạn — không cần người can thiệp.
- Hai cờ điều khiển nằm ở **hai tiến trình khác nhau**: `--rotate-certificates` trên **kubelet**
  quyết định *có tự xin mới hay không*; `--cluster-signing-duration` trên
  **kube-controller-manager** quyết định *certificate cấp ra sống bao lâu*.
- Vòng đời một CSR: kubelet bootstrap bằng `--bootstrap-kubeconfig` → gửi CSR → trạng thái
  `Pending` → controller manager **tự phê duyệt** nếu thỏa tiêu chí → `Approved` → ký và gắn
  certificate vào CSR → kubelet lấy về, ghi vào `--cert-dir`.
- Thời điểm xin gia hạn **không cố định**: kubelet gửi CSR mới khi certificate còn khoảng
  **30% đến 10%** thời hạn. Đây là lý do không nên trông đợi một mốc thời gian chính xác.
- Sau khi có certificate mới, kubelet **cập nhật các kết nối đang có** để dùng certificate mới,
  không cần restart.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chi tiết cơ chế kubelet TLS bootstrapping | thuộc nhánh `/docs/reference/`, chưa có bản dịch | khi cần dựng node thủ công ngoài kubeadm |
| Tên cũ `--experimental-cluster-signing-duration` | chỉ gặp trên cluster trước 1.19 | khi tiếp quản cluster đời cũ |

---

Trang này hướng dẫn cách bật và cấu hình việc xoay vòng certificate (certificate rotation)
cho kubelet.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.19 [stable]`

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần Kubernetes phiên bản 1.8.0 trở lên.

## Tổng quan (Overview)

kubelet dùng certificate để xác thực với Kubernetes API. Theo mặc định, các certificate này
được cấp với thời hạn một năm, để không phải gia hạn quá thường xuyên.

Kubernetes có sẵn cơ chế
[xoay vòng certificate của kubelet](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/),
cơ chế này tự động sinh key mới và yêu cầu Kubernetes API cấp certificate mới khi certificate
hiện tại sắp hết hạn. Ngay khi certificate mới sẵn sàng, nó sẽ được dùng để xác thực cho các
kết nối tới Kubernetes API.

## Bật xoay vòng client certificate (Enabling client certificate rotation) {#enabling-client-certificate-rotation}

Tiến trình `kubelet` nhận tham số `--rotate-certificates`; tham số này quyết định kubelet có
tự động yêu cầu certificate mới hay không khi certificate đang dùng sắp hết hạn.

Tiến trình `kube-controller-manager` nhận tham số `--cluster-signing-duration`
(trước phiên bản 1.19 là `--experimental-cluster-signing-duration`); tham số này quyết định
certificate được cấp với thời hạn bao lâu.

## Hiểu về cấu hình xoay vòng certificate (Understanding the certificate rotation configuration)

Khi kubelet khởi động, nếu nó được cấu hình để bootstrap (dùng cờ `--bootstrap-kubeconfig`),
nó sẽ dùng certificate ban đầu của mình để kết nối tới Kubernetes API và gửi lên một
certificate signing request (CSR). Bạn có thể xem trạng thái của các certificate signing
request bằng lệnh:

```sh
kubectl get csr
```

Ban đầu, certificate signing request do kubelet trên một node gửi lên sẽ có trạng thái
`Pending`. Nếu certificate signing request đó thỏa mãn những tiêu chí nhất định, nó sẽ được
controller manager tự động phê duyệt (auto-approve) và chuyển sang trạng thái `Approved`.
Tiếp theo, controller manager sẽ ký certificate, cấp với thời hạn được chỉ định bởi tham số
`--cluster-signing-duration`, rồi gắn certificate đã ký vào certificate signing request đó.

kubelet sẽ lấy certificate đã ký từ Kubernetes API và ghi xuống đĩa, tại vị trí được chỉ định
bởi `--cert-dir`. Sau đó kubelet dùng certificate mới này để kết nối tới Kubernetes API.

Khi certificate đã ký sắp hết hạn, kubelet sẽ tự động gửi lên một certificate signing request
mới thông qua Kubernetes API. Việc này có thể xảy ra tại bất kỳ thời điểm nào trong khoảng từ
30% đến 10% thời gian còn lại của certificate. Một lần nữa, controller manager sẽ tự động phê
duyệt yêu cầu cấp certificate và gắn certificate đã ký vào certificate signing request.
kubelet sẽ lấy certificate mới đã ký từ Kubernetes API và ghi xuống đĩa. Sau đó nó cập nhật
các kết nối đang có tới Kubernetes API để kết nối lại bằng certificate mới.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 18:

1. Trên cluster lab, `kubectl get csr` cho thấy CSR của kubelet ở trạng thái `Approved,Issued`.
   Ai đã phê duyệt nó, và ai đã ký?
2. **Câu bẫy.** Bạn muốn certificate kubelet sống lâu hơn. Đặt cờ nào, **trên tiến trình nào**?
   `--rotate-certificates` có làm được việc đó không?
3. Certificate kubelet còn hạn 100 ngày. Khi nào kubelet gửi CSR gia hạn — nêu khoảng, và vì sao
   bài không cho một con số chính xác?
4. Sau khi nhận certificate mới, kubelet có phải khởi động lại để dùng nó không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **kube-controller-manager làm cả hai việc.** Nó tự phê duyệt (auto-approve) CSR nếu CSR thỏa
   các tiêu chí nhất định, rồi chính nó ký certificate với thời hạn lấy từ
   `--cluster-signing-duration` và gắn certificate đã ký vào CSR. kubelet chỉ **gửi** CSR và
   **lấy về** kết quả.
2. Đặt **`--cluster-signing-duration`** trên **kube-controller-manager**. **`--rotate-certificates`
   không làm được** — nó chỉ bật/tắt việc kubelet *có tự xin certificate mới hay không*, không
   quyết định thời hạn. Đây là chỗ dễ nhầm vì cả hai cờ đều liên quan tới "xoay certificate",
   nhưng chúng nằm trên hai tiến trình khác nhau và trả lời hai câu hỏi khác nhau.
3. Trong khoảng còn **30% đến 10%** thời hạn — với 100 ngày thì rơi vào lúc còn khoảng 30 đến 10
   ngày. Bài cho một khoảng chứ không phải một mốc để tránh việc toàn bộ kubelet trong cluster
   cùng gửi CSR tại đúng một thời điểm.
4. **Không.** kubelet lấy certificate mới từ API, ghi xuống `--cert-dir`, rồi **cập nhật các kết
   nối đang có** để kết nối lại bằng certificate mới.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi sang bài kế của [Giai đoạn 18 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ).

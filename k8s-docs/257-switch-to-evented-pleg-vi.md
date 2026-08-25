# Chuyển từ polling sang cập nhật trạng thái container dựa trên sự kiện CRI (Switching from Polling to CRI Event-based Updates to Container Status)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/switch-to-evented-pleg/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 2 — Container và runtime](00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime),
bài 2/2 — bài **cuối** của dòng **Thực hành**, làm ngay trước khi mở lab ·
[Lab 2 — Container, image, CRI và cgroup](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md) **không
thực hành bài này**: bảng ở mục 1.1 của lab ghi rõ lý do — đổi feature gate của kubelet làm lệch
baseline Lab 00, việc đó thuộc
[Giai đoạn 20 — Cấu hình lại cluster đang chạy](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy).
Phần gần nhất bạn quan sát được trên cluster lab là **B1** của Lab 2, nơi bạn tự chứng minh
kubelet là client còn containerd là server của CRI.

Bài này là mặt sau của bài [44 — Container Runtime Interface (CRI)](44-cri-vi.md): 44 nói kubelet
và runtime nói chuyện bằng gì, còn bài này nói kubelet **hỏi** hay **được báo**. Đọc để hiểu cơ
chế, không phải để bật trên cluster lab.

**Phải hiểu ở lần đọc này:**

- Hai cơ chế đối lập: _generic PLEG_ là kubelet **polling** trạng thái container theo chu kỳ;
  _Evented PLEG_ là kubelet **nhận sự kiện** vòng đời container do runtime đẩy lên qua CRI. Mục
  tiêu của cơ chế mới là **bớt việc vô ích trong lúc node không có hoạt động**, từ đó giảm tài
  nguyên node mà kubelet tiêu thụ.
- Vì sao polling đắt, theo mục *Tại sao nên chuyển sang Evented PLEG?*: việc polling diễn ra
  **thường xuyên**, và kubelet polling trạng thái các container **song song** — điều này giới hạn
  khả năng mở rộng và kéo theo vấn đề về hiệu năng lẫn độ tin cậy.
- Điều kiện phải đúng ở **cả hai phía**: feature gate `EventedPLEG` bật trên kubelet, **và**
  container runtime công bố hỗ trợ sinh sự kiện vòng đời container (containerd 1.7+, CRI-O
  1.26+). Nếu runtime không công bố hỗ trợ, kubelet **tự động quay về generic PLEG** dù bạn đã
  bật gate.
- Đây là thao tác **theo từng node**, không phải một công tắc cấp cluster: sửa
  [file cấu hình kubelet](224-kubelet-config-file-vi.md) rồi khởi động lại dịch vụ kubelet, làm
  **trên từng node** bạn muốn dùng tính năng; bài còn yêu cầu **drain node** trước khi đi tiếp,
  và chỉ sau đó mới khởi động runtime với tính năng sinh sự kiện.
- Cách xác minh, ở bước 4: tìm chuỗi `EventedPLEG` trong log của kubelet. Đặt `--v` từ 4 trở lên
  thì thấy thêm các dòng `Evented PLEG: Generated pod status from the received event` — đó mới là
  bằng chứng cơ chế mới **đang chạy**, không chỉ **đã được bật**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Phần cấu hình riêng cho CRI-O (`crio config`, khóa `enable_pod_events`, cờ `--enable-pod-events=true`, file drop-in TOML) | cluster lab chạy containerd, không có CRI-O | bài [00 — Các container runtime](00-container-runtimes-vi.md) của chính giai đoạn 2, và **B1** của [Lab 2](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md) khi bạn xác định runtime thật của node |
| Thao tác thật: sửa feature gate rồi khởi động lại kubelet trên một node | làm lệch baseline đã khóa ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) | [Giai đoạn 20 — Cấu hình lại cluster đang chạy](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), bài [224 — Thiết lập tham số kubelet qua file cấu hình](224-kubelet-config-file-vi.md) |
| Bước drain node ở mục *Chuyển sang Evented PLEG* | quy trình drain là một bài riêng | bài [255 — Drain một node an toàn](255-safely-drain-node-vi.md) ở [Giai đoạn 16 — Vòng đời node](00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node) |
| KEP *Kubelet Evented PLEG for Better Performance* ở mục *Tiếp theo* | tài liệu thiết kế, không cần để vận hành | ngoài phạm vi lộ trình — đọc khi cần lý do thiết kế đằng sau tính năng |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [alpha]`

Trang này hướng dẫn cách chuyển các node sang sử dụng cơ chế cập nhật trạng thái container dựa trên
sự kiện (event-based updates). Cách triển khai dựa trên sự kiện giúp giảm mức tiêu thụ tài nguyên
node của kubelet, so với cách tiếp cận cũ dựa vào polling (thăm dò định kỳ).
Bạn có thể biết đến tính năng này với tên gọi _evented Pod lifecycle event generator (PLEG)_ —
bộ sinh sự kiện vòng đời Pod theo cơ chế sự kiện. Đó là tên được dùng nội bộ trong dự án Kubernetes
cho một chi tiết triển khai quan trọng.

Cách tiếp cận dựa trên polling được gọi là _generic PLEG_.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần chạy một phiên bản Kubernetes có cung cấp tính năng này.
  Kubernetes v1.27 bao gồm hỗ trợ beta cho cập nhật trạng thái container
  dựa trên sự kiện. Tính năng này ở trạng thái beta nhưng bị _tắt_ theo mặc định
  vì nó cần sự hỗ trợ từ phía container runtime.
* Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn phiên bản 1.26.
  Để kiểm tra phiên bản, nhập `kubectl version`.
  Nếu bạn đang chạy một phiên bản Kubernetes khác, hãy xem tài liệu của bản phát hành đó.
* Container runtime đang dùng phải hỗ trợ các sự kiện vòng đời container (container lifecycle
  events). Kubelet sẽ tự động chuyển về cơ chế generic PLEG cũ nếu container runtime
  không công bố hỗ trợ các sự kiện vòng đời container, ngay cả khi bạn đã bật
  feature gate này.

## Tại sao nên chuyển sang Evented PLEG? (Why switch to Evented PLEG?)

* _Generic PLEG_ gây ra chi phí (overhead) không hề nhỏ do việc polling trạng thái container diễn
  ra thường xuyên.
* Chi phí này càng trầm trọng hơn do kubelet polling song song trạng thái của các container, từ đó
  hạn chế khả năng mở rộng (scalability) của nó và gây ra các vấn đề về hiệu năng kém cũng như độ
  tin cậy.
* Mục tiêu của _Evented PLEG_ là giảm bớt công việc không cần thiết trong thời gian không có hoạt
  động, bằng cách thay thế việc polling định kỳ.

## Chuyển sang Evented PLEG (Switching to Evented PLEG)

1. Khởi động kubelet với [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
   `EventedPLEG` được bật. Bạn có thể quản lý các feature gate của kubelet bằng cách sửa
   [file cấu hình](224-kubelet-config-file-vi.md) của kubelet
   và khởi động lại dịch vụ kubelet. Bạn cần làm việc này trên từng node mà bạn muốn sử dụng
   tính năng này.

2. Đảm bảo node đã được [drain](255-safely-drain-node-vi.md)
   trước khi tiếp tục.

3. Khởi động container runtime với tính năng sinh sự kiện container (container event generation)
   được bật.

   #### Containerd

   Phiên bản 1.7+

   #### CRI-O

   Phiên bản 1.26+

   Kiểm tra xem CRI-O đã được cấu hình để phát ra các sự kiện CRI hay chưa bằng cách xác minh
   cấu hình,

   ```shell
   crio config | grep enable_pod_events
   ```

   Nếu đã được bật, output sẽ tương tự như sau:

   ```none
   enable_pod_events = true
   ```

   Để bật nó, khởi động CRI-O daemon với flag `--enable-pod-events=true` hoặc
   dùng một file cấu hình drop-in với các dòng sau:

   ```toml
   [crio.runtime]
   enable_pod_events: true
   ```

   Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn phiên bản 1.26.
   Để kiểm tra phiên bản, nhập `kubectl version`.

4. Xác minh rằng kubelet đang sử dụng cơ chế giám sát thay đổi trạng thái container dựa trên
   sự kiện. Để kiểm tra, hãy tìm cụm từ `EventedPLEG` trong log của kubelet.

   Output sẽ tương tự như sau:

   ```console
   I0314 11:10:13.909915 1105457 feature_gate.go:249] feature gates: &{map[EventedPLEG:true]}
   ```

   Nếu bạn đã đặt `--v` từ 4 trở lên, bạn có thể thấy thêm các mục cho biết
   kubelet đang dùng cơ chế giám sát trạng thái container dựa trên sự kiện.

   ```console
   I0314 11:12:42.009542 1110177 evented.go:238] "Evented PLEG: Generated pod status from the received event" podUID=3b2c6172-b112-447a-ba96-94e7022912dc
   I0314 11:12:44.623326 1110177 evented.go:238] "Evented PLEG: Generated pod status from the received event" podUID=b3fba5ea-a8c5-4b76-8f43-481e17e8ec40
   I0314 11:12:44.714564 1110177 evented.go:238] "Evented PLEG: Generated pod status from the received event" podUID=b3fba5ea-a8c5-4b76-8f43-481e17e8ec40
   ```

## Tiếp theo (What's next)

* Tìm hiểu thêm về thiết kế này trong Kubernetes Enhancement Proposal (KEP):
  [Kubelet Evented PLEG for Better Performance](https://github.com/kubernetes/enhancements/blob/5b258a990adabc2ffdc9d84581ea6ed696f7ce6c/keps/sig-node/3386-kubelet-evented-pleg/README.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 2:

1. Bài nêu hai lý do cụ thể khiến generic PLEG tốn kém. Đó là hai lý do nào, và Evented PLEG
   nhắm vào khoảng thời gian nào của node để tiết kiệm?
2. **Câu bẫy.** Tính năng này ở trạng thái beta từ Kubernetes v1.27. Vậy nó có được bật mặc định
   không, và vì sao?
3. Bạn bật `EventedPLEG` trên `lab-k8s-worker2` — node duy nhất được phép gây lỗi — và kubelet
   khởi động lại bình thường, nhưng trong log không có dòng nào của cơ chế mới. Ngoài giả thiết
   "gate chưa bật", bài nêu nguyên nhân nào khác, và lúc đó kubelet đang chạy cơ chế gì?
4. Kể lại đúng thứ tự bốn bước ở mục *Chuyển sang Evented PLEG*. Bước nào bảo vệ workload đang
   chạy trên node, và bước nào cho bạn bằng chứng chứ không chỉ là cấu hình?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Hai lý do: **việc polling trạng thái container diễn ra thường xuyên**, gây chi phí không hề
   nhỏ; và **kubelet polling trạng thái các container song song**, điều này hạn chế khả năng mở
   rộng và kéo theo hiệu năng kém cùng vấn đề về độ tin cậy. Evented PLEG nhắm vào **khoảng thời
   gian không có hoạt động**: khi không có gì thay đổi thì không có sự kiện nào, nên kubelet
   không phải làm gì — trong khi polling vẫn hỏi đều đặn.
2. **Không** — bài nói rõ tính năng ở trạng thái beta nhưng **bị tắt theo mặc định**. Trực giác
   thông thường "beta thì mặc định bật" hỏng ở đây vì lý do bài đưa ra: tính năng này **cần sự hỗ
   trợ từ phía container runtime**, thứ mà Kubernetes không tự quyết định được. Bật mặc định một
   thứ phụ thuộc thành phần bên ngoài sẽ chỉ tạo ra cấu hình không chạy.
3. Nguyên nhân còn lại là **container runtime không công bố hỗ trợ các sự kiện vòng đời
   container**. Khi đó kubelet **tự động quay về generic PLEG** — tức là vẫn chạy bình thường
   bằng cơ chế **polling** cũ, không báo lỗi, dù feature gate đã bật. Đây chính là lý do bước
   xác minh bằng log tồn tại: bật gate không đồng nghĩa với cơ chế mới đang chạy.
4. Thứ tự bốn bước: **(1)** khởi động kubelet với feature gate `EventedPLEG` bật — sửa file cấu
   hình kubelet rồi khởi động lại dịch vụ, trên từng node muốn dùng; **(2)** đảm bảo node đã được
   **drain**; **(3)** khởi động container runtime với tính năng sinh sự kiện container được bật;
   **(4)** xác minh bằng cách tìm `EventedPLEG` trong log kubelet. Bước **(2)** là bước bảo vệ
   workload, vì node được rút Pod trước khi bạn động vào runtime của nó. Bước **(4)** là bước cho
   bằng chứng: với `--v` từ 4 trở lên, các dòng
   `Evented PLEG: Generated pod status from the received event` chứng minh kubelet **đang thật sự
   sinh trạng thái Pod từ sự kiện nhận được**, chứ không chỉ đọc được một cấu hình đã bật.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của dòng **Thực hành**
giai đoạn 2 — trả lời xong cả bốn câu thì mở
[Lab 2 — Container, image, CRI và cgroup](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md) và bắt đầu
từ phần B0.

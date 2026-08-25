# Chuyển đổi từ dockershim (Migrating from dockershim)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 27 — Di chuyển khỏi dockershim (cluster cũ)](00-ALO-TRINH-ADMIN.md#giai-đoạn-27--di-chuyển-khỏi-dockershim-cluster-cũ),
bài 1/6 · Phần II không có lab riêng: tự chấm bằng **Checkpoint** ghi ở cuối mục giai đoạn 27 trong
lộ trình. Riêng trang này không có gì đo được trên cluster lab; phần đo được nằm ở bài
[239](239-find-out-runtime-vi.md).

**Cả giai đoạn 27 là kiến thức tiếp quản cluster của người khác, không phải bài thực hành của
bạn.** Lộ trình nói thẳng: *bỏ qua toàn bộ giai đoạn này nếu cluster của bạn đã dùng containerd* —
và cluster lab (`lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`) thuộc đúng nhóm đó; bạn đã
tự đọc ra runtime `containerd` ở
[Lab 2 phần B1.1](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md#b11-ai-thực-sự-chạy-container). Nghĩa
là bạn **không có** cluster dockershim để chạy thử, và sẽ không dựng một cái chỉ để tập. Giá trị của
giai đoạn này là ngày bạn nhận bàn giao một cluster đời cũ: nhận ra nó, biết phải kiểm gì trước khi
động vào, và biết trình tự chuyển runtime từng node. Đọc lấy lập luận và trình tự, đừng tìm chỗ gõ
lệnh.

**Phải hiểu ở lần đọc này:**

- Đây là **trang mục nhóm**, không phải runbook: nó dựng bối cảnh (dockershim bị loại bỏ khỏi
  Kubernetes ở bản phát hành v1.24) rồi giao việc cho hai tác vụ con mà mục *Các tác vụ sau sẽ giúp
  bạn thực hiện việc chuyển đổi* liệt kê. Đừng tìm thao tác ở trang này.
- Ranh giới của vấn đề nằm ở cụm **"dùng Docker Engine *thông qua dockershim* làm container
  runtime"**. Đúng nhóm đó mới phải hành động, và trang nêu **hai** hướng: chuyển sang một runtime
  khác, hoặc tìm một phương án thay thế để vẫn có hỗ trợ cho Docker Engine — danh mục lựa chọn nằm ở
  trang [container runtimes](00-container-runtimes-vi.md).
- Một cluster **có thể có nhiều hơn một loại node**, dù đó không phải cấu hình phổ biến. Hệ quả khi
  tiếp quản: kết luận phải rút ra trên **từng node**, không suy từ một node ra cả cluster.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Đoạn về vòng đời hỗ trợ của v1.23 và v1.24, và lời khuyên báo issue lên GitHub | là thông báo của dự án ở thời điểm gỡ dockershim, không phải thao tác vận hành | không cần — cluster lab chạy baseline mới hơn nhiều, xem [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) |
| Hai link tác vụ con ở cuối trang | đó chính là hai bài kế tiếp của giai đoạn này, đọc ở lượt của chúng | bài [238](238-check-dockershim-removal-vi.md) và bài [240](240-migrating-telemetry-agents-vi.md) |
| Danh mục runtime thay thế ở trang [container runtimes](00-container-runtimes-vi.md) | chỉ áp dụng khi tiếp quản cluster dockershim và phải chọn runtime mới; cluster lab đã chốt containerd từ Lab 00 | bài [237](237-change-runtime-containerd-vi.md) của chính giai đoạn 27 |

---

Phần này trình bày những thông tin bạn cần biết khi chuyển đổi (migrate) từ dockershim sang các
container runtime khác.

Kể từ thông báo về việc
[ngừng hỗ trợ dockershim (dockershim deprecation)](https://kubernetes.io/blog/2020/12/08/kubernetes-1-20-release-announcement/#dockershim-deprecation)
trong Kubernetes 1.20, đã có nhiều câu hỏi về việc điều này sẽ ảnh hưởng như thế nào tới các
workload và các bản cài đặt Kubernetes khác nhau. Bài
[Dockershim Removal FAQ](https://kubernetes.io/blog/2022/02/17/dockershim-faq/) sẽ giúp bạn
hiểu rõ hơn về vấn đề này.

Dockershim đã bị loại bỏ khỏi Kubernetes cùng với bản phát hành v1.24.
Nếu bạn đang dùng Docker Engine thông qua dockershim làm container runtime và muốn nâng cấp lên
v1.24, bạn nên chuyển sang một runtime khác hoặc tìm một phương án thay thế để có được sự hỗ trợ
cho Docker Engine. Hãy xem mục
[container runtimes](00-container-runtimes-vi.md)
để biết các lựa chọn của bạn.

Phiên bản Kubernetes còn có dockershim (1.23) đã hết hạn hỗ trợ và v1.24 cũng sẽ
[sớm](https://kubernetes.io/releases/#release-v1-24) hết hạn hỗ trợ. Hãy nhớ
[báo cáo các vấn đề (issue)](https://github.com/kubernetes/kubernetes/issues) mà bạn gặp phải
trong quá trình chuyển đổi, để các vấn đề đó có thể được sửa kịp thời và cluster của bạn sẵn
sàng cho việc loại bỏ dockershim. Sau khi v1.24 hết hạn hỗ trợ, bạn sẽ phải liên hệ nhà cung cấp
Kubernetes của mình để được hỗ trợ, hoặc nâng cấp nhiều phiên bản cùng lúc nếu có các vấn đề
nghiêm trọng ảnh hưởng tới cluster của bạn.

Cluster của bạn có thể có nhiều hơn một loại node, mặc dù đây không phải là cấu hình phổ biến.

Các tác vụ sau sẽ giúp bạn thực hiện việc chuyển đổi:

* [Kiểm tra xem việc loại bỏ Dockershim có ảnh hưởng tới bạn không](238-check-dockershim-removal-vi.md)
* [Chuyển đổi các agent telemetry và bảo mật khỏi dockershim](240-migrating-telemetry-agents-vi.md)

## Tiếp theo (What's next)

* Xem mục [container runtimes](00-container-runtimes-vi.md)
  để hiểu các lựa chọn thay thế của bạn.
* Nếu bạn phát hiện lỗi hoặc vấn đề kỹ thuật khác liên quan tới việc chuyển đổi khỏi dockershim,
  bạn có thể [báo cáo issue](https://github.com/kubernetes/kubernetes/issues/new/choose)
  cho dự án Kubernetes.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 27. Cả ba câu trả lời
được bằng lập luận, không cần cluster dockershim:

1. Trang này giao việc cho đúng hai tác vụ con. Đó là hai tác vụ nào, mỗi tác vụ trả lời câu hỏi gì,
   và vì sao trang này **không** chứa bước thao tác nào?
2. **Câu bẫy.** "Dockershim bị loại bỏ ở v1.24" — vậy mọi cluster có cài Docker trên node đều phải
   đổi runtime trước khi lên v1.24, đúng không? Và người còn muốn giữ Docker Engine thì hết đường?
3. Ba node lab của bạn — `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2` — đều chạy
   containerd, nên lộ trình cho phép bỏ qua giai đoạn 27. Giả sử bạn được bàn giao một cluster lạ:
   theo trang này, vì sao kiểm một node rồi kết luận cho cả cluster là sai?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **[Kiểm tra xem việc loại bỏ dockershim có ảnh hưởng tới bạn không](238-check-dockershim-removal-vi.md)
   và [Chuyển đổi các agent telemetry và bảo mật khỏi dockershim](240-migrating-telemetry-agents-vi.md)**
   — một cái trả lời "tôi có bị ảnh hưởng không", một cái trả lời "những agent đang bám vào Docker
   thì xử lý thế nào". Trang này **là trang mục nhóm**: nhiệm vụ của nó là dựng bối cảnh và chỉ
   đường, nên toàn bộ thao tác nằm ở các trang con.
2. **Không, cả hai vế đều sai.** Điều kiện mà trang nêu là dùng Docker Engine **thông qua
   dockershim làm container runtime** — chính cụm "thông qua dockershim" là ranh giới, chứ không
   phải sự có mặt của Docker trên node. Vế sau cũng sai: trang nêu **hai** hướng, không phải một —
   chuyển sang một runtime khác, **hoặc** tìm một phương án thay thế để vẫn có hỗ trợ cho Docker
   Engine. Trang [container runtimes](00-container-runtimes-vi.md) là nơi liệt kê các lựa chọn đó.
3. Vì trang nói rõ **cluster của bạn có thể có nhiều hơn một loại node**, dù đây không phải cấu
   hình phổ biến. Node được thêm ở những đợt khác nhau có thể được dựng bằng runtime khác nhau, nên
   kết luận phải rút ra **trên từng node**. Đây cũng là lý do bài [239](239-find-out-runtime-vi.md)
   bắt đầu bằng một lệnh liệt kê **tất cả** các node chứ không phải một node.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

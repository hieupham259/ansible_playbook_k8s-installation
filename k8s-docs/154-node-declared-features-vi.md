# Tính năng do Node khai báo (Node Declared Features)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/node-declared-features/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7b](00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên),
bài 6/6 · Kiểm chứng ở [Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md).

**Lộ trình đánh dấu bài này là ĐỌC LƯỚT.** Đọc một lượt là đủ, không cần dừng lại nghiền ngẫm,
và không có gì phải thực hành. Ba lý do: đây là tính năng beta từ v1.36 nên **cluster baseline
v1.35.6 của bạn chưa có nó**; bài nói thẳng cơ chế này **dành cho nhà phát triển tính năng của
Kubernetes** và hoạt động ngầm phía sau, còn người triển khai Pod **không cần tương tác trực
tiếp**; và toàn bộ trang chỉ có hơn 50 dòng, phần lớn là trỏ ra KEP. Mục tiêu duy nhất của lần
đọc này là **nhận ra cái tên** khi sau này gặp `.status.declaredFeatures` trong `kubectl
describe node` hoặc gặp một Pod `Pending` khó hiểu trên cluster đang nâng cấp dở.

**Phải hiểu ở lần đọc này:**

- Cơ chế giải quyết vấn đề gì: **chênh lệch phiên bản (version skew)** giữa các node, đặc biệt
  trong lúc nâng cấp cluster hoặc ở môi trường mixed-version, nơi không phải node nào cũng bật
  cùng một tập tính năng.
- Nó là framework **nội bộ**: kubelet phát hiện và báo cáo, control plane tiêu thụ. Người viết
  Pod không khai gì thêm trong spec.
- Hai chỗ nó can thiệp: **kube-scheduler** qua plugin `NodeDeclaredFeatures` (mặc định bật) loại
  bỏ node thiếu tính năng Pod cần, và **admission controller** `NodeDeclaredFeatureValidator`
  từ chối những Pod yêu cầu tính năng chưa được node mà chúng gắn vào khai báo.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chi tiết plugin ở giai đoạn `PreFilter` và `Filter`, và cách scheduler tùy chỉnh dùng `.status.declaredFeatures` | ở đây chỉ cần biết plugin can thiệp vào việc lọc node, không cần biết cách viết plugin | giai đoạn 14, bài [177](177-extend-kubernetes-vi.md) |
| Admission controller `NodeDeclaredFeatureValidator` | chưa học chuỗi ba chặng authn → authz → admission | giai đoạn 9, bài [119](119-controlling-access-vi.md) |
| *Bật tính năng do node khai báo* — feature gate trên `kube-apiserver`, `kube-scheduler`, `kubelet` | bạn chưa tự đặt cờ cho các thành phần control plane | giai đoạn 8, bài [03](03-control-plane-flags-vi.md) |
| KEP-5328 trong mục *Tiếp theo* | là tài liệu thiết kế | không cần |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Các node trong Kubernetes sử dụng _tính năng được khai báo_ (declared features) để báo cáo
tính khả dụng của những tính năng cụ thể còn mới hoặc đang được kiểm soát bằng feature gate.
Các thành phần của control plane tận dụng thông tin này để đưa ra quyết định tốt hơn.
kube-scheduler, thông qua plugin `NodeDeclaredFeatures`, đảm bảo các Pod chỉ được đặt lên
những node hỗ trợ tường minh các tính năng mà Pod yêu cầu. Ngoài ra, admission controller
`NodeDeclaredFeatureValidator` kiểm tra tính hợp lệ của các cập nhật Pod dựa trên các tính
năng mà node đã khai báo.

Cơ chế này giúp quản lý sự chênh lệch phiên bản (version skew) và cải thiện độ ổn định của
cluster, đặc biệt trong quá trình nâng cấp cluster hoặc trong các môi trường lẫn lộn nhiều
phiên bản (mixed-version), nơi không phải tất cả các node đều bật cùng một tập tính năng.
Cơ chế này dành cho các nhà phát triển tính năng của Kubernetes khi giới thiệu các tính năng
mới ở cấp node và hoạt động ngầm ở phía sau; các nhà phát triển ứng dụng triển khai Pod
không cần tương tác trực tiếp với framework này.

## Cách hoạt động (How it Works)

1.  **Kubelet báo cáo tính năng (Kubelet Feature Reporting):** Khi khởi động, kubelet trên
    mỗi node phát hiện những tính năng Kubernetes được quản lý nào hiện đang được bật và
    báo cáo chúng trong trường `.status.declaredFeatures` của Node. Chỉ những tính năng
    đang trong quá trình phát triển tích cực mới được đưa vào trường này.
2.  **Scheduler lọc node (Scheduler Filtering):** kube-scheduler mặc định sử dụng plugin
    `NodeDeclaredFeatures`. Plugin này:
    * Ở giai đoạn `PreFilter`, kiểm tra `PodSpec` để suy ra tập các tính năng cấp node
      mà Pod yêu cầu.
    * Ở giai đoạn `Filter`, kiểm tra xem các tính năng được liệt kê trong
      `.status.declaredFeatures` của node có thỏa mãn các yêu cầu đã suy ra cho Pod hay
      không. Pod sẽ không được lập lịch (schedule) lên những node thiếu các tính năng cần thiết.
    Các scheduler tùy chỉnh cũng có thể tận dụng trường
    `.status.declaredFeatures` để áp đặt các ràng buộc tương tự.
3.  **Kiểm soát admission (Admission Control):** Admission controller
    `nodedeclaredfeaturevalidator` có thể từ chối những Pod yêu cầu các tính năng chưa được
    khai báo bởi node mà chúng được gán vào (bound), giúp ngăn ngừa sự cố khi cập nhật Pod.

## Bật tính năng do node khai báo (Enabling node declared features)

Để sử dụng Node Declared Features, [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#NodeDeclaredFeatures)
`NodeDeclaredFeatures` phải được bật trên các thành phần `kube-apiserver`,
`kube-scheduler` và `kubelet`.

## Tiếp theo (What's next)

* Đọc KEP để biết thêm chi tiết:
    [KEP-5328: Node Declared Features](https://github.com/kubernetes/enhancements/blob/6d3210f7dd5d547c8f7f6a33af6a09eb45193cd7/keps/sig-node/5328-node-declared-features/README.md)
* Đọc về [admission controller `NodeDeclaredFeatureValidator`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#nodedeclaredfeaturevalidator).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Bài này chỉ đọc lướt, nên ba câu là đủ. Trả lời được mà không nhìn lại bài là xong:

1. Cơ chế này dành cho ai — người viết Pod hay người phát triển tính năng của Kubernetes? Bạn có
   phải khai thêm gì trong Pod spec để nó hoạt động không?
2. Node khai báo tính năng ở trường nào, và hai thành phần nào đọc trường đó, mỗi thành phần làm
   gì với nó?
3. Cluster lab của bạn có một control plane `lab-k8s-master` và hai worker, cả ba cùng Kubernetes
   v1.35.6. Cơ chế này có đang hoạt động trên cluster đó không? Và nó chống lại loại sự cố nào
   mà một cluster đồng nhất phiên bản như của bạn vốn không gặp?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Dành cho nhà phát triển tính năng của Kubernetes, không dành cho bạn.** Bài nói thẳng: cơ
   chế này dành cho các nhà phát triển tính năng khi giới thiệu tính năng mới ở cấp node và
   **hoạt động ngầm ở phía sau**; các nhà phát triển ứng dụng triển khai Pod **không cần tương
   tác trực tiếp với framework này**. Không có trường nào để bạn khai trong Pod spec — scheduler
   **tự suy ra** tập tính năng cấp node mà Pod yêu cầu bằng cách kiểm tra chính `PodSpec` ở giai
   đoạn `PreFilter`.
2. **Kubelet báo cáo vào `.status.declaredFeatures` của Node** khi khởi động, sau khi phát hiện
   những tính năng được quản lý nào đang bật; và chỉ những tính năng **đang trong quá trình phát
   triển tích cực** mới nằm trong đó. Hai bên đọc trường này: **kube-scheduler** qua plugin
   `NodeDeclaredFeatures` — ở `Filter` nó đối chiếu danh sách này với yêu cầu đã suy ra và
   **loại node thiếu tính năng**, khiến Pod không được lập lịch lên đó; và **admission
   controller** `NodeDeclaredFeatureValidator` — **từ chối các cập nhật Pod** yêu cầu tính năng
   chưa được node mà Pod đã gắn vào khai báo.
3. **Không.** Tính năng ở trạng thái `Kubernetes v1.36 [beta]`, còn cluster của bạn là v1.35.6 —
   phiên bản đó chưa có nó, và kể cả khi lên v1.36 thì vẫn phải bật feature gate
   `NodeDeclaredFeatures` trên cả `kube-apiserver`, `kube-scheduler` và `kubelet`. Chỗ dễ nhầm là
   nghĩ đây là một tính năng bạn cần bật để cluster "chặt chẽ hơn": không phải. Nó chống lại
   **chênh lệch phiên bản (version skew)** — tình huống các node **không cùng bật một tập tính
   năng**, xảy ra khi đang nâng cấp cluster dở dang hoặc ở môi trường mixed-version. Cluster ba
   máy cùng v1.35.6 của bạn theo định nghĩa không có sự lệch đó; nó chỉ xuất hiện khi bạn nâng
   cấp từng node một, và đó là lúc cái tên này quay lại.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của nhóm 7b — khi
trả lời được hết sáu bài, bạn sẵn sàng vào [Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md); trong lúc chờ, dùng checkpoint của
[Giai đoạn 7](00-ALO-TRINH-ADMIN.md#giai-đoạn-7--lập-lịch-và-chính-sách-tài-nguyên) để tự kiểm.

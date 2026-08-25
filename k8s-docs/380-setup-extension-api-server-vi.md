# Thiết lập một Extension API Server (Set up an Extension API Server)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/extend-kubernetes/setup-extension-api-server/>
>
> Thiết lập một extension API server để làm việc với aggregation layer cho phép mở rộng
> Kubernetes apiserver bằng các API bổ sung không thuộc nhóm API lõi của Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 28 — Mở rộng Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes), bài 6/11 ·
Phần II không có lab riêng: bạn thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) rồi tự chấm bằng **Checkpoint** của
[giai đoạn 28](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes).

**Trang rất ngắn, và đó là chủ ý.** Nó là **bản đồ 15 bước ở mức tổng quát**, không phải runbook
chạy được: bản thân bài nói rõ "các bước sau mô tả cách thiết lập một extension-apiserver *ở mức
tổng quát*". Điều kiện tiên quyết duy nhất của nó — cấu hình aggregation layer và bật các flag
tương ứng của apiserver — nằm trọn ở bài [374](374-configure-aggregation-layer-vi.md), bài bạn vừa
đọc ở chính giai đoạn này. Đọc bài này để **nhớ được hình dạng của việc**, rồi quay về 374 cho chi
tiết.

**Phải hiểu ở lần đọc này:**

- Đây là danh sách việc, không phải hướng dẫn sao chép: nó giả định bạn **đã có** một image
  extension api-server hoạt động được (bước 7) và **đã** cấu hình xong aggregation layer ở phía
  kube-apiserver (mục *Trước khi bạn bắt đầu*). Thiếu một trong hai thì mười lăm bước này không
  chạy được bước nào.
- Ba binding phân quyền ở bước 11–13 và vì sao thiếu cái nào cũng hỏng: ClusterRoleBinding tới
  ClusterRole **của riêng bạn** (cho các thao tác trên resource bạn định nghĩa),
  ClusterRoleBinding tới `system:auth-delegator` (ủy quyền quyết định xác thực và phân quyền về
  API server lõi), và RoleBinding tới Role `extension-apiserver-authentication-reader` (để đọc
  ConfigMap `extension-apiserver-authentication`).
- Chuỗi tin cậy TLS đi qua bốn bước 4, 5, 6 và 14: một CA ký server certificate; CN của
  certificate đó **phải** là `<service name>.<service name namespace>.svc`; cặp cert/key vào Pod
  qua Secret dạng volume; và **chính CA đó** mã hóa base64 (bỏ ký tự xuống dòng) trở thành
  `spec.caBundle` của APIService. Cộng cách nghiệm thu ở bước 15: `kubectl get` trả về
  `No resources found.` là dấu hiệu **thành công**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Điều kiện tiên quyết "cấu hình aggregation layer và bật các flag tương ứng của apiserver" | bài này chỉ nêu một dòng và trỏ đi, không giải thích cờ nào | bài [374](374-configure-aggregation-layer-vi.md) — bài 2/11 của chính [giai đoạn 28](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes), đọc ngay trước bài này; ở mức khái niệm là bài [180](180-apiserver-aggregation-vi.md) |
| `sample-apiserver` và `apiserver-builder` (bước 7 cần một image "hoạt động được") | phải viết và build binary cùng image từ mã nguồn Go — ngoài baseline của Lab 00, đúng lý do [Lab 14 ghi ở bảng phần không thực hành được](labs/LAB-14-CRD-VA-OPERATOR.md#11-ánh-xạ-tài-liệu-sang-bài-thực-hành) | không thuộc lộ trình — extension API server duy nhất bạn quan sát được trên cluster lab là metrics-server, đã mổ ở [Lab 14 phần B2](labs/LAB-14-CRD-VA-OPERATOR.md#b2-đúng-hai-cách-thêm-custom-resource-đo-trên-cluster-thật) |
| Bước 1 — kiểm `--runtime-config` xem API APIService còn bật không | bài nói rõ mặc định nó đã bật; chỉ phải đụng khi có người cố ý tắt, mà đó là sửa cờ của kube-apiserver đang chạy | [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |

---

Thiết lập một extension API server để làm việc với aggregation layer cho phép mở rộng Kubernetes
apiserver bằng các API bổ sung không thuộc nhóm API lõi của Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

* Bạn phải [cấu hình aggregation layer](374-configure-aggregation-layer-vi.md) và bật các flag
  tương ứng của apiserver.

## Thiết lập một extension api-server để làm việc với aggregation layer (Set up an extension api-server to work with the aggregation layer)

Các bước sau mô tả cách thiết lập một extension-apiserver *ở mức tổng quát*. Các bước này áp
dụng được dù bạn dùng file cấu hình YAML hay dùng API trực tiếp; chỗ nào hai cách khác nhau thì
tài liệu sẽ cố gắng chỉ rõ. Để xem một ví dụ cụ thể về cách hiện thực chúng bằng file cấu hình
YAML, bạn có thể tham khảo
[sample-apiserver](https://github.com/kubernetes/sample-apiserver/blob/master/README.md) trong
repo của Kubernetes.

Ngoài ra, bạn có thể dùng một giải pháp bên thứ ba có sẵn, chẳng hạn
[apiserver-builder](https://github.com/kubernetes-sigs/apiserver-builder-alpha/blob/master/README.md);
công cụ này sẽ sinh ra bộ khung (skeleton) và tự động hóa toàn bộ các bước dưới đây cho bạn.

1. Đảm bảo API APIService đang được bật (kiểm tra `--runtime-config`). Mặc định nó đã bật, trừ
   khi có ai đó cố ý tắt nó trong cluster của bạn.
2. Bạn có thể cần tạo một luật RBAC cho phép mình thêm các đối tượng APIService, hoặc nhờ quản
   trị viên cluster tạo giúp. (Vì các phần mở rộng API ảnh hưởng tới toàn bộ cluster, không nên
   thử nghiệm/phát triển/gỡ lỗi một API extension trên một cluster đang chạy thật.)
3. Tạo namespace Kubernetes mà bạn muốn chạy extension api-service bên trong đó.
4. Tạo (hoặc lấy) một CA certificate dùng để ký server certificate mà extension api-server sẽ
   dùng cho HTTPS.
5. Tạo một cặp certificate/key phía server để api-server dùng cho HTTPS. Certificate này phải
   được ký bởi CA ở trên. Nó cũng phải có CN là tên DNS trong Kubernetes (Kube DNS name). Tên
   này được suy ra từ Service Kubernetes và có dạng `<service name>.<service name namespace>.svc`
6. Tạo một Secret Kubernetes chứa cặp certificate/key phía server, đặt trong namespace của bạn.
7. Tạo một Deployment Kubernetes cho extension api-server và đảm bảo bạn nạp Secret đó vào dưới
   dạng một volume. Deployment phải tham chiếu tới một image hoạt động được của extension
   api-server. Deployment cũng phải nằm trong namespace của bạn.
8. Đảm bảo extension-apiserver của bạn nạp các certificate đó từ volume vừa nói, và chúng thực
   sự được dùng trong quá trình bắt tay (handshake) HTTPS.
9. Tạo một ServiceAccount Kubernetes trong namespace của bạn.
10. Tạo một ClusterRole Kubernetes cho những thao tác mà bạn muốn cho phép thực hiện trên các
    resource của mình.
11. Tạo một ClusterRoleBinding Kubernetes từ ServiceAccount trong namespace của bạn tới
    ClusterRole mà bạn vừa tạo.
12. Tạo một ClusterRoleBinding Kubernetes từ ServiceAccount trong namespace của bạn tới
    ClusterRole `system:auth-delegator`, để ủy quyền (delegate) các quyết định xác thực và phân
    quyền cho API server lõi của Kubernetes.
13. Tạo một RoleBinding Kubernetes từ ServiceAccount trong namespace của bạn tới Role
    `extension-apiserver-authentication-reader`. Việc này cho phép extension api-server của bạn
    truy cập ConfigMap `extension-apiserver-authentication`.
14. Tạo một apiservice Kubernetes. CA certificate ở trên phải được mã hóa base64, loại bỏ các ký
    tự xuống dòng, rồi dùng làm `spec.caBundle` trong apiservice. Đối tượng này không thuộc
    namespace nào. Nếu bạn dùng [kube-aggregator API](https://github.com/kubernetes/kube-aggregator/),
    chỉ cần truyền vào CA bundle ở dạng mã hóa PEM, vì phần mã hóa base64 đã được thực hiện sẵn
    cho bạn.
15. Dùng kubectl để lấy resource của bạn. Khi chạy, kubectl sẽ trả về "No resources found.".
    Thông báo này cho biết mọi thứ đã hoạt động, nhưng hiện tại bạn chưa tạo đối tượng nào thuộc
    loại resource đó.

## Tiếp theo (What's next)

* Đi qua các bước để [cấu hình aggregation layer của API](374-configure-aggregation-layer-vi.md)
  và bật các flag tương ứng của apiserver.
* Để có cái nhìn tổng quan ở mức cao, xem
  [Mở rộng Kubernetes API bằng aggregation layer](180-apiserver-aggregation-vi.md).
* Tìm hiểu cách [mở rộng Kubernetes API bằng CustomResourceDefinition](378-custom-resource-definitions-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 28:

1. Bước 11, 12 và 13 tạo ba binding khác nhau cho cùng một ServiceAccount. Mỗi binding cho
   extension api-server quyền làm gì, và cái nào là thứ khiến nó **không phải tự xác thực người
   dùng**?
2. **Câu bẫy.** Bạn chạy đủ mười lăm bước rồi `kubectl get` resource mới và nhận về
   `No resources found.`. Đó là dấu hiệu hỏng hay dấu hiệu chạy đúng? Trước đó, CN của server
   certificate phải đặt bằng gì, và cùng một CA certificate đó còn xuất hiện ở trường nào nữa?
3. Trên ba VM `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`, cluster đang chạy đúng
   một extension API server thật là metrics-server. Bạn **không** dựng thêm được extension API
   server của bài này trên chính ba VM đó. Hai thứ nào bài giả định bạn đã có mà cluster lab
   không có?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Ba việc tách rời.** Binding thứ nhất (bước 11) nối ServiceAccount tới ClusterRole **của
   riêng bạn** — quyền trên các resource mà chính extension api-server phục vụ. Binding thứ hai
   (bước 12) nối tới ClusterRole `system:auth-delegator`; **đây là cái trả lời vế sau**: nó
   **ủy quyền các quyết định xác thực và phân quyền cho API server lõi của Kubernetes**, nên
   extension api-server không phải tự làm việc đó. Binding thứ ba (bước 13) là một RoleBinding
   tới Role `extension-apiserver-authentication-reader`, cho phép đọc ConfigMap
   `extension-apiserver-authentication`.
2. **Chạy đúng.** Bài nói thẳng ở bước 15: thông báo đó cho biết mọi thứ đã hoạt động, chỉ là
   bạn **chưa tạo đối tượng nào** thuộc loại resource đó. Nếu chuỗi tin cậy hỏng thì lỗi sẽ
   không có dạng này. CN của server certificate phải là **tên DNS trong Kubernetes suy ra từ
   Service**, dạng `<service name>.<service name namespace>.svc`. Cùng CA certificate đó xuất
   hiện lần thứ hai ở **`spec.caBundle` của APIService** (bước 14), sau khi mã hóa base64 và bỏ
   các ký tự xuống dòng — trừ khi bạn dùng kube-aggregator API, vì ở đó phần base64 đã được làm
   sẵn.
3. **Một image extension api-server hoạt động được, và một aggregation layer đã cấu hình.**
   Bước 7 đòi Deployment tham chiếu tới một image chạy được — muốn có nó phải viết và build
   binary cùng image, chẳng hạn từ `sample-apiserver`. Mục *Trước khi bạn bắt đầu* đòi bạn đã
   [cấu hình aggregation layer](374-configure-aggregation-layer-vi.md) và bật các flag tương
   ứng của apiserver trên `lab-k8s-master`. metrics-server chạy được vì nó đã tới cluster ở
   dạng image dựng sẵn; bài này thì không phát cho bạn image nào.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

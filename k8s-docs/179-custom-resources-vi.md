# Tài nguyên tùy chỉnh (Custom Resources)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/>
>
> *Custom resource* (tài nguyên tùy chỉnh) là các phần mở rộng của Kubernetes API. Trang này bàn về việc khi nào nên thêm một custom resource vào cluster Kubernetes của bạn và khi nào nên dùng một dịch vụ độc lập. Trang cũng mô tả hai phương pháp để thêm custom resource và cách lựa chọn giữa chúng.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 14](LO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), bài 3/7 ·
Kiểm chứng ở Lab 14 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Giai đoạn này lộ trình ghi rõ là **dành cho platform administrator / người phát triển operator**.

Bài này viết cho **hai loại người đọc trộn lẫn**: người thiết kế API (nên chọn API nào) và người
vận hành cluster (cài custom resource vào có hệ quả gì). Với vai trò admin, **trọng tâm là hai
bảng so sánh CRD với aggregated API** ở mục *Chọn phương pháp thêm custom resource* — đó là thứ
lộ trình nhắm tới. Các mục về thiết kế API khai báo đọc lướt được.

**Phải hiểu ở lần đọc này:**

- Custom resource **một mình chỉ cho phép lưu trữ và truy xuất dữ liệu có cấu trúc**. Phải ghép
  với một **custom controller** thì nó mới thành *API khai báo* thực thụ — bạn khai báo trạng
  thái mong muốn, controller giữ trạng thái hiện tại đồng bộ với nó.
- Bảng *So sánh mức độ dễ dùng*: CRD **không đòi hỏi lập trình**, **không thêm dịch vụ nào phải
  chạy**, vá lỗi đến theo các đợt nâng cấp Kubernetes thông thường. Aggregated API thì phải
  build binary và image, **thêm một dịch vụ có thể hỏng**, và bạn tự tiếp nhận bản vá upstream.
- Bảng *Tính năng nâng cao và tính linh hoạt*: bốn thứ **CRD không làm được** — custom storage,
  subresource ngoài CRUD (`logs`, `exec`), `strategic-merge-patch`, Protocol Buffers. Đổi lại,
  validation, defaulting, multi-versioning, scale và status subresource thì CRD **có**.
- Ranh giới ConfigMap ↔ custom resource: ConfigMap khi file cấu hình được chương trình trong Pod
  đọc để tự cấu hình; custom resource khi bạn muốn `kubectl` hỗ trợ ở mức cao nhất, muốn watch
  thay đổi để tự động hóa, và muốn dùng quy ước `.spec`/`.status`/`.metadata`.
- Hệ quả vận hành ở mục *Chuẩn bị cài đặt*: CRD **luôn dùng chung** cơ chế xác thực, phân quyền
  và audit log với resource dựng sẵn (aggregated API server thì có thể không); và **hầu hết role
  RBAC sẽ không tự cấp quyền** cho resource mới — phải cấp tường minh.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bảng *Cân nhắc dùng API aggregation nếu / Ưu tiên một API độc lập nếu* | trả lời câu hỏi "có nên đưa API này vào Kubernetes không", chỉ cần khi bạn tự thiết kế API | Lab 14 |
| *API khai báo* — danh sách dấu hiệu của API mệnh lệnh | là tiêu chí thiết kế API, không phải việc của admin | Lab 14 |
| Bảng *Các tính năng chung* | liệt kê thứ bạn được cho không; bạn đã dùng chúng từ lâu trên resource dựng sẵn | giai đoạn 1 — bài [21](21-kubernetes-api-vi.md) |
| *Lưu trữ* — storage version và các phép chuyển đổi | chỉ gặp khi CRD của bạn đã có nhiều version | Lab 14 |
| *Field selector cho custom resource* và `selectableFields` | là tinh chỉnh sau khi CRD đã chạy | Lab 14 |

---

*Custom resource* là các phần mở rộng của Kubernetes API. Trang này bàn về việc khi nào nên thêm một custom resource vào cluster Kubernetes của bạn và khi nào nên dùng một dịch vụ độc lập (standalone service). Trang cũng mô tả hai phương pháp để thêm custom resource và cách lựa chọn giữa chúng.

## Custom resource

Một *resource* (tài nguyên) là một endpoint trong [Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) lưu trữ một tập hợp các đối tượng API (API object) thuộc một loại (kind) nhất định; ví dụ, resource dựng sẵn *pods* chứa một tập hợp các đối tượng Pod.

Một *custom resource* là một phần mở rộng của Kubernetes API mà không nhất thiết có sẵn trong một bản cài đặt Kubernetes mặc định. Nó thể hiện sự tùy biến của một bản cài đặt Kubernetes cụ thể. Tuy nhiên, ngày nay nhiều chức năng lõi của Kubernetes cũng được xây dựng bằng custom resource, khiến Kubernetes trở nên module hóa hơn.

Custom resource có thể xuất hiện và biến mất trong một cluster đang chạy thông qua cơ chế đăng ký động (dynamic registration), và quản trị viên cluster có thể cập nhật custom resource một cách độc lập với chính cluster. Khi một custom resource đã được cài đặt, người dùng có thể tạo và truy cập các đối tượng của nó bằng kubectl, y như cách họ làm với các resource dựng sẵn như *Pod*.

## Custom controller

Bản thân custom resource chỉ cho phép bạn lưu trữ và truy xuất dữ liệu có cấu trúc. Khi bạn kết hợp một custom resource với một *custom controller*, custom resource sẽ cung cấp một _API khai báo (declarative API)_ thực thụ.

[API khai báo](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) của Kubernetes áp đặt sự phân tách trách nhiệm. Bạn khai báo trạng thái mong muốn (desired state) cho resource của mình. Controller của Kubernetes giữ cho trạng thái hiện tại của các đối tượng Kubernetes đồng bộ với trạng thái mong muốn mà bạn đã khai báo. Điều này trái ngược với một API mệnh lệnh (imperative API), nơi bạn *ra lệnh* cho server phải làm gì.

Bạn có thể triển khai và cập nhật một custom controller trên một cluster đang chạy, độc lập với vòng đời của cluster. Custom controller có thể làm việc với bất kỳ loại resource nào, nhưng chúng đặc biệt hiệu quả khi được kết hợp với custom resource. [Mẫu thiết kế Operator (Operator pattern)](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) kết hợp custom resource và custom controller. Bạn có thể dùng custom controller để mã hóa tri thức nghiệp vụ (domain knowledge) của những ứng dụng cụ thể thành một phần mở rộng của Kubernetes API.

## Tôi có nên thêm custom resource vào cluster Kubernetes của mình không? (Should I add a custom resource to my Kubernetes cluster?)

Khi tạo một API mới, hãy cân nhắc nên [tổng hợp (aggregate) API của bạn với các API của cluster Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) hay để API của bạn đứng độc lập.

| Cân nhắc dùng API aggregation nếu: | Ưu tiên một API độc lập (stand-alone) nếu: |
| ---------------------------- | ---------------------------- |
| API của bạn mang tính [khai báo (Declarative)](#declarative-apis). | API của bạn không phù hợp với mô hình [khai báo (Declarative)](#declarative-apis). |
| Bạn muốn các kiểu (type) mới của mình đọc và ghi được bằng `kubectl`. | Không cần hỗ trợ `kubectl`. |
| Bạn muốn xem các kiểu mới của mình trong một giao diện Kubernetes UI, chẳng hạn dashboard, cùng với các kiểu dựng sẵn. | Không cần hỗ trợ Kubernetes UI. |
| Bạn đang phát triển một API mới. | Bạn đã có sẵn một chương trình phục vụ API của bạn và nó hoạt động tốt. |
| Bạn sẵn sàng chấp nhận các ràng buộc về định dạng mà Kubernetes áp đặt lên đường dẫn REST resource, chẳng hạn API Group và Namespace. (Xem [Tổng quan API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/).) | Bạn cần có các đường dẫn REST cụ thể để tương thích với một REST API đã được định nghĩa từ trước. |
| Các resource của bạn tự nhiên có phạm vi ở mức cluster hoặc namespace của một cluster. | Resource ở phạm vi cluster hoặc namespace không phù hợp; bạn cần kiểm soát chi tiết đường dẫn resource. |
| Bạn muốn tái sử dụng [các tính năng hỗ trợ của Kubernetes API](#common-features). | Bạn không cần những tính năng đó. |

### API khai báo (Declarative APIs) {#declarative-apis}

Trong một API khai báo, thường thì:

- API của bạn gồm một số lượng tương đối nhỏ các đối tượng (resource) tương đối nhỏ.
- Các đối tượng định nghĩa cấu hình của ứng dụng hoặc hạ tầng.
- Các đối tượng được cập nhật tương đối ít.
- Con người thường cần đọc và ghi các đối tượng đó.
- Các thao tác chính trên đối tượng mang tính CRUD (tạo, đọc, cập nhật và xóa).
- Không cần giao dịch (transaction) trải trên nhiều đối tượng: API biểu diễn một trạng thái mong muốn, không phải một trạng thái chính xác.

API mệnh lệnh (imperative API) thì không mang tính khai báo. Các dấu hiệu cho thấy API của bạn có thể không mang tính khai báo bao gồm:

- Client nói "hãy làm việc này", rồi nhận về một phản hồi đồng bộ khi việc đó hoàn tất.
- Client nói "hãy làm việc này", rồi nhận về một ID thao tác (operation ID), và phải kiểm tra một đối tượng Operation riêng để xác định request đã hoàn thành hay chưa.
- Bạn nói tới Remote Procedure Call (RPC).
- Lưu trữ trực tiếp lượng dữ liệu lớn; ví dụ, > vài kB mỗi đối tượng, hoặc > hàng nghìn đối tượng.
- Cần truy cập băng thông cao (duy trì hàng chục request mỗi giây).
- Lưu dữ liệu người dùng cuối (như hình ảnh, thông tin định danh cá nhân — PII, v.v.) hoặc dữ liệu quy mô lớn khác do ứng dụng xử lý.
- Các thao tác tự nhiên trên đối tượng không mang tính CRUD.
- API khó mô hình hóa thành các đối tượng.
- Bạn chọn cách biểu diễn các thao tác đang chờ xử lý bằng một operation ID hoặc một đối tượng operation.

## Tôi nên dùng ConfigMap hay custom resource? (Should I use a ConfigMap or a custom resource?)

Hãy dùng ConfigMap nếu một trong các điều sau đúng:

* Đã có sẵn một định dạng file cấu hình được tài liệu hóa tốt, chẳng hạn `mysql.cnf` hoặc `pom.xml`.
* Bạn muốn đưa toàn bộ cấu hình vào một key của một ConfigMap.
* Mục đích chính của file cấu hình là để một chương trình đang chạy trong một Pod trên cluster của bạn đọc file đó nhằm tự cấu hình chính nó.
* Bên tiêu thụ file muốn dùng nó thông qua file trong Pod hoặc biến môi trường trong Pod, thay vì qua Kubernetes API.
* Bạn muốn thực hiện rolling update thông qua Deployment, v.v., khi file được cập nhật.

> **Ghi chú:** Hãy dùng Secret cho dữ liệu nhạy cảm; Secret tương tự ConfigMap nhưng an toàn hơn.

Hãy dùng custom resource (CRD hoặc Aggregated API) nếu hầu hết các điều sau đúng:

* Bạn muốn dùng các thư viện client và CLI của Kubernetes để tạo và cập nhật resource mới.
* Bạn muốn `kubectl` hỗ trợ ở mức cao nhất (top-level); ví dụ, `kubectl get my-object object-name`.
* Bạn muốn xây dựng các cơ chế tự động hóa mới theo dõi (watch) các thay đổi trên đối tượng mới, rồi thực hiện CRUD trên các đối tượng khác, hoặc ngược lại.
* Bạn muốn viết cơ chế tự động hóa xử lý các cập nhật đối với đối tượng.
* Bạn muốn dùng các quy ước của Kubernetes API như `.spec`, `.status` và `.metadata`.
* Bạn muốn đối tượng đó là một lớp trừu tượng bao trên một tập hợp các resource được kiểm soát, hoặc là bản tóm lược của các resource khác.

## Thêm custom resource (Adding custom resources)

Kubernetes cung cấp hai cách để thêm custom resource vào cluster của bạn:

- CRD thì đơn giản và có thể được tạo mà không cần lập trình.
- [API Aggregation](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) đòi hỏi lập trình, nhưng cho phép kiểm soát nhiều hơn với các hành vi của API như cách dữ liệu được lưu trữ và cách chuyển đổi giữa các phiên bản API.

Kubernetes cung cấp hai lựa chọn này để đáp ứng nhu cầu của các nhóm người dùng khác nhau, sao cho không phải hy sinh tính dễ dùng lẫn tính linh hoạt.

Aggregated API là các API server phụ (subordinate) nằm phía sau API server chính, và API server chính đóng vai trò proxy. Cách bố trí này được gọi là [API Aggregation](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) (AA). Với người dùng, Kubernetes API trông như đã được mở rộng.

CRD cho phép người dùng tạo các loại resource mới mà không cần thêm một API server nữa. Bạn không cần hiểu về API Aggregation mới dùng được CRD.

Bất kể được cài đặt theo cách nào, các resource mới đều được gọi là Custom Resource để phân biệt với các resource dựng sẵn của Kubernetes (như pod).

> **Ghi chú:** Tránh dùng Custom Resource làm nơi lưu trữ dữ liệu của ứng dụng, dữ liệu người dùng cuối hay dữ liệu giám sát: các thiết kế kiến trúc lưu dữ liệu ứng dụng bên trong Kubernetes API thường thể hiện một thiết kế bị ràng buộc chặt (too closely coupled).
>
> Về mặt kiến trúc, các kiến trúc ứng dụng [cloud native](https://www.cncf.io/about/faq/#what-is-cloud-native) ưa chuộng sự ràng buộc lỏng (loose coupling) giữa các thành phần. Nếu một phần workload của bạn cần một dịch vụ hỗ trợ (backing service) cho hoạt động thường ngày, hãy chạy dịch vụ đó như một thành phần hoặc tiêu thụ nó như một dịch vụ bên ngoài. Bằng cách này, workload của bạn không phụ thuộc vào Kubernetes API cho hoạt động bình thường của nó.

## CustomResourceDefinition {#customresourcedefinitions}

API resource [CustomResourceDefinition](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) cho phép bạn định nghĩa custom resource. Việc định nghĩa một đối tượng CRD sẽ tạo ra một custom resource mới với tên và schema do bạn chỉ định. Kubernetes API sẽ phục vụ và đảm nhận việc lưu trữ custom resource của bạn. Bản thân tên của đối tượng CRD phải là một [tên DNS subdomain hợp lệ](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names) được suy ra từ tên resource đã định nghĩa và API group của nó; xem [cách tạo một CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions#create-a-customresourcedefinition) để biết thêm chi tiết. Hơn nữa, tên của một đối tượng có kind/resource được định nghĩa bởi một CRD cũng phải là một tên DNS subdomain hợp lệ.

Điều này giúp bạn không phải tự viết API server để xử lý custom resource, nhưng bản chất tổng quát của cách hiện thực này khiến bạn kém linh hoạt hơn so với [tổng hợp API server (API server aggregation)](#api-server-aggregation).

Hãy tham khảo [ví dụ custom controller](https://github.com/kubernetes/sample-controller) để xem cách đăng ký một custom resource mới, làm việc với các thực thể (instance) thuộc loại resource mới của bạn, và dùng một controller để xử lý sự kiện.

## Tổng hợp API server (API server aggregation) {#api-server-aggregation}

Thông thường, mỗi resource trong Kubernetes API đòi hỏi mã nguồn xử lý các request REST và quản lý việc lưu trữ bền vững (persistent storage) của các đối tượng. API server chính của Kubernetes xử lý các resource dựng sẵn như *pod* và *service*, đồng thời cũng có thể xử lý custom resource một cách tổng quát thông qua [CRD](#customresourcedefinitions).

[Tầng tổng hợp (aggregation layer)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) cho phép bạn cung cấp các hiện thực chuyên biệt cho custom resource của mình bằng cách viết và triển khai API server của riêng bạn. API server chính ủy quyền (delegate) các request tới API server của bạn đối với những custom resource mà bạn xử lý, giúp chúng khả dụng cho tất cả các client của nó.

## Chọn phương pháp thêm custom resource (Choosing a method for adding custom resources)

CRD dễ dùng hơn. Aggregated API linh hoạt hơn. Hãy chọn phương pháp đáp ứng tốt nhất nhu cầu của bạn.

Thông thường, CRD là lựa chọn phù hợp nếu:

* Bạn chỉ có một vài trường (field).
* Bạn dùng resource đó trong nội bộ công ty, hoặc như một phần của một dự án mã nguồn mở nhỏ (khác với một sản phẩm thương mại).

### So sánh mức độ dễ dùng (Comparing ease of use)

CRD dễ tạo hơn Aggregated API.

| CRD | Aggregated API |
| --------------------------- | -------------- |
| Không đòi hỏi lập trình. Người dùng có thể chọn bất kỳ ngôn ngữ nào cho controller của CRD. | Đòi hỏi lập trình và build binary cùng image. |
| Không có dịch vụ bổ sung nào phải chạy; CRD được API server xử lý. | Có thêm một dịch vụ phải tạo ra và dịch vụ đó có thể hỏng. |
| Không cần hỗ trợ liên tục sau khi CRD được tạo. Mọi bản vá lỗi sẽ được tiếp nhận như một phần của các đợt nâng cấp Kubernetes Master thông thường. | Có thể cần định kỳ tiếp nhận các bản vá lỗi từ upstream rồi build lại và cập nhật Aggregated API server. |
| Không cần xử lý nhiều phiên bản của API; ví dụ, khi bạn kiểm soát client cho resource này, bạn có thể nâng cấp nó đồng bộ với API. | Bạn cần xử lý nhiều phiên bản của API; ví dụ, khi phát triển một extension để chia sẻ với cộng đồng. |

### Tính năng nâng cao và tính linh hoạt (Advanced features and flexibility)

Aggregated API cung cấp nhiều tính năng API nâng cao hơn và cho phép tùy biến các tính năng khác; ví dụ, tầng lưu trữ (storage layer).

| Tính năng | Mô tả | CRD | Aggregated API |
| ------- | ----------- | ---- | -------------- |
| Validation (kiểm tra hợp lệ) | Giúp người dùng tránh lỗi và cho phép bạn phát triển API độc lập với các client của mình. Những tính năng này hữu ích nhất khi có nhiều client mà không phải tất cả đều cập nhật cùng lúc. | Có. Hầu hết việc validation có thể được khai báo trong CRD bằng [validation theo OpenAPI v3.0](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#validation). Feature gate [CRDValidationRatcheting](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#validation-ratcheting) cho phép bỏ qua các validation khai báo bằng OpenAPI bị thất bại nếu phần resource gây lỗi không thay đổi. Mọi validation khác được hỗ trợ bằng cách bổ sung một [Validating Webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook-alpha-in-1-8-beta-in-1-9). | Có, kiểm tra hợp lệ tùy ý. |
| Defaulting (giá trị mặc định) | Xem phần trên | Có, hoặc thông qua từ khóa `default` của [validation theo OpenAPI v3.0](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#defaulting) (GA từ 1.17), hoặc thông qua một [Mutating Webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook) (tuy nhiên webhook này sẽ không chạy khi đọc các đối tượng cũ từ etcd). | Có |
| Multi-versioning (đa phiên bản) | Cho phép phục vụ cùng một đối tượng qua hai phiên bản API. Có thể giúp việc thay đổi API dễ dàng hơn, chẳng hạn đổi tên trường. Ít quan trọng hơn nếu bạn kiểm soát được phiên bản client của mình. | [Có](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning) | Có |
| Custom Storage (lưu trữ tùy chỉnh) | Nếu bạn cần kho lưu trữ với đặc tính hiệu năng khác biệt (ví dụ, cơ sở dữ liệu chuỗi thời gian thay vì kho key-value) hoặc cần cô lập vì lý do bảo mật (ví dụ, mã hóa thông tin nhạy cảm, v.v.) | Không | Có |
| Custom Business Logic (logic nghiệp vụ tùy chỉnh) | Thực hiện các kiểm tra hoặc hành động tùy ý khi tạo, đọc, cập nhật hoặc xóa một đối tượng | Có, dùng [Webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks). | Có |
| Scale Subresource | Cho phép các hệ thống như HorizontalPodAutoscaler và PodDisruptionBudget tương tác với resource mới của bạn | [Có](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#scale-subresource) | Có |
| Status Subresource | Cho phép kiểm soát truy cập chi tiết, trong đó người dùng ghi phần spec còn controller ghi phần status. Cho phép tăng giá trị Generation của đối tượng khi dữ liệu custom resource thay đổi (yêu cầu resource có phần spec và status tách biệt) | [Có](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#status-subresource) | Có |
| Other Subresources (subresource khác) | Thêm các thao tác ngoài CRUD, chẳng hạn "logs" hoặc "exec". | Không | Có |
| strategic-merge-patch | Các endpoint mới hỗ trợ PATCH với `Content-Type: application/strategic-merge-patch+json`. Hữu ích khi cập nhật các đối tượng có thể bị sửa đổi cả ở phía cục bộ lẫn bởi server. Để biết thêm, xem ["Update API Objects in Place Using kubectl patch"](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/) | Không | Có |
| Protocol Buffers | Resource mới hỗ trợ các client muốn dùng Protocol Buffers | Không | Có |
| OpenAPI Schema | Có schema OpenAPI (swagger) cho các kiểu dữ liệu để có thể lấy động từ server hay không? Người dùng có được bảo vệ khỏi việc gõ sai tên trường bằng cách chỉ cho phép đặt các trường hợp lệ hay không? Kiểu dữ liệu có được ràng buộc hay không (nói cách khác, không đặt một `int` vào một trường `string`)? | Có, dựa trên schema [validation theo OpenAPI v3.0](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#validation) (GA từ 1.16). | Có |
| Instance Name (tên thực thể) | Cơ chế mở rộng này có áp đặt ràng buộc nào lên tên của các đối tượng có kind/resource được định nghĩa theo cách này hay không? | Có, tên của đối tượng như vậy phải là một tên DNS subdomain hợp lệ. | Không |

### Các tính năng chung (Common Features) {#common-features}

Khi bạn tạo một custom resource, dù bằng CRD hay AA, bạn nhận được nhiều tính năng cho API của mình, so với việc tự hiện thực nó bên ngoài nền tảng Kubernetes:

| Tính năng | Tính năng đó làm gì |
| ------- | ------------ |
| CRUD | Các endpoint mới hỗ trợ các thao tác CRUD cơ bản qua HTTP và `kubectl` |
| Watch | Các endpoint mới hỗ trợ các thao tác Watch của Kubernetes qua HTTP |
| Discovery | Các client như `kubectl` và dashboard tự động cung cấp các thao tác liệt kê, hiển thị và chỉnh sửa trường trên resource của bạn |
| json-patch | Các endpoint mới hỗ trợ PATCH với `Content-Type: application/json-patch+json` |
| merge-patch | Các endpoint mới hỗ trợ PATCH với `Content-Type: application/merge-patch+json` |
| HTTPS | Các endpoint mới dùng HTTPS |
| Xác thực dựng sẵn (Built-in Authentication) | Việc truy cập phần mở rộng sử dụng API server lõi (tầng tổng hợp) để xác thực |
| Phân quyền dựng sẵn (Built-in Authorization) | Việc truy cập phần mở rộng có thể tái sử dụng cơ chế phân quyền mà API server lõi đang dùng; ví dụ, RBAC. |
| Finalizers | Chặn việc xóa resource mở rộng cho tới khi quá trình dọn dẹp bên ngoài hoàn tất. |
| Admission Webhooks | Đặt giá trị mặc định và kiểm tra hợp lệ resource mở rộng trong bất kỳ thao tác tạo/cập nhật/xóa nào. |
| Hiển thị UI/CLI | Kubectl và dashboard có thể hiển thị resource mở rộng. |
| Chưa đặt so với rỗng (Unset versus Empty) | Client có thể phân biệt trường chưa được đặt với trường có giá trị bằng 0/rỗng. |
| Sinh thư viện client (Client Libraries Generation) | Kubernetes cung cấp các thư viện client tổng quát, cũng như công cụ để sinh ra các thư viện client riêng cho từng kiểu dữ liệu. |
| Label và annotation | Metadata dùng chung cho mọi đối tượng mà các công cụ đều biết cách chỉnh sửa, áp dụng cho cả resource lõi và custom resource. |

## Chuẩn bị cài đặt một custom resource (Preparing to install a custom resource)

Có một vài điểm cần lưu ý trước khi thêm một custom resource vào cluster của bạn.

### Mã của bên thứ ba và các điểm hỏng hóc mới (Third party code and new points of failure)

Mặc dù việc tạo một CRD không tự động thêm bất kỳ điểm hỏng hóc (point of failure) mới nào (ví dụ, không khiến mã của bên thứ ba chạy trên API server của bạn), các gói (ví dụ, Chart) hoặc các bộ cài đặt khác thường bao gồm cả CRD lẫn một Deployment chứa mã của bên thứ ba hiện thực logic nghiệp vụ cho custom resource mới.

Việc cài đặt một Aggregated API server thì luôn kéo theo việc chạy một Deployment mới.

### Lưu trữ (Storage)

Custom resource tiêu tốn không gian lưu trữ theo cách giống như ConfigMap. Việc tạo quá nhiều custom resource có thể làm quá tải không gian lưu trữ của API server.

Custom resource được đưa vào kho lưu trữ dựa trên phiên bản lưu trữ (storage version) hiện hành của resource, được định nghĩa trong phần spec của CRD. Mọi cập nhật đối với một custom resource sẽ dùng phiên bản lưu trữ đang được định nghĩa để lưu resource đó. Tất cả các phiên bản khác hoặc phải có đầy đủ các trường của phiên bản đó, hoặc phải định nghĩa các phép chuyển đổi (conversion) để hoạt động đúng.

Aggregated API server có thể dùng chung kho lưu trữ với API server chính, và trong trường hợp đó cảnh báo trên cũng áp dụng.

### Xác thực, phân quyền và kiểm toán (Authentication, authorization, and auditing)

CRD luôn dùng cùng cơ chế xác thực, phân quyền và ghi log kiểm toán (audit logging) như các resource dựng sẵn của API server.

Nếu bạn dùng RBAC để phân quyền, hầu hết các role RBAC sẽ không cấp quyền truy cập tới resource mới (ngoại trừ role cluster-admin hoặc bất kỳ role nào được tạo với luật wildcard). Bạn sẽ cần cấp quyền truy cập một cách tường minh cho resource mới. CRD và Aggregated API thường được đóng gói kèm các định nghĩa role mới cho những kiểu dữ liệu mà chúng bổ sung.

Aggregated API server có thể dùng hoặc không dùng cùng cơ chế xác thực, phân quyền và kiểm toán như API server chính.

## Truy cập một custom resource (Accessing a custom resource)

Có thể dùng [các thư viện client](https://kubernetes.io/docs/reference/using-api/client-libraries/) của Kubernetes để truy cập custom resource. Không phải mọi thư viện client đều hỗ trợ custom resource. Thư viện client _Go_ và _Python_ thì có hỗ trợ.

Khi bạn thêm một custom resource, bạn có thể truy cập nó bằng:

- `kubectl`
- Dynamic client của Kubernetes.
- Một REST client do bạn tự viết.
- Một client được sinh ra bằng [các công cụ sinh client của Kubernetes](https://github.com/kubernetes/code-generator) (việc tự sinh client là một công việc nâng cao, nhưng một số dự án có thể cung cấp sẵn client kèm theo CRD hoặc AA).

## Field selector cho custom resource (Custom resource field selectors)

[Field Selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/) cho phép client chọn lọc custom resource dựa trên giá trị của một hoặc nhiều trường của resource.

Mọi custom resource đều hỗ trợ các field selector `metadata.name` và `metadata.namespace`.

Các trường được khai báo trong một CustomResourceDefinition cũng có thể được dùng với field selector khi chúng được đưa vào trường `spec.versions[*].selectableFields` của CustomResourceDefinition.

### Các trường có thể chọn lọc cho custom resource (Selectable fields for custom resources) {#crd-selectable-fields}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [stable]`

Trường `spec.versions[*].selectableFields` của một CustomResourceDefinition có thể được dùng để khai báo những trường nào khác trong một custom resource có thể được dùng trong field selector.

Ví dụ sau đây thêm các trường `.spec.color` và `.spec.size` làm các trường có thể chọn lọc.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: shirts.stable.example.com
spec:
  group: stable.example.com
  scope: Namespaced
  names:
    plural: shirts
    singular: shirt
    kind: Shirt
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              color:
                type: string
              size:
                type: string
    selectableFields:
    - jsonPath: .spec.color
    - jsonPath: .spec.size
    additionalPrinterColumns:
    - jsonPath: .spec.color
      name: Color
      type: string
    - jsonPath: .spec.size
      name: Size
      type: string
```

Sau đó có thể dùng field selector để chỉ lấy các resource có `color` là `blue`:

```shell
kubectl get shirts.stable.example.com --field-selector spec.color=blue
```

Kết quả sẽ như sau:

```
NAME       COLOR  SIZE
example1   blue   S
example2   blue   M
```

## Tiếp theo (What's next)

* Tìm hiểu cách [Mở rộng Kubernetes API bằng tầng tổng hợp (aggregation layer)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/).
* Tìm hiểu cách [Mở rộng Kubernetes API bằng CustomResourceDefinition](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 14:

1. Bạn tạo một CRD `SampleDB` và `kubectl apply` một đối tượng của nó, nhưng không triển khai
   controller nào. Theo bài, bạn vừa có được thứ gì — và **chưa** có thứ gì?
2. Kể bốn khả năng mà **aggregated API có còn CRD không có**, theo bảng *Tính năng nâng cao và
   tính linh hoạt*. Còn *validation* và *defaulting* thì sao — CRD làm được không, và bằng cách
   nào?
3. Bạn cần một API mới gồm ba trường, dùng nội bộ công ty, không cần kho lưu trữ riêng. Bài
   khuyên chọn phương pháp nào và vì sao? Nếu sáu tháng sau bạn cần thêm một subresource `logs`
   thì quyết định đó còn đứng vững không?
4. Nhóm dev muốn dùng custom resource làm nơi lưu kết quả đo của ứng dụng, mỗi bản ghi vài chục
   kB và ghi hàng chục lần mỗi giây. Bài phản đối bằng những lập luận nào?
5. Trên cluster lab v1.35.6, bạn cài một CRD rồi cấp cho một ServiceAccount ClusterRole `view`
   dựng sẵn. ServiceAccount đó có đọc được custom resource mới không? Bài nói gì về RBAC và
   resource mới?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Bạn mới có **một kho lưu trữ dữ liệu có cấu trúc**: "Bản thân custom resource chỉ cho phép bạn
   lưu trữ và truy xuất dữ liệu có cấu trúc." Bạn **chưa có API khai báo**. API khai báo chỉ
   thành hình khi ghép custom resource với **custom controller** — thứ giữ cho trạng thái hiện
   tại đồng bộ với trạng thái mong muốn bạn khai báo. Không có controller thì `kubectl apply`
   chỉ ghi một bản ghi vào etcd và không có gì trong cluster thay đổi.
2. **Bốn thứ CRD không có:** *Custom Storage* (kho lưu trữ có đặc tính hiệu năng khác hoặc cô
   lập vì bảo mật), *Other Subresources* (thao tác ngoài CRUD như `logs`, `exec`),
   *strategic-merge-patch*, và *Protocol Buffers*. Ngoài ra còn một ràng buộc riêng của CRD:
   **tên đối tượng phải là DNS subdomain hợp lệ**, aggregated API thì không bị ràng buộc này.
   **Validation và defaulting thì CRD có**: validation khai báo bằng OpenAPI v3.0 (phần còn lại
   bổ sung bằng Validating Webhook), defaulting bằng từ khóa `default` của OpenAPI v3.0 hoặc
   Mutating Webhook. Đây là chỗ dễ nhầm — người ta hay tưởng CRD "không validate được".
3. **CRD.** Bài nêu đúng hai dấu hiệu này: "Bạn chỉ có một vài trường" và "Bạn dùng resource đó
   trong nội bộ công ty". Nhưng nếu sau này cần subresource `logs` thì quyết định **không còn
   đứng vững**: subresource ngoài CRUD nằm trong cột "Không" của CRD, chỉ aggregated API mới làm
   được. Nói cách khác, ràng buộc thực sự không phải số trường mà là **những tính năng API bạn
   sẽ cần**.
4. Ba lập luận. **(a)** Ở mục *Thêm custom resource*, bài ghi thẳng: "Tránh dùng Custom Resource
   làm nơi lưu trữ dữ liệu của ứng dụng, dữ liệu người dùng cuối hay dữ liệu giám sát" — đó là
   thiết kế **bị ràng buộc quá chặt**, trái với ưa chuộng ràng buộc lỏng của kiến trúc cloud
   native; workload sẽ phụ thuộc Kubernetes API cho hoạt động bình thường. **(b)** Ở mục *API
   khai báo*, "lưu trữ trực tiếp lượng dữ liệu lớn (> vài kB mỗi đối tượng)" và "cần truy cập
   băng thông cao (hàng chục request mỗi giây)" là dấu hiệu API **không mang tính khai báo**.
   **(c)** Ở mục *Lưu trữ*, custom resource tiêu tốn không gian lưu trữ như ConfigMap, và tạo
   quá nhiều có thể **làm quá tải không gian lưu trữ của API server**.
5. **Không.** Bài nói: "hầu hết các role RBAC sẽ không cấp quyền truy cập tới resource mới (ngoại
   trừ role cluster-admin hoặc bất kỳ role nào được tạo với luật wildcard). Bạn sẽ cần cấp quyền
   truy cập một cách tường minh cho resource mới." Lý do sâu hơn: **CRD dùng đúng cơ chế xác
   thực, phân quyền và audit log của API server** như resource dựng sẵn — nên nó cũng thừa hưởng
   nguyên tắc "không có rule thì không có quyền". Thực tế, CRD và aggregated API thường được
   đóng gói kèm sẵn các định nghĩa role mới cho kiểu dữ liệu chúng thêm vào.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

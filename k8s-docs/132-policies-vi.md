# Chính sách (Policies)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/policy/>
>
> Quản lý bảo mật và các thực hành tốt nhất (best practices) bằng chính sách.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7b](00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên),
bài 1/6 · Kiểm chứng ở Lab 7b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này là **trang mục lục** của cả nhánh policy, không phải bài dạy một cơ chế. Nó dài chưa
tới 90 dòng và phần lớn chỉ liệt kê rồi trỏ đi nơi khác. Đọc nó để lấy khung phân loại: sau
bài này bạn phải biết một yêu cầu bị chặn thì bị chặn ở đâu — trong API server hay trên node.
Bốn bài còn lại của nhóm 7b nằm gọn trong khung đó.

**Phải hiểu ở lần đọc này:**

- "Chính sách" trong Kubernetes không phải một loại đối tượng: đó là tên gọi chung cho các
  cấu hình dùng để quản lý cấu hình khác hoặc hành vi lúc chạy.
- Ba đối tượng API đóng vai trò chính sách mà bài nêu: NetworkPolicy hạn chế lưu lượng vào/ra,
  LimitRange quản lý ràng buộc cấp phát trên nhiều loại đối tượng, ResourceQuota giới hạn mức
  tiêu thụ của một namespace. Hai cái sau chính là hai bài kế tiếp.
- Ranh giới giữa **admission controller tích hợp** — chạy bên trong API server, bật bằng cờ
  `--enable-admission-plugins` — và **dynamic admission controller (admission webhook)** —
  chạy bên ngoài như ứng dụng riêng, đăng ký nhận webhook, tra cứu được cả dữ liệu bên ngoài.
- `ValidatingAdmissionPolicy` nằm giữa hai thứ đó: kiểm tra bằng CEL ngay trong API server, và
  có ba mức tác động — chặn, ghi kiểm toán, cảnh báo.
- Nhánh thứ hai, hoàn toàn không đi qua API server: **chính sách áp bằng cấu hình kubelet trên
  từng worker node** — giới hạn/dự trữ PID và Node Resource Managers. Đây đúng là hai bài
  [135](135-pid-limiting-vi.md) và [74](74-resource-managers-vi.md) ở cuối nhóm.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Danh sách đầy đủ admission controller tích hợp và cờ `--enable-admission-plugins` | chưa học chuỗi ba chặng authn → authz → admission | giai đoạn 9, bài [119](119-controlling-access-vi.md) |
| Cú pháp CEL và cách viết một `ValidatingAdmissionPolicy` | là kỹ thuật viết policy, không phải khung phân loại | giai đoạn 9, bài [119](119-controlling-access-vi.md) |
| *Áp dụng chính sách bằng dynamic admission control* — đăng ký webhook, failure policy | webhook hỏng có thể làm chết cluster; cần đọc kỹ đúng chỗ | giai đoạn 9, bài [173](173-admission-webhooks-vi.md) |
| *Các triển khai* — Kubewarden, Kyverno, OPA Gatekeeper, Polaris | dự án bên thứ ba | không cần |

---

Chính sách (policy) trong Kubernetes là các cấu hình dùng để quản lý các cấu hình khác
hoặc các hành vi lúc chạy (runtime). Kubernetes cung cấp nhiều hình thức chính sách khác nhau,
được mô tả dưới đây:

## Áp dụng chính sách bằng các đối tượng API (Apply policies using API objects)

Một số đối tượng API đóng vai trò như chính sách. Dưới đây là một vài ví dụ:

* [NetworkPolicy](./84-network-policies-vi.md) có thể được dùng để hạn chế
  lưu lượng vào (ingress) và ra (egress) của một workload.
* [LimitRange](./133-limit-range-vi.md) quản lý các ràng buộc cấp phát tài nguyên
  trên nhiều loại đối tượng khác nhau.
* [ResourceQuota](./134-resource-quotas-vi.md) giới hạn mức tiêu thụ tài nguyên
  của một namespace.

## Áp dụng chính sách bằng admission controller (Apply policies using admission controllers)

Một admission controller chạy bên trong API server
và có thể xác thực (validate) hoặc biến đổi (mutate) các yêu cầu API. Một số admission controller
đóng vai trò áp dụng chính sách.
Ví dụ, admission controller [AlwaysPullImages](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages)
sửa đổi một Pod mới để đặt chính sách kéo image thành `Always`.

Kubernetes có sẵn một số admission controller tích hợp, có thể cấu hình được thông qua
cờ `--enable-admission-plugins` của API server.

Chi tiết về admission controller, cùng với danh sách đầy đủ các admission controller hiện có,
được ghi lại trong một mục riêng:

* [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)

## Áp dụng chính sách bằng ValidatingAdmissionPolicy (Apply policies using ValidatingAdmissionPolicy)

Validating admission policy cho phép thực thi các kiểm tra xác thực có thể cấu hình được
ngay trong API server bằng ngôn ngữ Common Expression Language (CEL). Ví dụ, một `ValidatingAdmissionPolicy`
có thể được dùng để cấm sử dụng tag image `latest`.

Một `ValidatingAdmissionPolicy` hoạt động trên một yêu cầu API và có thể được dùng để chặn (block),
ghi kiểm toán (audit) và cảnh báo (warn) người dùng về các cấu hình không tuân thủ.

Chi tiết về API `ValidatingAdmissionPolicy`, kèm theo ví dụ, được ghi lại trong một mục riêng:

* [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)

## Áp dụng chính sách bằng dynamic admission control (Apply policies using dynamic admission control)

Dynamic admission controller (hay admission webhook) chạy bên ngoài API server
dưới dạng các ứng dụng riêng biệt, đăng ký nhận các yêu cầu webhook để thực hiện
xác thực hoặc biến đổi các yêu cầu API.

Dynamic admission controller có thể được dùng để áp dụng chính sách lên các yêu cầu API
và kích hoạt các luồng công việc (workflow) khác dựa trên chính sách. Một dynamic admission controller
có thể thực hiện các kiểm tra phức tạp, bao gồm cả những kiểm tra cần truy xuất
tài nguyên khác của cluster và dữ liệu bên ngoài. Ví dụ, một bước kiểm tra xác minh image
có thể tra cứu dữ liệu từ các OCI registry để xác thực chữ ký (signature)
và chứng thực (attestation) của container image.

Chi tiết về dynamic admission control được ghi lại trong một mục riêng:

* [Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)

### Các triển khai (Implementations) {#implementations-admission-control}

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần.
> Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án bên thứ ba này.

Trong hệ sinh thái Kubernetes, các dynamic admission controller đóng vai trò như những
bộ máy chính sách (policy engine) linh hoạt đang được phát triển, chẳng hạn:

- [Kubewarden](https://github.com/kubewarden)
- [Kyverno](https://kyverno.io)
- [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper)
- [Polaris](https://polaris.docs.fairwinds.com/admission-controller/)

## Áp dụng chính sách bằng cấu hình Kubelet (Apply policies using Kubelet configurations)

Kubernetes cho phép cấu hình Kubelet trên mỗi worker node. Một số cấu hình Kubelet đóng vai trò như chính sách:

* [Giới hạn và dự trữ Process ID](135-pid-limiting-vi.md)
  được dùng để giới hạn và dự trữ số PID có thể cấp phát.
* [Node Resource Managers](https://kubernetes.io/docs/concepts/policy/node-resource-managers/)
  có thể quản lý tài nguyên tính toán, bộ nhớ và thiết bị cho các workload
  nhạy cảm với độ trễ (latency-critical) và có thông lượng cao (high-throughput).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Bài chia việc áp chính sách thành mấy nhánh? Với mỗi nhánh, nói rõ chính sách được cưỡng
   chế ở đâu — trong API server hay trên node.
2. Admission controller tích hợp và dynamic admission controller khác nhau ở chỗ nào về nơi
   chạy và về khả năng? Cái nào tra cứu được dữ liệu bên ngoài cluster, và vì sao lại thế?
3. Trong ba đối tượng API mà bài liệt kê, cái nào giới hạn mức tiêu thụ của **cả một
   namespace**, còn cái nào ràng buộc việc cấp phát trên **từng đối tượng**?
4. Hai worker của bạn mỗi máy 2 vCPU / 6 GB RAM. Bạn muốn một Pod chạy loạn (fork bomb) không
   thể làm treo cả node. Theo bài này, cơ chế đó thuộc nhánh chính sách nào — và vì sao không
   tạo được nó bằng một đối tượng API trong namespace?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Bốn cách áp chính sách, gom thành hai nhánh theo nơi cưỡng chế.** Nhánh **API server**:
   (a) các đối tượng API đóng vai trò chính sách — NetworkPolicy, LimitRange, ResourceQuota;
   (b) admission controller tích hợp, chạy *bên trong* API server; (c) `ValidatingAdmissionPolicy`,
   cũng chạy *ngay trong* API server bằng CEL; (d) dynamic admission control, chạy *bên ngoài*
   nhưng vẫn được API server gọi tới trong lúc xử lý yêu cầu. Nhánh **node**: cấu hình Kubelet
   trên mỗi worker node — giới hạn/dự trữ PID và Node Resource Managers. Nhánh này **không đi
   qua API server** chút nào.
2. **Khác ở chỗ chạy trong hay ngoài API server, và hệ quả là khác về tầm với.** Admission
   controller tích hợp chạy bên trong API server và được bật bằng cờ `--enable-admission-plugins`;
   nó xác thực hoặc biến đổi yêu cầu API — ví dụ bài đưa ra là `AlwaysPullImages` sửa Pod mới để
   đặt chính sách kéo image thành `Always`. Dynamic admission controller (admission webhook) chạy
   **bên ngoài API server dưới dạng ứng dụng riêng biệt**, đăng ký nhận yêu cầu webhook. Chính vì
   là ứng dụng độc lập nên **nó mới thực hiện được các kiểm tra phức tạp cần truy xuất tài nguyên
   khác của cluster và dữ liệu bên ngoài** — bài lấy ví dụ bước xác minh image tra cứu OCI
   registry để kiểm chữ ký và chứng thực.
3. **ResourceQuota giới hạn mức tiêu thụ tài nguyên của một namespace; LimitRange quản lý các
   ràng buộc cấp phát tài nguyên trên nhiều loại đối tượng khác nhau.** NetworkPolicy không nằm
   trong trục tài nguyên — nó hạn chế lưu lượng ingress và egress của một workload. Giữ đúng cặp
   này vì hai bài ngay sau đây đi sâu vào từng cái.
4. **Nhánh cấu hình Kubelet** — cụ thể là giới hạn và dự trữ Process ID, thứ bài này liệt kê
   trong mục *Áp dụng chính sách bằng cấu hình Kubelet*. Không tạo được bằng đối tượng API trong
   namespace vì bài nói rõ đây là **cấu hình Kubelet trên mỗi worker node**, tức nó sống ở phía
   node chứ không phải trong API. Hệ quả trực tiếp cho cluster của bạn: muốn áp chính sách đó
   thì phải đụng vào cả `k8s-worker1` lẫn `k8s-worker2`, không có cách nào `kubectl apply` một
   file YAML là xong.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

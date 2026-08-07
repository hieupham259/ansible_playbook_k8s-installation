# Tên và ID của đối tượng (Object Names and IDs)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/names/>
>
> Mỗi đối tượng trong cluster của bạn có một Tên (Name) duy nhất cho loại tài nguyên đó.
> Mỗi đối tượng Kubernetes cũng có một UID duy nhất trên toàn bộ cluster của bạn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1b](LO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl),
bài 1/9 · Kiểm chứng ở Lab 1b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Phải hiểu ở lần đọc này:**

- Kubernetes định danh một object bằng **bốn thuộc tính**: API group, resource type, namespace,
  name. **Phiên bản API không nằm trong đó** — không thể lách trùng tên bằng cách dùng version
  khác.
- Name duy nhất trong phạm vi đó **tại một thời điểm**; xóa rồi tạo lại trùng tên là hợp lệ.
- UID do hệ thống sinh, duy nhất toàn cluster và **phân biệt các lần xuất hiện trong lịch sử**
  của cùng một tên. Bạn đã dùng chính điều này ở Lab 1a phần B8.
- Ràng buộc **DNS subdomain** (≤253 ký tự, chữ-số thường, `-` và `.`) — dạng phổ biến nhất.
- `generateName`: server tự nối hậu tố vào tiền tố bạn đưa.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Khác biệt chi tiết giữa RFC 1123 và RFC 1035 label | chỉ cần khi gặp lỗi validation cụ thể | tra lại khi bị từ chối tên |
| Feature gate `RelaxedServiceNameValidation` | liên quan tới Service | giai đoạn 5 |
| *Tên dùng làm phân đoạn đường dẫn* | ràng buộc hiếm gặp | tra khi cần |

Đừng học thuộc bốn bộ ràng buộc. Chỉ cần biết **chúng khác nhau** và khi API server từ chối
một cái tên thì phải tra xem resource đó đòi dạng nào.

---

Mỗi đối tượng trong cluster của bạn có một [_Tên (Name)_](#names) duy nhất cho loại tài nguyên (resource) đó.
Mỗi đối tượng Kubernetes cũng có một [_UID_](#uids) duy nhất trên toàn bộ cluster của bạn.

Ví dụ, bạn chỉ có thể có một Pod tên `myapp-1234` trong cùng một [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/), nhưng bạn có thể có một Pod và một Deployment cùng mang tên `myapp-1234`.

Với các thuộc tính do người dùng cung cấp mà không cần duy nhất, Kubernetes cung cấp [label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) và [annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/).

## Tên (Names) {#names}

Một chuỗi do client cung cấp dùng để tham chiếu tới một đối tượng trong URL tài nguyên (resource URL), chẳng hạn `/api/v1/pods/some-name`.

Tại một thời điểm, chỉ một đối tượng của một loại (kind) nhất định được mang một tên nhất định. Tuy nhiên, nếu bạn xóa đối tượng đó, bạn có thể tạo một đối tượng mới trùng tên.

Tên phải là duy nhất trên tất cả các [phiên bản API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-groups-and-versioning) của cùng một tài nguyên.

Kubernetes định danh duy nhất các đối tượng bằng tổ hợp của bốn thuộc tính:
* **Nhóm API (API group)** (ví dụ: `apps`)
* **Loại tài nguyên (Resource type)** (ví dụ: `deployments`)
* **Namespace** (đối với các tài nguyên thuộc namespace)
* **Tên (Name)**

Mặc dù bạn có thể truy cập một tài nguyên qua các phiên bản API khác nhau (chẳng hạn `v1` hoặc `v1beta1`), phiên bản chỉ đơn giản là một cách biểu diễn khác của cùng một đối tượng bên dưới. Vì phiên bản không phải là một phần của định danh duy nhất, bạn không thể tạo hai đối tượng có cùng tên và cùng loại tài nguyên trong cùng một namespace bằng cách dùng các phiên bản API khác nhau.

> **Ghi chú:** Trong những trường hợp đối tượng đại diện cho một thực thể vật lý, như một Node đại diện cho một máy chủ (host) vật lý, khi máy chủ được tạo lại với cùng tên mà Node không bị xóa đi và tạo lại, Kubernetes sẽ coi máy chủ mới là máy chủ cũ, điều này có thể dẫn đến những điểm không nhất quán.

Server có thể sinh tên khi `generateName` được cung cấp thay cho `name` trong yêu cầu tạo tài nguyên.
Khi dùng `generateName`, giá trị được cung cấp sẽ được dùng làm tiền tố (prefix) của tên, và server sẽ nối thêm
một hậu tố (suffix) được sinh ra vào đó. Dù tên được sinh tự động, nó vẫn có thể xung đột với các tên đã tồn tại,
dẫn đến phản hồi HTTP 409. Điều này trở nên ít có khả năng xảy ra hơn nhiều trong Kubernetes v1.31 trở đi,
vì server sẽ thực hiện tối đa 8 lần thử sinh một tên duy nhất trước khi trả về phản hồi HTTP 409.

Dưới đây là bốn loại ràng buộc tên (name constraint) thường dùng cho các tài nguyên.

### Tên DNS Subdomain (DNS Subdomain Names) {#dns-subdomain-names}

Hầu hết các loại tài nguyên yêu cầu tên có thể dùng làm tên DNS subdomain
như định nghĩa trong [RFC 1123](https://tools.ietf.org/html/rfc1123).
Điều này nghĩa là tên phải:

- chứa không quá 253 ký tự
- chỉ chứa các ký tự chữ và số viết thường (lowercase alphanumeric), '-' hoặc '.'
- bắt đầu bằng một ký tự chữ hoặc số
- kết thúc bằng một ký tự chữ hoặc số

### Tên label theo RFC 1123 (RFC 1123 Label Names) {#dns-label-names}

Một số loại tài nguyên yêu cầu tên của chúng tuân theo chuẩn DNS label
như định nghĩa trong [RFC 1123](https://tools.ietf.org/html/rfc1123).
Điều này nghĩa là tên phải:

- chứa tối đa 63 ký tự
- chỉ chứa các ký tự chữ và số viết thường hoặc '-'
- bắt đầu bằng một ký tự chữ cái (alphabetic)
- kết thúc bằng một ký tự chữ hoặc số

> **Ghi chú:** Khi feature gate `RelaxedServiceNameValidation` được bật,
> tên các đối tượng Service được phép bắt đầu bằng chữ số.

### Tên label theo RFC 1035 (RFC 1035 Label Names)

Một số loại tài nguyên yêu cầu tên của chúng tuân theo chuẩn DNS label
như định nghĩa trong [RFC 1035](https://tools.ietf.org/html/rfc1035).
Điều này nghĩa là tên phải:

- chứa tối đa 63 ký tự
- chỉ chứa các ký tự chữ và số viết thường hoặc '-'
- bắt đầu bằng một ký tự chữ cái (alphabetic)
- kết thúc bằng một ký tự chữ hoặc số

> **Ghi chú:** Mặc dù về mặt kỹ thuật RFC 1123 cho phép label bắt đầu bằng chữ số, hiện thực (implementation)
> hiện tại của Kubernetes yêu cầu cả label theo RFC 1035 lẫn RFC 1123 đều phải bắt đầu
> bằng một ký tự chữ cái. Ngoại lệ là khi feature gate `RelaxedServiceNameValidation`
> được bật cho các đối tượng Service, khi đó tên Service được phép bắt đầu bằng chữ số.

### Tên dùng làm phân đoạn đường dẫn (Path Segment Names)

Một số loại tài nguyên yêu cầu tên của chúng có thể được mã hóa an toàn thành một
phân đoạn đường dẫn (path segment). Nói cách khác, tên không được là "." hay ".." và tên
không được chứa "/" hay "%".

Dưới đây là một manifest ví dụ cho một Pod tên `nginx-demo`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

> **Ghi chú:** Một số loại tài nguyên có thêm các ràng buộc bổ sung đối với tên của chúng.

## Các UID (UIDs) {#uids}

Một chuỗi do hệ thống Kubernetes sinh ra để định danh duy nhất các đối tượng.

Mỗi đối tượng được tạo ra trong suốt vòng đời của một cluster Kubernetes đều có một UID riêng biệt. UID nhằm mục đích phân biệt giữa các lần xuất hiện trong lịch sử của những thực thể tương tự nhau.

UID của Kubernetes là các định danh duy nhất toàn cầu (universally unique identifiers, còn được gọi là UUID).
UUID được chuẩn hóa theo ISO/IEC 9834-8 và ITU-T X.667.

## Tiếp theo (What's next)

* Đọc về [label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) và [annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) trong Kubernetes.
* Xem tài liệu thiết kế [Identifiers and Names in Kubernetes](https://git.k8s.io/design-proposals-archive/architecture/identifiers.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Trong cùng một namespace, có thể vừa có Pod tên `web` vừa có Deployment tên `web` không?
   Còn hai Pod cùng tên `web` ở hai namespace khác nhau?
2. Kể bốn thuộc tính định danh duy nhất một object. Phiên bản API có nằm trong đó không, và
   hệ quả là gì?
3. Bạn xóa Pod `web` rồi tạo lại một Pod cũng tên `web`. Object mới có cùng UID với object cũ
   không? Vì sao Kubernetes thiết kế như vậy?
4. Ở Lab 1a phần B8, bạn so sánh giá trị nào để chứng minh ServiceAccount `default` được
   controller **tạo mới** chứ không phải chưa từng bị xóa?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Được cả hai.** Name chỉ cần duy nhất trong phạm vi *(API group, resource type, namespace)*.
   Pod và Deployment là hai resource type khác nhau nên không đụng nhau; hai namespace khác
   nhau cũng là hai phạm vi khác nhau.
2. API group, resource type, namespace, name. **Phiên bản API không nằm trong định danh** —
   các version chỉ là cách biểu diễn khác của cùng một dữ liệu. Hệ quả: không thể tạo hai
   object trùng tên cùng resource type trong cùng namespace bằng cách dùng `v1` và `v1beta1`.
3. **Không** — UID mới. UID sinh ra để phân biệt các lần xuất hiện trong lịch sử của những
   thực thể trông giống nhau, nên hệ thống luôn nói được "đây là object khác, không phải cái
   cũ hồi sinh".
4. So sánh **UID** trước và sau. Tên vẫn là `default` nên tên không chứng minh được gì; UID
   khác mới chứng minh đây là object mới do controller tạo.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

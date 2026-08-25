# Quản lý object Kubernetes theo kiểu imperative bằng file cấu hình (Imperative Management of Kubernetes Objects Using Configuration Files)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-config/>
>
> Các object Kubernetes có thể được tạo, cập nhật và xóa bằng công cụ dòng lệnh `kubectl` cùng
> với một file cấu hình object viết bằng YAML hoặc JSON. Tài liệu này giải thích cách định
> nghĩa và quản lý object bằng file cấu hình.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** tài liệu tra cứu thuộc nhánh `/docs/tasks/`
([Checkpoint tiếp nối](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)) — bài này không
thuộc CP nào của lộ trình; nó là bản thực hành cho kỹ thuật **imperative object
configuration**, kỹ thuật thứ hai trong ba kỹ thuật quản lý object mà bài
[27 — Quản lý object trong Kubernetes](27-object-management-vi.md) đã so sánh, nằm giữa
[lệnh imperative](320-imperative-command-vi.md) và
[quản lý declarative bằng `apply`](319-declarative-config-vi.md).

Giá trị lớn nhất của bài không phải cú pháp (chỉ có `create -f`, `replace -f`, `delete -f`,
`get -f`) mà là **các giới hạn** của kiểu quản lý này — chính các giới hạn đó lý giải vì sao
thực tế vận hành thường dùng `kubectl apply`.

**Phải hiểu ở lần đọc này:**

- Bốn lệnh cốt lõi làm việc với file cấu hình: `kubectl create -f`, `kubectl replace -f`,
  `kubectl delete -f`, `kubectl get -f <file> -o yaml`.
- Cảnh báo về `replace`: nó **bỏ toàn bộ phần spec không có trong file** — nguy hiểm với
  object có field do cluster quản lý độc lập, như `externalIPs` của Service kiểu
  `LoadBalancer`; các field đó phải được chép vào file để không bị mất.
- Giới hạn nhiều writer: nếu một nguồn khác (ví dụ HorizontalPodAutoscaler) cập nhật trực
  tiếp live object mà thay đổi không được gộp lại vào file, lần `replace` kế tiếp sẽ **xóa
  mất** thay đổi đó; cần nhiều writer thì dùng `kubectl apply`.
- Quy trình di trú từ lệnh imperative sang cấu hình imperative: export live object ra file
  (`kubectl get <kind>/<name> -o yaml >`), xóa field `status` thủ công, và từ đó về sau chỉ
  dùng `replace`.
- Trường hợp đặc biệt: file dùng `generateName` thay cho `name` trong `metadata` thì **không
  xóa được bằng `kubectl delete -f`** — phải xóa theo tên thật hoặc theo label.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục "Định nghĩa selector của controller và label của PodTemplate" | khuyến nghị thiết kế manifest cho người viết cấu hình, không ảnh hưởng thao tác `create`/`replace`/`delete` | đọc lại khi làm việc sâu với Deployment ở bài [63](63-deployment-vi.md) |

---

Các object Kubernetes có thể được tạo, cập nhật và xóa bằng công cụ dòng lệnh `kubectl` cùng
với một file cấu hình object viết bằng YAML hoặc JSON. Tài liệu này giải thích cách định nghĩa
và quản lý object bằng file cấu hình.

## Trước khi bạn bắt đầu (Before you begin)

Cài đặt [`kubectl`](185-tools-vi.md).

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

## Đánh đổi (Trade-offs)

Công cụ `kubectl` hỗ trợ ba kiểu quản lý object:

* Lệnh imperative (imperative commands)
* Cấu hình object kiểu imperative (imperative object configuration)
* Cấu hình object kiểu declarative (declarative object configuration)

Xem [Quản lý object trong Kubernetes](27-object-management-vi.md) để đọc phần thảo luận về ưu
điểm và nhược điểm của từng kiểu quản lý object.

## Cách tạo object (How to create objects)

Bạn có thể dùng `kubectl create -f` để tạo một object từ một file cấu hình. Tham khảo
[tài liệu tham khảo Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/)
để biết chi tiết.

* `kubectl create -f <filename|url>`

## Cách cập nhật object (How to update objects)

> **Cảnh báo:**
> Cập nhật object bằng lệnh `replace` sẽ bỏ toàn bộ những phần của spec không được chỉ định
> trong file cấu hình. Không nên dùng cách này với các object có spec được cluster quản lý một
> phần, chẳng hạn Service kiểu `LoadBalancer`, nơi field `externalIPs` được quản lý độc lập
> với file cấu hình. Các field được quản lý độc lập phải được chép vào file cấu hình để
> `replace` không xóa mất chúng.

Bạn có thể dùng `kubectl replace -f` để cập nhật một live object theo một file cấu hình.

* `kubectl replace -f <filename|url>`

## Cách xóa object (How to delete objects)

Bạn có thể dùng `kubectl delete -f` để xóa một object được mô tả trong một file cấu hình.

* `kubectl delete -f <filename|url>`

> **Ghi chú:**
> Nếu file cấu hình chỉ định field `generateName` trong phần `metadata` thay vì field `name`,
> bạn không thể xóa object bằng `kubectl delete -f <filename|url>`.
> Bạn sẽ phải dùng các flag khác để xóa object đó. Ví dụ:
>
> ```shell
> kubectl delete <type> <name>
> kubectl delete <type> -l <label>
> ```

## Cách xem một object (How to view an object)

Bạn có thể dùng `kubectl get -f` để xem thông tin về một object được mô tả trong một file
cấu hình.

* `kubectl get -f <filename|url> -o yaml`

Flag `-o yaml` chỉ định rằng toàn bộ cấu hình của object sẽ được in ra. Dùng `kubectl get -h`
để xem danh sách các tùy chọn.

## Giới hạn (Limitations)

Các lệnh `create`, `replace` và `delete` hoạt động tốt khi cấu hình của mỗi object được định
nghĩa đầy đủ và được lưu lại trong file cấu hình của nó. Tuy nhiên, khi một live object được
cập nhật mà các cập nhật đó không được gộp (merge) vào file cấu hình của nó, những cập nhật đó
sẽ bị mất ở lần chạy `replace` kế tiếp. Điều này có thể xảy ra khi một controller, chẳng hạn
HorizontalPodAutoscaler, thực hiện cập nhật trực tiếp lên live object. Dưới đây là một ví dụ:

1. Bạn tạo một object từ một file cấu hình.
1. Một nguồn khác cập nhật object bằng cách thay đổi một field nào đó.
1. Bạn replace object từ file cấu hình. Các thay đổi mà nguồn khác đã thực hiện ở bước 2 bị
   mất.

Nếu bạn cần hỗ trợ nhiều writer cùng ghi vào một object, bạn có thể dùng `kubectl apply` để
quản lý object đó.

## Tạo và chỉnh sửa một object từ URL mà không lưu cấu hình (Creating and editing an object from a URL without saving the configuration)

Giả sử bạn có URL của một file cấu hình object. Bạn có thể dùng `kubectl create --edit` để
thay đổi cấu hình trước khi object được tạo. Cách này đặc biệt hữu ích cho các bài hướng dẫn
(tutorial) và tác vụ trỏ tới một file cấu hình mà người đọc có thể muốn sửa đổi.

```shell
kubectl create -f <url> --edit
```

## Di trú từ lệnh imperative sang cấu hình object kiểu imperative (Migrating from imperative commands to imperative object configuration)

Việc di trú từ lệnh imperative sang cấu hình object kiểu imperative gồm một số bước thủ công.

1. Export live object ra một file cấu hình object cục bộ:

    ```shell
    kubectl get <kind>/<name> -o yaml > <kind>_<name>.yaml
    ```

1. Xóa thủ công field status khỏi file cấu hình object.

1. Với việc quản lý object từ đó về sau, chỉ dùng duy nhất `replace`.

    ```shell
    kubectl replace -f <kind>_<name>.yaml
    ```

## Định nghĩa selector của controller và label của PodTemplate (Defining controller selectors and PodTemplate labels)

> **Cảnh báo:**
> Việc cập nhật selector trên các controller là điều rất không nên làm.

Cách tiếp cận được khuyến nghị là định nghĩa một label PodTemplate duy nhất, bất biến
(immutable), chỉ được dùng bởi selector của controller và không mang ý nghĩa ngữ nghĩa nào
khác.

Ví dụ về label:

```yaml
selector:
  matchLabels:
      controller-selector: "apps/v1/deployment/nginx"
template:
  metadata:
    labels:
      controller-selector: "apps/v1/deployment/nginx"
```

## Tiếp theo (What's next)

* [Quản lý object Kubernetes bằng lệnh imperative](320-imperative-command-vi.md)
* [Quản lý object Kubernetes theo kiểu declarative bằng file cấu hình](319-declarative-config-vi.md)
* [Tài liệu tham khảo lệnh Kubectl](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/)
* [Tài liệu tham khảo Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn này.

1. Vì sao bài cảnh báo không dùng `replace` với Service kiểu `LoadBalancer`? Nếu vẫn phải
   dùng, bạn phải làm gì với field `externalIPs` trước khi chạy `replace`?
2. Trên cluster lab của bạn, một Deployment được quản lý bằng `replace -f` và đồng thời có
   một HorizontalPodAutoscaler đang điều chỉnh số replica. Chuyện gì xảy ra với số replica ở
   lần `replace` kế tiếp, và bài đề xuất lối thoát nào?
3. Câu bẫy: file cấu hình dùng `generateName` trong `metadata` — sau khi tạo object thành
   công bằng `create -f`, bạn có xóa được nó bằng `delete -f` chính file đó không? Vì sao, và
   xóa bằng cách nào?
4. Ba bước di trú một object từ quản lý bằng lệnh imperative sang cấu hình imperative là gì,
   và vì sao phải xóa field status khỏi file export?
5. Khuyến nghị của bài về label dùng cho selector của controller là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì `replace` **bỏ toàn bộ phần spec không có trong file cấu hình**, mà với Service kiểu
   `LoadBalancer`, field `externalIPs` được cluster quản lý độc lập với file — nên `replace`
   sẽ xóa mất nó. Nếu vẫn dùng, bạn phải **chép các field được quản lý độc lập đó vào file cấu
   hình** trước khi chạy `replace`.
2. Số replica do HPA đặt trực tiếp trên live object **sẽ bị mất**, vì thay đổi đó không được
   gộp vào file cấu hình — `replace` ghi đè theo đúng nội dung file. Lối thoát của bài: khi
   cần **nhiều writer cùng ghi một object thì dùng `kubectl apply`** thay cho `replace`.
3. **Không.** Trực giác "tạo bằng file nào thì xóa bằng file đó" sai ở đây: file chỉ có
   `generateName`, còn tên thật của object được server sinh ra khi tạo, nên `delete -f` không
   biết object nào cần xóa. Phải xóa bằng flag khác: `kubectl delete <type> <name>` với tên
   thật, hoặc `kubectl delete <type> -l <label>` theo label.
4. Ba bước: (1) export live object ra file bằng `kubectl get <kind>/<name> -o yaml > file`;
   (2) xóa thủ công field status khỏi file; (3) từ đó về sau chỉ dùng `replace` để quản lý.
   Phải xóa status vì đó là phần do cluster ghi nhận trạng thái thực tế, không phải cấu hình
   mà bạn định nghĩa — bài yêu cầu file cấu hình chỉ chứa phần bạn quản lý.
5. Định nghĩa **một label PodTemplate duy nhất và bất biến, chỉ dùng cho selector của
   controller, không mang ý nghĩa ngữ nghĩa nào khác** (ví dụ
   `controller-selector: "apps/v1/deployment/nginx"`) — và tuyệt đối tránh cập nhật selector
   trên controller.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

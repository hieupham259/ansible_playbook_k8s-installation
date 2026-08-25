# Quản lý object Kubernetes bằng lệnh imperative (Managing Kubernetes Objects Using Imperative Commands)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-command/>
>
> Các object Kubernetes có thể được tạo, cập nhật và xóa nhanh chóng bằng các lệnh imperative
> tích hợp sẵn trong công cụ dòng lệnh `kubectl`. Tài liệu này giải thích cách các lệnh đó được
> tổ chức và cách dùng chúng để quản lý các object đang chạy (live object).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** tài liệu tra cứu thuộc nhánh `/docs/tasks/`
([Checkpoint tiếp nối](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)) — bài này không
thuộc CP nào của lộ trình; nó là bản thực hành chi tiết cho kỹ thuật **imperative commands**,
kỹ thuật thứ nhất trong ba kỹ thuật quản lý object mà bài
[27 — Quản lý object trong Kubernetes](27-object-management-vi.md) đã so sánh.

Bài này là bài tra cứu cú pháp: không có khái niệm mới, chỉ hệ thống hóa các lệnh `kubectl`
mà bạn đã dùng rải rác trong các lab.

**Phải hiểu ở lần đọc này:**

- Lệnh tạo object có hai họ: lệnh theo **động từ** (`run`, `expose`, `autoscale`) dành cho
  người chưa thuộc các loại object, và lệnh theo **loại object**
  (`create <objecttype> [<subtype>] <instancename>`) tường minh hơn nhưng đòi hỏi biết loại
  object cần tạo.
- Lệnh cập nhật cũng có hai họ tương ứng: theo động từ (`scale`, `annotate`, `label`) và theo
  khía cạnh của object (`set <field>`); ngoài ra `edit` và `patch` sửa trực tiếp live object
  nhưng đòi hỏi hiểu schema của object.
- `kubectl delete` dùng được cho cả imperative command lẫn imperative object configuration —
  khác nhau ở **đối số**: truyền object dạng `<type>/<name>` là imperative command, truyền
  file bằng `-f` là imperative object configuration.
- Hai kỹ thuật chỉnh object **trước khi tạo** khi `create` không có flag cho field cần đặt:
  ghép pipeline `create --dry-run=client -o yaml | set --local -f - | create -f -`, hoặc
  `create --edit` để mở editor sửa tùy ý.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chi tiết chuỗi patch (patch string) trong link API Conventions | tài liệu dành cho người phát triển; bài này chỉ cần biết `patch` sửa field cụ thể của live object | bài 324 — Update API Objects in Place Using kubectl patch (cùng nhóm `manage-kubernetes-objects`) |

---

Các object Kubernetes có thể được tạo, cập nhật và xóa nhanh chóng, trực tiếp bằng các lệnh
imperative tích hợp sẵn trong công cụ dòng lệnh `kubectl`. Tài liệu này giải thích cách các
lệnh đó được tổ chức và cách dùng chúng để quản lý các object đang chạy (live object).

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

Công cụ `kubectl` hỗ trợ các lệnh theo động từ (verb-driven) để tạo một số loại object phổ
biến nhất. Các lệnh này được đặt tên sao cho người dùng chưa quen với các loại object của
Kubernetes vẫn nhận ra được.

- `run`: Tạo một Pod mới để chạy một Container.
- `expose`: Tạo một object Service mới để cân bằng tải traffic giữa các Pod.
- `autoscale`: Tạo một object Autoscaler mới để tự động scale theo chiều ngang một controller,
  chẳng hạn một Deployment.

Công cụ `kubectl` cũng hỗ trợ các lệnh tạo object theo loại object (object type). Các lệnh này
hỗ trợ nhiều loại object hơn và tường minh hơn về ý định của chúng, nhưng đòi hỏi người dùng
phải biết loại object mà họ định tạo.

- `create <objecttype> [<subtype>] <instancename>`

Một số loại object có các loại con (subtype) mà bạn có thể chỉ định trong lệnh `create`.
Ví dụ, object Service có nhiều subtype bao gồm ClusterIP, LoadBalancer và NodePort. Dưới đây
là ví dụ tạo một Service với subtype NodePort:

```shell
kubectl create service nodeport <myservicename>
```

Trong ví dụ trên, lệnh `create service nodeport` được gọi là một lệnh con (subcommand) của
lệnh `create service`.

Bạn có thể dùng flag `-h` để tìm các đối số và flag mà một subcommand hỗ trợ:

```shell
kubectl create service nodeport -h
```

## Cách cập nhật object (How to update objects)

Lệnh `kubectl` hỗ trợ các lệnh theo động từ cho một số thao tác cập nhật phổ biến. Các lệnh
này được đặt tên để người dùng chưa quen với object Kubernetes vẫn thực hiện được việc cập
nhật mà không cần biết những field cụ thể nào phải được thiết lập:

- `scale`: Scale theo chiều ngang một controller để thêm hoặc bớt Pod bằng cách cập nhật số
  lượng replica của controller đó.
- `annotate`: Thêm hoặc gỡ một annotation khỏi một object.
- `label`: Thêm hoặc gỡ một label khỏi một object.

Lệnh `kubectl` cũng hỗ trợ các lệnh cập nhật theo một khía cạnh (aspect) của object. Việc
thiết lập khía cạnh này có thể đặt các field khác nhau tùy theo loại object:

- `set` `<field>`: Thiết lập một khía cạnh của một object.

> **Ghi chú:**
> Trong Kubernetes phiên bản 1.5, không phải lệnh theo động từ nào cũng có lệnh theo khía cạnh
> tương ứng.

Công cụ `kubectl` còn hỗ trợ các cách sau để cập nhật trực tiếp một live object, tuy nhiên
chúng đòi hỏi hiểu biết tốt hơn về schema object của Kubernetes.

- `edit`: Trực tiếp chỉnh sửa cấu hình thô của một live object bằng cách mở cấu hình đó trong
  một trình soạn thảo (editor).
- `patch`: Trực tiếp sửa các field cụ thể của một live object bằng một chuỗi patch (patch
  string). Để biết thêm chi tiết về chuỗi patch, xem phần patch trong
  [API Conventions](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#patch-operations).

## Cách xóa object (How to delete objects)

Bạn có thể dùng lệnh `delete` để xóa một object khỏi cluster:

- `delete <type>/<name>`

> **Ghi chú:**
> Bạn có thể dùng `kubectl delete` cho cả lệnh imperative lẫn cấu hình object kiểu imperative.
> Khác biệt nằm ở các đối số truyền cho lệnh. Để dùng `kubectl delete` như một lệnh
> imperative, hãy truyền object cần xóa làm đối số. Dưới đây là ví dụ truyền một object
> Deployment có tên nginx:

```shell
kubectl delete deployment/nginx
```

## Cách xem một object (How to view an object)

Có một số lệnh để in thông tin về một object:

- `get`: In thông tin cơ bản về các object khớp điều kiện. Dùng `get -h` để xem danh sách các
  tùy chọn.
- `describe`: In thông tin chi tiết được tổng hợp về các object khớp điều kiện.
- `logs`: In stdout và stderr của một container đang chạy trong một Pod.

## Dùng lệnh `set` để sửa object trước khi tạo (Using `set` commands to modify objects before creation)

Có một số field của object không có flag tương ứng để dùng trong lệnh `create`. Trong một số
trường hợp như vậy, bạn có thể kết hợp `set` và `create` để chỉ định giá trị cho field đó
trước khi object được tạo. Cách làm là nối (pipe) output của lệnh `create` sang lệnh `set`,
rồi nối ngược lại cho lệnh `create`. Dưới đây là một ví dụ:

```sh
kubectl create service clusterip my-svc --clusterip="None" -o yaml --dry-run=client | kubectl set selector --local -f - 'environment=qa' -o yaml | kubectl create -f -
```

1. Lệnh `kubectl create service -o yaml --dry-run=client` tạo cấu hình cho Service, nhưng in
   nó ra stdout dưới dạng YAML thay vì gửi tới Kubernetes API server.
1. Lệnh `kubectl set selector --local -f - -o yaml` đọc cấu hình từ stdin và ghi cấu hình đã
   cập nhật ra stdout dưới dạng YAML.
1. Lệnh `kubectl create -f -` tạo object bằng cấu hình được cung cấp qua stdin.

## Dùng `--edit` để sửa object trước khi tạo (Using `--edit` to modify objects before creation)

Bạn có thể dùng `kubectl create --edit` để thực hiện các thay đổi tùy ý trên một object trước
khi nó được tạo. Dưới đây là một ví dụ:

```sh
kubectl create service clusterip my-svc --clusterip="None" -o yaml --dry-run=client > /tmp/srv.yaml
kubectl create --edit -f /tmp/srv.yaml
```

1. Lệnh `kubectl create service` tạo cấu hình cho Service và lưu vào `/tmp/srv.yaml`.
1. Lệnh `kubectl create --edit` mở file cấu hình để chỉnh sửa trước khi tạo object.

## Tiếp theo (What's next)

* [Quản lý object Kubernetes theo kiểu imperative bằng file cấu hình](321-imperative-config-vi.md)
* [Quản lý object Kubernetes theo kiểu declarative bằng file cấu hình](319-declarative-config-vi.md)
* [Tài liệu tham khảo lệnh Kubectl](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/)
* [Tài liệu tham khảo Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn này.

1. `run`, `expose`, `autoscale` và `create <objecttype>` đều tạo object. Hai họ lệnh này khác
   nhau ở điểm gì, và họ nào đòi hỏi người dùng biết trước loại object cần tạo?
2. Câu bẫy: `kubectl delete deployment/nginx` và `kubectl delete -f nginx.yaml` cùng là lệnh
   `delete` — vì sao một lệnh được xếp vào imperative command còn lệnh kia thì không?
3. Trong pipeline
   `kubectl create service clusterip ... --dry-run=client -o yaml | kubectl set selector --local -f - ... | kubectl create -f -`,
   flag `--dry-run=client` và `--local` đóng vai trò gì? Nếu bỏ `--dry-run=client` thì chuyện
   gì xảy ra?
4. Trên cluster lab của bạn, muốn tạo một Service mà phải đặt trước một field không có flag
   trong `kubectl create service`, bài đưa ra hai cách làm nào?
5. `scale` và `set` đều cập nhật live object. Vì sao bài xếp `edit` và `patch` vào nhóm riêng
   "đòi hỏi hiểu schema tốt hơn"?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Lệnh theo động từ (`run`, `expose`, `autoscale`) được đặt tên theo **hành động** để người
   chưa quen các loại object Kubernetes vẫn dùng được. **`create <objecttype>` đòi hỏi biết
   trước loại object** cần tạo — đổi lại nó hỗ trợ nhiều loại object hơn và tường minh hơn về
   ý định.
2. **Khác biệt nằm ở đối số, không phải ở tên lệnh.** Truyền thẳng object dạng `<type>/<name>`
   làm đối số là dùng `delete` như một imperative command; truyền file cấu hình bằng `-f` là
   imperative object configuration. Trực giác "mỗi lệnh thuộc đúng một kiểu quản lý" là sai —
   `kubectl delete` phục vụ cả hai kiểu.
3. `--dry-run=client` khiến `kubectl create` **chỉ in cấu hình ra stdout dạng YAML thay vì gửi
   tới API server**; `--local` khiến `kubectl set` xử lý cấu hình đọc từ stdin ngay tại client
   chứ không đụng tới live object. Nếu bỏ `--dry-run=client`, Service sẽ bị tạo ngay từ lệnh
   đầu tiên — trước khi selector kịp được đặt.
4. Cách một: ghép pipeline `create --dry-run=client -o yaml | set --local -f - | create -f -`
   để `set` đặt field trên cấu hình trước khi `create -f -` gửi nó lên. Cách hai: dùng
   `kubectl create --edit -f <file>` để mở editor sửa tùy ý cấu hình trước khi tạo.
5. Vì `scale`, `annotate`, `label`, `set` được thiết kế để người dùng **không cần biết field
   cụ thể nào phải đặt** — lệnh tự ánh xạ sang field đúng. Còn `edit` mở cấu hình thô của live
   object và `patch` yêu cầu tự viết chuỗi patch trỏ đúng field, nên cả hai đòi hỏi nắm schema
   object của Kubernetes.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

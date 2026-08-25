# Liệt kê tất cả Container image đang chạy trong Cluster (List All Container Images Running in a Cluster)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/access-application-cluster/list-all-running-container-images/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 1 — Mô hình Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-1--mô-hình-kubernetes)
→ nhóm [1b. Làm việc với object và kubectl](00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl),
bài 8/8 — bài **cuối** của dòng **Thực hành**, làm ngay trước khi mở lab · Kiểm chứng ở
[Lab 1b — Object, label, kubectl và kubeconfig](labs/LAB-1B-OBJECT-LABEL-KUBECTL-VA-KUBECONFIG.md)
phần B3, nơi lab dùng `-l` để chọn tập object và `-o jsonpath` để đọc đúng một field ra khỏi kết
quả.

Bài ngắn và có một mục đích hẹp: biến `kubectl get` thành công cụ trích dữ liệu. Nội dung về
image ở đây chỉ là ví dụ; thứ đáng mang theo là **cách viết biểu thức jsonpath và ranh giới giữa
việc kubectl lọc và việc shell xử lý**. Bài dùng cụm `initContainers` — khái niệm này thuộc
[nhóm 3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời); ở đây chỉ cần biết đó là một danh sách
container khác nằm trong `spec`.

**Phải hiểu ở lần đọc này:**

- Cách đọc biểu thức `{.items[*].spec['initContainers', 'containers'][*].image}` theo đúng bốn
  vế mà bài tự diễn giải: `.items[*]` là mỗi phần tử trả về, `.spec` lấy phần spec,
  `['initContainers', 'containers'][*]` là mỗi container trong **cả hai** danh sách, `.image`
  lấy field `image`.
- Khi lấy **một** Pod theo tên (`kubectl get pod nginx`), phải **bỏ phần `.items[*]`** đi, vì kết
  quả trả về là một Pod đơn lẻ chứ không phải một danh sách — ghi chú này nằm ngay sau lệnh gộp
  đầu tiên.
- Ranh giới công việc: `kubectl` chọn Pod và trích field; phần gom nhóm và đếm là **công cụ
  shell** — `tr` đổi khoảng trắng thành dòng, `sort` sắp xếp, `uniq -c` đếm. Không có tùy chọn
  nào của `kubectl` làm việc đếm đó.
- Ba cách thu hẹp tập Pod trước khi định dạng: `--all-namespaces` để quét cả cluster, `-l` để lọc
  theo label của Pod (mục *Liệt kê Container image có lọc theo label của Pod*), và `--namespace`
  để giới hạn trong một namespace (mục *Liệt kê Container image có lọc theo namespace của Pod*).
- Hai cách kiểm soát định dạng ngoài biểu thức phẳng: phép `range` của jsonpath để lặp và in
  theo từng Pod, và `-o go-template --template=...` như một cơ chế thay thế jsonpath.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Gợi ý dựng cluster bằng minikube hoặc các playground ở mục *Trước khi bạn bắt đầu* | lộ trình đã có cluster ba VM và không dùng minikube | [Lab 00 — Môi trường](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Cú pháp go-template và trang tham khảo của nó | jsonpath đủ cho mọi việc trích dữ liệu trong lộ trình; Lab 1b chỉ dùng `-o jsonpath` | không quay lại trong lộ trình — đọc khi cần định dạng phức tạp hơn jsonpath làm được |

---

Trang này hướng dẫn cách dùng kubectl để liệt kê tất cả các Container image
của các Pod đang chạy trong một cluster.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

Trong bài thực hành này, bạn sẽ dùng kubectl để lấy tất cả các Pod
đang chạy trong một cluster, rồi định dạng đầu ra để rút ra danh sách
các Container của từng Pod.

## Liệt kê tất cả Container image trong mọi namespace (List all Container images in all namespaces)

- Lấy tất cả các Pod trong mọi namespace bằng `kubectl get pods --all-namespaces`
- Định dạng đầu ra để chỉ bao gồm danh sách tên Container image
  bằng `-o jsonpath={.items[*].spec['initContainers', 'containers'][*].image}`. Cách này sẽ
  phân giải đệ quy trường `image` từ JSON trả về.
  - Xem [tài liệu tham khảo jsonpath](https://kubernetes.io/docs/reference/kubectl/jsonpath/)
    để biết thêm thông tin về cách dùng jsonpath.
- Định dạng đầu ra bằng các công cụ tiêu chuẩn: `tr`, `sort`, `uniq`
  - Dùng `tr` để thay khoảng trắng bằng ký tự xuống dòng
  - Dùng `sort` để sắp xếp kết quả
  - Dùng `uniq` để gom và đếm số lượng image

```shell
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec['initContainers', 'containers'][*].image}" |\
tr -s '[[:space:]]' '\n' |\
sort |\
uniq -c
```
Biểu thức jsonpath được diễn giải như sau:

- `.items[*]`: với mỗi giá trị trả về
- `.spec`: lấy phần spec
- `['initContainers', 'containers'][*]`: với mỗi container
- `.image`: lấy image

> **Ghi chú:**
> Khi lấy một Pod đơn lẻ theo tên, ví dụ `kubectl get pod nginx`,
> phần `.items[*]` của biểu thức phải được bỏ đi vì kết quả trả về là một
> Pod đơn lẻ chứ không phải một danh sách các phần tử.

## Liệt kê Container image theo từng Pod (List Container images by Pod)

Bạn có thể kiểm soát định dạng chi tiết hơn bằng cách dùng phép toán `range` để
lặp qua từng phần tử một.

```shell
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}' |\
sort
```

## Liệt kê Container image có lọc theo label của Pod (List Container images filtering by Pod label)

Để chỉ nhắm tới các Pod khớp với một label cụ thể, hãy dùng cờ -l. Lệnh
sau chỉ khớp các Pod có label khớp với `app=nginx`.

```shell
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" -l app=nginx
```

## Liệt kê Container image có lọc theo namespace của Pod (List Container images filtering by Pod namespace)

Để chỉ nhắm tới các Pod trong một namespace cụ thể, hãy dùng cờ namespace. Lệnh
sau chỉ khớp các Pod trong namespace `kube-system`.

```shell
kubectl get pods --namespace kube-system -o jsonpath="{.items[*].spec.containers[*].image}"
```

## Liệt kê Container image dùng go-template thay cho jsonpath (List Container images using a go-template instead of jsonpath)

Thay cho jsonpath, Kubectl hỗ trợ dùng [go-templates](https://pkg.go.dev/text/template)
để định dạng đầu ra:

```shell
kubectl get pods --all-namespaces -o go-template --template="{{range .items}}{{range .spec.containers}}{{.image}} {{end}}{{end}}"
```

## Tiếp theo (What's next)

### Tài liệu tham khảo (Reference)

* Tài liệu tham khảo [Jsonpath](https://kubernetes.io/docs/reference/kubectl/jsonpath/)
* Tài liệu tham khảo [Go template](https://pkg.go.dev/text/template)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 1b:

1. Trên `lab-k8s-master`, bạn chạy nguyên chuỗi lệnh gộp của bài và nhận về một bảng đếm image.
   Phần nào của chuỗi do `kubectl` làm, phần nào do shell làm? Và nếu bỏ `initContainers` khỏi
   biểu thức thì bạn mất những image nào?
2. **Câu bẫy.** Chuỗi lệnh chạy đúng cho cả cluster. Bạn giữ nguyên biểu thức jsonpath đó nhưng
   đổi lệnh thành `kubectl get pod <tên> -n kube-system -o jsonpath=...` cho một Pod cụ thể, và
   không nhận được gì. Vì sao, và phải sửa biểu thức thế nào?
3. `-l app=nginx` và `--namespace kube-system` tác động ở bước nào — trước hay sau khi biểu thức
   jsonpath chạy? Nói cách khác, chúng đổi cái gì trong kết quả cuối cùng?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `kubectl` làm hai việc: **chọn Pod** (`get pods --all-namespaces`) và **trích field**
   (`-o jsonpath=...` lấy ra từng giá trị `image`). Phần còn lại là **shell**: `tr` thay khoảng
   trắng bằng ký tự xuống dòng để mỗi image một dòng, `sort` sắp xếp, `uniq -c` gom và đếm số
   lượng. Nếu bỏ `initContainers` khỏi `['initContainers', 'containers']` thì bạn **mất image
   của mọi init container**, chỉ còn image của container thường — danh sách sẽ thiếu, dù lệnh
   vẫn chạy trơn tru và không báo lỗi.
2. Vì `.items[*]` chỉ tồn tại khi kết quả là **một danh sách**. Lấy một Pod theo tên thì
   `kubectl` trả về **một Pod đơn lẻ**, không có trường `items`, nên biểu thức không khớp được gì.
   Đây là chỗ dễ tưởng jsonpath sai hoặc Pod không có image. Cách sửa đúng như ghi chú của bài:
   **bỏ phần `.items[*]` đi**, để biểu thức bắt đầu thẳng từ `.spec`.
3. Chúng tác động **trước**: cả hai đều là cờ của `kubectl get`, dùng để **thu hẹp tập Pod được
   trả về** — `-l app=nginx` chỉ khớp Pod có label đó, `--namespace kube-system` chỉ lấy Pod
   trong namespace đó. Biểu thức jsonpath chạy **sau**, và chỉ định dạng những gì đã được trả về.
   Vì vậy chúng đổi **tập image xuất hiện trong kết quả**, chứ không đổi cách mỗi image được in
   ra.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của dòng **Thực hành**
nhóm 1b — trả lời xong cả ba câu thì mở
[Lab 1b — Object, label, kubectl và kubeconfig](labs/LAB-1B-OBJECT-LABEL-KUBECTL-VA-KUBECONFIG.md)
và bắt đầu từ phần B0.

# Xác định nguyên nhân Pod bị lỗi (Determine the Reason for Pod Failure)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-application/determine-reason-pod-failure/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 24 — Xử lý sự cố](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố),
bài 8/10 · Giai đoạn này của Phần II không có lab riêng: thực hành trực tiếp trên cluster lab ở mốc
`04-metrics-ready` (xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)), tự chấm bằng Checkpoint ghi ở
cuối mục giai đoạn trong lộ trình.

Bài rất ngắn và rất hẹp: nó chỉ nói về **một kênh duy nhất** — termination message. Đừng đọc nó
như một bài chẩn đoán tổng quát. Giá trị của bài nằm đúng ở chỗ termination message **không phải
là log**: nó là trạng thái cuối cùng của container, nằm trong `status` của Pod, đọc được bằng
`kubectl` mà không cần vào node và vẫn còn đó sau khi container đã chết.

**Phải hiểu ở lần đọc này:**

- Cơ chế ở mục *Ghi và đọc thông điệp kết thúc*: container **ghi vào file** `/dev/termination-log`
  trước khi kết thúc, Kubernetes lấy nội dung đó đưa vào `status` của Pod — cụ thể là
  `lastState.terminated.message`, thấy được bằng `kubectl get pod <tên> --output=yaml`.
- Cách lọc đúng thứ cần: Go template
  `kubectl get pod <tên> -o go-template="{{range .status.containerStatuses}}{{.lastState.terminated.message}}{{end}}"`,
  và với Pod nhiều container thì thêm `.name` vào template để biết **container nào** đang lỗi.
- Mục *Tùy chỉnh thông điệp kết thúc*: `terminationMessagePath` mặc định là `/dev/termination-log`,
  đổi được sang file khác, nhưng **không đổi được sau khi Pod đã khởi chạy**. Kubernetes dùng nội
  dung file đó để điền status message **cho cả trường hợp thành công lẫn thất bại**.
- Giới hạn kích thước, và đó là lý do bài nói thông điệp chỉ nên chứa **trạng thái cuối cùng ngắn
  gọn**: kubelet cắt bớt thông điệp dài hơn **4096 byte**; tổng của mọi container bị giới hạn
  **12KiB chia đều** — 12 container thì mỗi container còn 1024 byte.
- `terminationMessagePolicy` mặc định là **`File`**, nghĩa là **chỉ** đọc từ file thông điệp kết
  thúc. Đặt **`FallbackToLogsOnError`** thì khi file rỗng **và** container thoát với lỗi,
  Kubernetes lấy đoạn cuối log container thay thế — phần lấy bị giới hạn 2048 byte hoặc 80 dòng,
  tùy giá trị nào nhỏ hơn.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* (minikube và các playground) | cluster lab đã sẵn sàng ở mốc `04-metrics-ready` với đủ hai worker không phải control plane | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Cú pháp Go template của `-o go-template` | là kỹ năng công cụ `kubectl`, không phải cơ chế Kubernetes; chép nguyên lệnh mẫu rồi thay tên Pod là đủ | Checkpoint [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố) — dùng lại đúng hai lệnh mẫu của bài |
| Danh sách *Tiếp theo* (ImagePullBackOff, Pod phase, trạng thái container) | là bảng chỉ đường sang bài khác | bài [40](40-images-vi.md) ở [giai đoạn 2](00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime) và bài [47](47-pod-lifecycle-vi.md) ở [giai đoạn 3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời) |

---

Trang này hướng dẫn cách ghi và đọc thông điệp kết thúc (termination message) của một Container.

Thông điệp kết thúc cung cấp một cách để container ghi thông tin về các sự kiện nghiêm trọng
(fatal event) vào một vị trí mà các công cụ như dashboard và phần mềm giám sát (monitoring) có
thể dễ dàng truy xuất và hiển thị. Trong hầu hết các trường hợp, thông tin bạn đưa vào thông
điệp kết thúc cũng nên được ghi vào
[log chung của Kubernetes](158-logging-vi.md).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Ghi và đọc thông điệp kết thúc (Writing and reading a termination message)

Trong bài thực hành này, bạn tạo một Pod chạy một container. Manifest của Pod đó chỉ định một
lệnh sẽ chạy khi container khởi động:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: termination-demo
spec:
  containers:
  - name: termination-demo-container
    image: debian
    command: ["/bin/sh"]
    args: ["-c", "sleep 10 && echo Sleep expired > /dev/termination-log"]
```

1. Tạo Pod dựa trên file cấu hình YAML:

    ```shell
    kubectl apply -f https://k8s.io/examples/debug/termination.yaml
    ```

    Trong file YAML, ở các trường `command` và `args`, bạn có thể thấy container ngủ (sleep)
    trong 10 giây rồi ghi "Sleep expired" vào file `/dev/termination-log`. Sau khi container
    ghi xong thông điệp "Sleep expired", nó kết thúc.

1. Hiển thị thông tin về Pod:

    ```shell
    kubectl get pod termination-demo
    ```

    Lặp lại lệnh trên cho đến khi Pod không còn chạy nữa.

1. Hiển thị thông tin chi tiết về Pod:

    ```shell
    kubectl get pod termination-demo --output=yaml
    ```

    Kết quả đầu ra bao gồm thông điệp "Sleep expired":

    ```yaml
    apiVersion: v1
    kind: Pod
    ...
        lastState:
          terminated:
            containerID: ...
            exitCode: 0
            finishedAt: ...
            message: |
              Sleep expired
            ...
    ```

1. Dùng một Go template để lọc kết quả đầu ra sao cho chỉ còn thông điệp kết thúc:

    ```shell
    kubectl get pod termination-demo -o go-template="{{range .status.containerStatuses}}{{.lastState.terminated.message}}{{end}}"
    ```

Nếu bạn đang chạy một Pod có nhiều container, bạn có thể dùng Go template để kèm theo tên của
container. Bằng cách đó, bạn có thể phát hiện container nào đang bị lỗi:

```shell
kubectl get pod multi-container-pod -o go-template='{{range .status.containerStatuses}}{{printf "%s:\n%s\n\n" .name .lastState.terminated.message}}{{end}}'
```

## Tùy chỉnh thông điệp kết thúc (Customizing the termination message) {#customizing-the-termination-message}

Kubernetes truy xuất thông điệp kết thúc từ file thông điệp kết thúc được chỉ định trong trường
`terminationMessagePath` của một Container, với giá trị mặc định là `/dev/termination-log`.
Bằng cách tùy chỉnh trường này, bạn có thể yêu cầu Kubernetes dùng một file khác. Kubernetes
dùng nội dung của file được chỉ định để điền vào thông điệp trạng thái (status message) của
Container trong cả trường hợp thành công lẫn thất bại.

Thông điệp kết thúc được thiết kế để chứa trạng thái cuối cùng ngắn gọn, ví dụ như một thông
báo lỗi assertion. Kubelet cắt bớt (truncate) các thông điệp dài hơn 4096 byte.

Tổng độ dài thông điệp của tất cả các container bị giới hạn ở 12KiB, chia đều cho mỗi
container. Ví dụ, nếu có 12 container (`initContainers` hoặc `containers`), mỗi container có
1024 byte không gian dành cho thông điệp kết thúc.

Đường dẫn thông điệp kết thúc mặc định là `/dev/termination-log`. Bạn không thể thay đổi đường
dẫn thông điệp kết thúc sau khi Pod đã được khởi chạy.

Trong ví dụ sau, container ghi thông điệp kết thúc vào `/tmp/my-log` để Kubernetes truy xuất:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: msg-path-demo
spec:
  containers:
  - name: msg-path-demo-container
    image: debian
    terminationMessagePath: "/tmp/my-log"
```

Hơn nữa, người dùng có thể thiết lập trường `terminationMessagePolicy` của một Container để
tùy chỉnh sâu hơn. Trường này mặc định là "`File`", nghĩa là thông điệp kết thúc chỉ được truy
xuất từ file thông điệp kết thúc. Bằng cách đặt `terminationMessagePolicy` thành
"`FallbackToLogsOnError`", bạn có thể yêu cầu Kubernetes dùng đoạn cuối cùng của log container
nếu file thông điệp kết thúc rỗng và container thoát với lỗi. Phần log được lấy bị giới hạn ở
2048 byte hoặc 80 dòng, tùy theo giá trị nào nhỏ hơn.

## Tiếp theo (What's next)

* Xem trường `terminationMessagePath` trong
  [Container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#container-v1-core).
* Xem [ImagePullBackOff](40-images-vi.md#imagepullbackoff)
  trong [Images](40-images-vi.md).
* Tìm hiểu về [truy xuất log](158-logging-vi.md).
* Tìm hiểu về [Go templates](https://pkg.go.dev/text/template).
* Tìm hiểu về [trạng thái Pod](298-debug-init-containers-vi.md#understanding-pod-status)
  và [Pod phase](47-pod-lifecycle-vi.md#pod-phase).
* Tìm hiểu về [trạng thái container](47-pod-lifecycle-vi.md#container-states).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 24:

1. **Câu bẫy.** Container của bạn thoát với exit code 1 và đã in nguyên nhân lỗi ra stdout. Nhưng
   `lastState.terminated.message` lại rỗng. Vì sao, và sửa bằng trường nào?
2. Bạn chạy Pod `termination-demo` trên cluster lab và nó được đặt lên `lab-k8s-worker2`. Pod đã
   kết thúc, không còn chạy nữa. Bạn lấy thông điệp kết thúc bằng lệnh nào, và vì sao không cần
   SSH vào `lab-k8s-worker2` để đọc file?
3. Pod đang chạy, bạn muốn đổi `terminationMessagePath` từ `/dev/termination-log` sang
   `/tmp/my-log`. Làm được không?
4. Ứng dụng ghi một stack trace 20 KB vào `/dev/termination-log`. Bạn đọc lại được bao nhiêu, và
   bài khuyên đưa phần thông tin còn lại đi đâu?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì `terminationMessagePolicy` **mặc định là `File`**: Kubernetes **chỉ** truy xuất thông điệp
   kết thúc **từ file thông điệp kết thúc**, mặc định là `/dev/termination-log`. Ứng dụng in ra
   stdout thì file đó vẫn rỗng. Sửa bằng cách đặt **`terminationMessagePolicy:
   FallbackToLogsOnError`** cho container — khi file rỗng **và** container thoát với lỗi,
   Kubernetes sẽ lấy đoạn cuối log container (giới hạn 2048 byte hoặc 80 dòng, tùy giá trị nào nhỏ
   hơn).
2. Bằng **`kubectl get pod termination-demo --output=yaml`** rồi đọc
   `status.containerStatuses[].lastState.terminated.message`, hoặc lọc gọn bằng
   **`kubectl get pod termination-demo -o go-template="{{range .status.containerStatuses}}{{.lastState.terminated.message}}{{end}}"`**.
   Không cần vào node vì **Kubernetes đã đọc nội dung file đó và điền vào status message của
   Container**: từ lúc ấy thông điệp là một phần của đối tượng Pod trên API server, không còn nằm
   trên đĩa của worker nữa. Đó chính là điểm khiến bài nói termination message dễ được các công cụ
   như dashboard và phần mềm giám sát truy xuất và hiển thị.
3. **Không.** Bài nói thẳng: bạn **không thể thay đổi đường dẫn thông điệp kết thúc sau khi Pod đã
   được khởi chạy**. Muốn dùng `/tmp/my-log` thì phải khai `terminationMessagePath` ngay trong spec
   container và tạo lại Pod.
4. **Khoảng 4096 byte** — kubelet **cắt bớt (truncate)** các thông điệp dài hơn 4096 byte, và nếu
   Pod có nhiều container thì còn bị chia nhỏ hơn nữa vì tổng của mọi container bị giới hạn 12KiB
   chia đều. Phần còn lại phải đi đường khác: bài nói ngay đoạn mở đầu rằng trong hầu hết trường
   hợp, thông tin bạn đưa vào thông điệp kết thúc **cũng nên được ghi vào
   [log chung của Kubernetes](158-logging-vi.md)**. Termination message chỉ để chứa **trạng thái
   cuối cùng ngắn gọn**, ví dụ một thông báo lỗi assertion.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

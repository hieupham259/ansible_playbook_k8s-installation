# Định nghĩa biến môi trường cho một Container (Define Environment Variables for a Container)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ [nhóm 3b — Cấu hình ứng dụng](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod),
bài 9/12 · Kiểm chứng ở [Lab 3b — Cấu hình ứng dụng](labs/LAB-3B-CAU-HINH-UNG-DUNG.md), `phần B2.2`
(`envFrom` và `prefix`) và `phần B4.3` (`env` ghi đè biến của image, `$(VAR)` trong cấu hình).

Bài ngắn nhưng là bài **gốc** của ba bài liên tiếp về biến môi trường: bài này đặt luật chung,
[332](332-define-env-via-file-vi.md) thêm một nguồn giá trị mới, còn
[333](333-interdependent-env-variables-vi.md) đào sâu đúng một ghi chú của bài này — ghi chú về thứ
tự tham chiếu. Vì vậy đừng bỏ qua hai ghi chú ở cuối mục đầu tiên, chúng là bản lề sang hai bài sau.

**Phải hiểu ở lần đọc này:**

- `env` và `envFrom` **không thay thế nhau**: `env` đặt từng biến bạn tự đặt tên kèm giá trị viết
  thẳng; `envFrom` nạp **toàn bộ** cặp key-value của một ConfigMap hoặc Secret, và có thể gắn một
  chuỗi `prefix` chung.
- Với `envFrom`, **tên biến chính là key** trong ConfigMap hoặc Secret — bạn không đặt tên biến.
- Biến đặt bằng `env` hoặc `envFrom` **ghi đè** mọi biến môi trường đã có trong container image
  (ghi chú thứ nhất). Đây là cách đổi hành vi của image mà không build lại image.
- Biến khai ở `.spec.containers[*].env[*]` còn **dùng lại được ở chỗ khác trong chính cấu hình
  Pod** — mục *Sử dụng biến môi trường bên trong cấu hình của bạn*: `MESSAGE` ghép ba biến đứng
  trước nó, rồi `args: ["$(MESSAGE)"]` khiến container chạy
  `echo Warm greetings to The Most Honorable Kubernetes`.
- Thứ tự trong danh sách `env` là ràng buộc thật: biến dùng biến khác **phải đứng sau** biến nó
  tham chiếu, và phải tránh tham chiếu vòng (ghi chú thứ hai). Kiểm chứng biến thật trong container
  bằng `kubectl exec <pod> -- printenv`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Link "đọc thêm về ConfigMap" ở cuối phần định nghĩa `envFrom` | ở đây chỉ cần biết `envFrom` tồn tại và khác `env` ở điểm nào | bài [275](275-configure-pod-configmap-vi.md) cùng nhóm 3b, kiểm chứng ở `phần B2.2` của [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) |
| Link "đọc thêm về Secret" ngay cạnh đó | Secret là nửa sau của nhóm 3b, chưa động tới ở bài này | bài [334](334-distribute-credentials-secure-vi.md), bài cuối nhóm 3b |
| Điều gì xảy ra khi `$(VAR)` trỏ tới biến chưa định nghĩa, và cách thoát bằng `$$(VAR)` | bài này chỉ nêu quy tắc thứ tự trong một ghi chú, không nói kết quả | bài [333](333-interdependent-env-variables-vi.md) cùng nhóm 3b |
| Link `EnvVarSource` và các nguồn `valueFrom` khác trong mục *Tiếp theo* | tra cứu API, không phải nội dung bài | bài [336](336-env-variable-expose-pod-info-vi.md) — Downward API, đã đọc ở [nhóm 3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời) |

---

Trang này chỉ cách định nghĩa các biến môi trường (environment variable) cho một container
trong một Pod của Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Định nghĩa một biến môi trường cho một container (Define an environment variable for a container)

Khi bạn tạo một Pod, bạn có thể thiết lập các biến môi trường cho các container chạy trong
Pod đó. Để thiết lập biến môi trường, hãy đưa field `env` hoặc `envFrom` vào file cấu hình.

Hai field `env` và `envFrom` có tác dụng khác nhau.

`env`
: cho phép bạn thiết lập các biến môi trường cho một container, chỉ định trực tiếp giá trị cho từng biến mà bạn đặt tên.

`envFrom`
: cho phép bạn thiết lập các biến môi trường cho một container bằng cách tham chiếu tới một ConfigMap hoặc một Secret.
 Khi bạn dùng `envFrom`, tất cả các cặp key-value trong ConfigMap hoặc Secret được tham chiếu
 sẽ được thiết lập thành biến môi trường cho container.
 Bạn cũng có thể chỉ định một chuỗi tiền tố (prefix) chung.

Bạn có thể đọc thêm về [ConfigMap](275-configure-pod-configmap-vi.md#configure-all-key-value-pairs-in-a-configmap-as-container-environment-variables)
và [Secret](334-distribute-credentials-secure-vi.md#configure-all-key-value-pairs-in-a-secret-as-container-environment-variables).

Trang này giải thích cách dùng `env`.

Trong bài thực hành này, bạn tạo một Pod chạy một container. File cấu hình của Pod định nghĩa
một biến môi trường có tên `DEMO_GREETING` với giá trị `"Hello from the environment"`. Dưới đây
là manifest cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envar-demo
  labels:
    purpose: demonstrate-envars
spec:
  containers:
  - name: envar-demo-container
    image: gcr.io/google-samples/hello-app:2.0
    env:
    - name: DEMO_GREETING
      value: "Hello from the environment"
    - name: DEMO_FAREWELL
      value: "Such a sweet sorrow"
```

1. Tạo một Pod dựa trên manifest đó:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/inject/envars.yaml
   ```

1. Liệt kê các Pod đang chạy:

   ```shell
   kubectl get pods -l purpose=demonstrate-envars
   ```

   Kết quả tương tự như sau:

   ```
   NAME            READY     STATUS    RESTARTS   AGE
   envar-demo      1/1       Running   0          9s
   ```

1. Liệt kê các biến môi trường của container trong Pod:

   ```shell
   kubectl exec envar-demo -- printenv
   ```

   Kết quả tương tự như sau:

   ```
   NODE_VERSION=4.4.2
   EXAMPLE_SERVICE_PORT_8080_TCP_ADDR=10.3.245.237
   HOSTNAME=envar-demo
   ...
   DEMO_GREETING=Hello from the environment
   DEMO_FAREWELL=Such a sweet sorrow
   ```

> **Ghi chú:** Các biến môi trường được thiết lập bằng field `env` hoặc `envFrom`
> sẽ ghi đè mọi biến môi trường được chỉ định trong container image.

> **Ghi chú:** Các biến môi trường có thể tham chiếu lẫn nhau, tuy nhiên thứ tự rất quan trọng.
> Biến nào sử dụng các biến khác được định nghĩa trong cùng ngữ cảnh thì phải đứng sau trong
> danh sách. Tương tự, hãy tránh tham chiếu vòng (circular reference).

## Sử dụng biến môi trường bên trong cấu hình của bạn (Using environment variables inside of your config) {#using-environment-variables-inside-of-your-config}

Các biến môi trường mà bạn định nghĩa trong cấu hình của một Pod tại
`.spec.containers[*].env[*]` có thể được dùng ở những chỗ khác trong cấu hình, ví dụ trong
các command và argument mà bạn thiết lập cho các container của Pod.
Trong cấu hình ví dụ dưới đây, các biến môi trường `GREETING`, `HONORIFIC` và `NAME` lần lượt
được gán các giá trị `Warm greetings to`, `The Most Honorable` và `Kubernetes`. Biến môi trường
`MESSAGE` gộp tất cả các biến môi trường này lại rồi dùng nó làm argument CLI truyền vào
container `env-print-demo`.

Tên biến môi trường có thể gồm bất kỳ [ký tự ASCII in được (printable ASCII characters)](https://www.ascii-code.com/characters/printable-characters) nào ngoại trừ '='.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: print-greeting
spec:
  containers:
  - name: env-print-demo
    image: bash
    env:
    - name: GREETING
      value: "Warm greetings to"
    - name: HONORIFIC
      value: "The Most Honorable"
    - name: NAME
      value: "Kubernetes"
    - name: MESSAGE
      value: "$(GREETING) $(HONORIFIC) $(NAME)"
    command: ["echo"]
    args: ["$(MESSAGE)"]
```

Khi Pod được tạo, lệnh `echo Warm greetings to The Most Honorable Kubernetes` sẽ được chạy trong container.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [biến môi trường](336-env-variable-expose-pod-info-vi.md).
* Tìm hiểu về [sử dụng Secret làm biến môi trường](109-secret-vi.md#using-secrets-as-environment-variables).
* Xem [EnvVarSource](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#envvarsource-v1-core).

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Trường hợp A: bạn cần đặt đúng ba biến, mỗi biến một giá trị viết thẳng trong manifest. Trường
   hợp B: bạn cần đặt **tất cả** các key của một ConfigMap thành biến. Mỗi trường hợp dùng field
   nào, và trong trường hợp B thì **tên biến môi trường lấy từ đâu**?
2. **Câu bẫy.** Container image đã có sẵn một biến môi trường, ví dụ `PATH`. Manifest của bạn khai
   một mục `env` trùng tên với giá trị khác. Trong container đang chạy, giá trị nào có hiệu lực?
3. Trong Pod `print-greeting`, `command` là `["echo"]` và `args` là `["$(MESSAGE)"]`. Ai khai triển
   `$(MESSAGE)`, và lệnh thực sự chạy trong container là gì? Nếu bạn đảo `MESSAGE` lên **đầu** danh
   sách `env` thì bài nói điều gì về tham chiếu đó?
4. Trên cluster lab, bạn apply Pod `envar-demo` từ `lab-k8s-master`. Lệnh nào trong bài cho phép
   khẳng định `DEMO_GREETING` và `DEMO_FAREWELL` thật sự tồn tại **bên trong** container chứ không
   chỉ tồn tại trong manifest? Muốn chứng minh thêm điều ở câu 2, bạn phải thêm gì vào phép kiểm
   chứng đó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Trường hợp A dùng **`env`**: mỗi biến một mục, bạn tự đặt `name` và `value`. Trường hợp B dùng
   **`envFrom`** trỏ tới ConfigMap. Ở B, **tên biến chính là key trong ConfigMap** — bạn không đặt
   tên biến, và cũng không chọn lẻ được: bài nói rõ khi dùng `envFrom` thì *tất cả* cặp key-value
   trong ConfigMap hoặc Secret được thiết lập thành biến môi trường. Nếu cần phân biệt nguồn, gắn
   thêm một chuỗi `prefix` chung.
2. **Giá trị trong manifest thắng.** Ghi chú thứ nhất của bài phát biểu thẳng chiều ghi đè: biến
   thiết lập bằng `env` hoặc `envFrom` **ghi đè mọi biến môi trường được chỉ định trong container
   image**. Trực giác "image là thứ dựng nên tiến trình nên image thắng" là sai — chiều ghi đè
   không phụ thuộc vào việc image định nghĩa biến đó kỹ tới đâu.
3. **Kubernetes khai triển, không phải shell trong container.** Bài nói biến khai ở
   `.spec.containers[*].env[*]` dùng được ở những chỗ khác trong cấu hình, ví dụ trong `command` và
   `args` của container. Lệnh thực chạy là
   `echo Warm greetings to The Most Honorable Kubernetes`. Nếu `MESSAGE` bị đưa lên đầu danh sách
   thì nó tham chiếu ba biến đứng **sau** nó, vi phạm đúng ghi chú thứ hai: biến nào dùng biến khác
   trong cùng ngữ cảnh thì phải đứng sau trong danh sách.
4. `kubectl exec envar-demo -- printenv` — liệt kê biến môi trường thật của container, và bài in
   sẵn kết quả mẫu có hai dòng `DEMO_GREETING=` và `DEMO_FAREWELL=`. Để chứng minh câu 2 bạn cần
   **một Pod đối chứng**: cùng image, cùng mọi thứ, chỉ **bỏ khối `env`** đi, rồi so hai output.
   Một Pod duy nhất không cho biết giá trị đó đến từ manifest hay vốn có sẵn trong image.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

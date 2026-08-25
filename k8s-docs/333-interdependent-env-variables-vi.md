# Định nghĩa các biến môi trường phụ thuộc (Define Dependent Environment Variables)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/inject-data-application/define-interdependent-environment-variables/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ [nhóm 3b — Cấu hình ứng dụng](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod),
bài 11/12 · Kiểm chứng ở [Lab 3b — Cấu hình ứng dụng](labs/LAB-3B-CAU-HINH-UNG-DUNG.md), `phần B4.4`
(thứ tự trong danh sách `env` và cách thoát `$$(VAR)`).

Bài rất ngắn và chỉ có **một** manifest, nhưng manifest đó là toàn bộ bài học: ba biến
`UNCHANGED_REFERENCE`, `SERVICE_ADDRESS`, `ESCAPED_REFERENCE` có `value` gần như giống hệt nhau mà
cho ba kết quả khác nhau. Đây là chỗ trả lời chi tiết cho ghi chú về thứ tự tham chiếu mà
[331](331-define-environment-variable-vi.md) mới chỉ nêu ra.

**Phải hiểu ở lần đọc này:**

- Cú pháp tham chiếu là `$(VAR_NAME)` đặt trong `value` của một mục `env` — đó là cách khai một
  biến môi trường phụ thuộc.
- **Thứ tự trong danh sách `env` quyết định**: một biến chỉ được coi là "đã định nghĩa" khi nó đứng
  **trước** chỗ tham chiếu. `SERVICE_ADDRESS` phân giải được vì `PROTOCOL` nằm phía trên;
  `UNCHANGED_REFERENCE` thì không, vì nó tham chiếu `PROTOCOL` nằm phía dưới.
- Tham chiếu không phân giải được **không phải lỗi**: biến chưa định nghĩa được xử lý như một chuỗi
  bình thường, và bài nói rõ biến môi trường phân tích cú pháp sai **không ngăn container khởi
  động** — bạn chỉ phát hiện qua `kubectl logs`.
- `$$(VAR_NAME)` là cú pháp **thoát**: tham chiếu đã thoát **không bao giờ** được khai triển, bất
  kể biến được tham chiếu có được định nghĩa hay không. `ESCAPED_REFERENCE` cho output giống
  `UNCHANGED_REFERENCE` nhưng vì lý do hoàn toàn khác.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cách lấy giá trị biến từ ConfigMap hoặc Secret thay vì viết thẳng vào `value` | bài này chỉ dùng `value` cố định để cô lập đúng chuyện thứ tự | bài [275](275-configure-pod-configmap-vi.md) và [334](334-distribute-credentials-secure-vi.md) cùng nhóm 3b |
| Dùng `$(VAR)` trong `command` và `args` thay vì trong `value` của một biến khác | cùng cú pháp nhưng khác chỗ đặt; bài không bàn tới | bài [330](330-define-command-argument-vi.md) cùng nhóm 3b, kiểm chứng ở `phần B4.2` của [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) |
| Địa chỉ `172.17.0.1` và image `busybox:1.28` trong manifest | chỉ là giá trị minh họa của trang gốc, không liên quan tới mạng cluster lab | `phần B4.4` của [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) chạy lại ví dụ này bằng image của lab |

---

Trang này chỉ cách định nghĩa các biến môi trường phụ thuộc (dependent environment variable)
cho một container trong một Pod của Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Định nghĩa một biến môi trường phụ thuộc cho một container (Define an environment dependent variable for a container)

Khi bạn tạo một Pod, bạn có thể thiết lập các biến môi trường phụ thuộc cho các container chạy
trong Pod đó. Để thiết lập biến môi trường phụ thuộc, bạn có thể dùng cú pháp $(VAR_NAME)
trong `value` của `env` trong file cấu hình.

Trong bài thực hành này, bạn tạo một Pod chạy một container. File cấu hình của Pod định nghĩa
một biến môi trường phụ thuộc theo cách dùng phổ biến. Dưới đây là manifest cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dependent-envars-demo
spec:
  containers:
    - name: dependent-envars-demo
      args:
        - while true; do echo -en '\n'; printf UNCHANGED_REFERENCE=$UNCHANGED_REFERENCE'\n'; printf SERVICE_ADDRESS=$SERVICE_ADDRESS'\n';printf ESCAPED_REFERENCE=$ESCAPED_REFERENCE'\n'; sleep 30; done;
      command:
        - sh
        - -c
      image: busybox:1.28
      env:
        - name: SERVICE_PORT
          value: "80"
        - name: SERVICE_IP
          value: "172.17.0.1"
        - name: UNCHANGED_REFERENCE
          value: "$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"
        - name: PROTOCOL
          value: "https"
        - name: SERVICE_ADDRESS
          value: "$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"
        - name: ESCAPED_REFERENCE
          value: "$$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"
```

1. Tạo một Pod dựa trên manifest đó:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/inject/dependent-envars.yaml
   ```
   ```
   pod/dependent-envars-demo created
   ```

2. Liệt kê các Pod đang chạy:

   ```shell
   kubectl get pods dependent-envars-demo
   ```
   ```
   NAME                      READY     STATUS    RESTARTS   AGE
   dependent-envars-demo     1/1       Running   0          9s
   ```

3. Kiểm tra log của container đang chạy trong Pod của bạn:

   ```shell
   kubectl logs pod/dependent-envars-demo
   ```
   ```

   UNCHANGED_REFERENCE=$(PROTOCOL)://172.17.0.1:80
   SERVICE_ADDRESS=https://172.17.0.1:80
   ESCAPED_REFERENCE=$(PROTOCOL)://172.17.0.1:80
   ```

Như trên, bạn đã định nghĩa một tham chiếu phụ thuộc đúng ở `SERVICE_ADDRESS`, một tham chiếu
phụ thuộc sai ở `UNCHANGED_REFERENCE`, và bỏ qua tham chiếu phụ thuộc ở `ESCAPED_REFERENCE`.

Khi một biến môi trường đã được định nghĩa từ trước lúc nó được tham chiếu, tham chiếu đó sẽ
được phân giải (resolve) đúng, như trong trường hợp `SERVICE_ADDRESS`.

Lưu ý rằng thứ tự trong danh sách `env` rất quan trọng. Một biến môi trường không được coi là
"đã định nghĩa" nếu nó nằm phía dưới trong danh sách. Đó là lý do `UNCHANGED_REFERENCE` không
phân giải được `$(PROTOCOL)` trong ví dụ trên.

Khi biến môi trường chưa được định nghĩa hoặc chỉ có một phần các biến được định nghĩa, biến
môi trường chưa định nghĩa sẽ được xử lý như một chuỗi bình thường, như trường hợp
`UNCHANGED_REFERENCE`. Lưu ý rằng nói chung, các biến môi trường được phân tích cú pháp sai sẽ
không ngăn container khởi động.

Cú pháp `$(VAR_NAME)` có thể được thoát (escape) bằng hai dấu `$`, tức là: `$$(VAR_NAME)`.
Tham chiếu đã thoát không bao giờ được khai triển (expand), bất kể biến được tham chiếu có
được định nghĩa hay không. Điều này thể hiện qua trường hợp `ESCAPED_REFERENCE` ở trên.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [biến môi trường](336-env-variable-expose-pod-info-vi.md).
* Xem [EnvVarSource](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#envvarsource-v1-core).

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. `UNCHANGED_REFERENCE` và `SERVICE_ADDRESS` có `value` giống hệt nhau, chỉ khác vị trí trong danh
   sách `env`. Vì sao một biến phân giải được `$(PROTOCOL)` còn biến kia thì không?
2. **Câu bẫy.** `UNCHANGED_REFERENCE` và `ESCAPED_REFERENCE` in ra hai chuỗi giống hệt nhau. Vậy
   hai khai báo đó tương đương nhau chứ? Nếu bạn chuyển `PROTOCOL` lên **đầu** danh sách `env` thì
   output của từng biến thay đổi thế nào?
3. Bạn apply manifest này từ `lab-k8s-master` và `kubectl get pods dependent-envars-demo` báo
   `Running`. Điều đó có chứng minh mọi tham chiếu đều đúng không? Bạn phát hiện tham chiếu hỏng
   bằng cách nào?
4. Bạn muốn `UNCHANGED_REFERENCE` cho ra `https://172.17.0.1:80`. Sửa gì trong manifest, và giá trị
   `value` của nó có phải đổi một ký tự nào không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì **thứ tự trong danh sách `env` là yếu tố quyết định**. Một biến môi trường chỉ được coi là
   "đã định nghĩa" khi nó nằm **phía trên** chỗ tham chiếu. `PROTOCOL` được khai **sau**
   `UNCHANGED_REFERENCE` nhưng **trước** `SERVICE_ADDRESS`, nên chỉ `SERVICE_ADDRESS` phân giải
   được. Ở `UNCHANGED_REFERENCE`, `$(PROTOCOL)` được giữ lại như một chuỗi bình thường.
2. **Không tương đương** — kết quả trùng nhau chỉ là trùng hợp, hai nguyên nhân khác hẳn.
   `UNCHANGED_REFERENCE` giữ nguyên `$(PROTOCOL)` vì biến đó **chưa được định nghĩa tại thời điểm
   tham chiếu**; `ESCAPED_REFERENCE` giữ nguyên vì `$$(VAR_NAME)` là **cú pháp thoát**, và tham
   chiếu đã thoát **không bao giờ được khai triển**, bất kể biến có tồn tại hay không. Bằng chứng
   phân biệt: chuyển `PROTOCOL` lên đầu danh sách thì `UNCHANGED_REFERENCE` **đổi** thành
   `https://172.17.0.1:80`, còn `ESCAPED_REFERENCE` **vẫn** in ra `$(PROTOCOL)://172.17.0.1:80`.
3. **Không.** Bài nói rõ: nói chung, các biến môi trường được phân tích cú pháp sai **không ngăn
   container khởi động**. Pod vẫn `Running`, biến hỏng vẫn được truyền vào dưới dạng chuỗi thô. Chỗ
   duy nhất nhìn thấy sự khác biệt trong bài là output của container: `kubectl logs
   pod/dependent-envars-demo`, nơi ba biến được in ra để so.
4. Chỉ cần **di chuyển mục `PROTOCOL` lên trước `UNCHANGED_REFERENCE`** trong danh sách `env`.
   Chuỗi `value` **không đổi một ký tự nào** — `"$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"` đã
   đúng sẵn, đúng bằng chuỗi mà `SERVICE_ADDRESS` đang dùng. Vấn đề nằm ở vị trí, không nằm ở cú
   pháp.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

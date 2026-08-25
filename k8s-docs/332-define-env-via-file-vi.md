# Định nghĩa giá trị biến môi trường bằng một Init Container (Define Environment Variable Values Using An Init Container)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-via-file/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ [nhóm 3b — Cấu hình ứng dụng](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod),
bài 10/12 · Kiểm chứng ở [Lab 3b — Cấu hình ứng dụng](labs/LAB-3B-CAU-HINH-UNG-DUNG.md), `phần B5`
(`fileKeyRef`, `emptyDir` và cú pháp file env).

Đây là bài duy nhất trong nhóm nói về một tính năng còn ở trạng thái **beta**, và bài cũng đòi
phiên bản server tối thiểu. Con số phiên bản trong bài là của trang gốc; phiên bản cluster lab đang
khóa nằm ở [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) — đối
chiếu ở đó trước khi thử, đừng lấy số trong bài. Bài đứng ngay sau
[331](331-define-environment-variable-vi.md): 331 cho biến **giá trị viết thẳng**, bài này thêm
**một nguồn giá trị mới** cho cùng field `env`.

**Phải hiểu ở lần đọc này:**

- Kiến trúc trong mục *Cách thiết kế này hoạt động*: một `initContainer` mount volume `emptyDir` và
  **ghi file env** vào đó; container tiêu thụ **không mount volume này**, nó chỉ trỏ tới file bằng
  `fileKeyRef` với ba thông tin `volumeName` + `path` + `key`.
- Kubelet lấy giá trị từ file **trong quá trình khởi tạo container**, rồi cung cấp cho container
  như một biến môi trường bình thường.
- `optional: false` nghĩa là `key` chỉ định trong `fileKeyRef` **bắt buộc phải tồn tại** trong file
  biến môi trường.
- Định dạng file env là **tập con chặt** của bash tuân thủ POSIX (mục *Cú pháp file env*): giá trị
  bắt buộc nằm trong nháy đơn và được giữ nguyên văn, hỗ trợ giá trị nhiều dòng, dòng bắt đầu bằng
  `#` là chú thích còn `#` nằm trong nháy đơn thì không. Bị **cấm**: `VAR=value`, `VAR="value"`,
  hai chuỗi nháy liền nhau, và mọi hình thức nội suy hay nối chuỗi (`$OTHER`, `${OTHER}`).
- Ranh giới an toàn mà bài nói thẳng trong ghi chú: `emptyDir` **không** có các cơ chế bảo vệ của
  đối tượng Secret, nên đưa biến môi trường bí mật vào container theo đường này **không** phải một
  thực hành bảo mật tốt.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ghi chú "mọi loại container đều hỗ trợ nạp biến từ file", gồm sidecar và ephemeral container | bài chỉ dựng đúng một đường: init container ghi → container thường đọc | bài [51](51-sidecar-containers-vi.md) và [52](52-ephemeral-containers-vi.md) đã đọc ở [nhóm 3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời) |
| Đường **đúng** để đưa dữ liệu nhạy cảm vào container | bài này chỉ cảnh báo rằng `emptyDir` không phải chỗ đó, không chỉ chỗ thay thế | bài [334](334-distribute-credentials-secure-vi.md) cuối nhóm 3b, kiểm chứng ở `phần B8` của [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) |
| Bản thân volume `emptyDir`: vòng đời, nơi lưu, giới hạn dung lượng | ở đây nó chỉ đóng vai chỗ trung chuyển một file | bài [91](91-volumes-vi.md) ở [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]` (được bật mặc định: true)

Trang này chỉ cách cấu hình biến môi trường (environment variable) cho các container trong
một Pod thông qua file.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn v1.34. Để kiểm tra phiên bản, nhập
`kubectl version`.

## Cách thiết kế này hoạt động (How the design works)

Trong bài thực hành này, bạn sẽ tạo một Pod lấy biến môi trường từ các file, rồi đưa các giá
trị này vào container đang chạy.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfile-test-pod
spec:
  initContainers:
    - name: setup-envfile
      image:  nginx
      command: ['sh', '-c', "echo \"DB_ADDRESS=\'address\'\nREST_ENDPOINT=\'endpoint\'\" > /data/config.env"]
      volumeMounts:
        - name: config
          mountPath: /data
  containers:
    - name: use-envfile
      image: nginx
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: DB_ADDRESS
          valueFrom:
            fileKeyRef:
              path: config.env
              volumeName: config
              key: DB_ADDRESS
              optional: false
  restartPolicy: Never
  volumes:
    - name: config
      emptyDir: {}
```

Trong manifest này, bạn có thể thấy `initContainer` mount một volume `emptyDir` và ghi các
biến môi trường vào một file bên trong volume đó, còn các container thường thì tham chiếu tới
cả file lẫn key của biến môi trường thông qua field `fileKeyRef` mà không cần mount volume.
Khi field `optional` được đặt là false, `key` được chỉ định trong `fileKeyRef` bắt buộc phải
tồn tại trong file biến môi trường.

Volume chỉ được mount vào container ghi file (`initContainer`), còn container tiêu thụ
(consumer) — container sử dụng biến môi trường — sẽ không được mount volume này.

Định dạng file env tuân theo [chuẩn file env của Kubernetes](#env-file-syntax).

Trong quá trình khởi tạo container, kubelet lấy các biến môi trường từ các file được chỉ định
trong volume `emptyDir` và cung cấp chúng cho container.

> **Ghi chú:** Tất cả các loại container (initContainers, container thường, sidecar container
> và ephemeral container) đều hỗ trợ nạp biến môi trường từ file.
>
> Mặc dù các biến môi trường này có thể chứa thông tin nhạy cảm, volume `emptyDir` không cung
> cấp các cơ chế bảo vệ như đối tượng Secret chuyên dụng. Do đó, việc cung cấp các biến môi
> trường bí mật cho container thông qua tính năng này không được coi là một thực hành bảo mật
> tốt (security best practice).

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/inject/envars-file-container.yaml
```

Kiểm tra rằng container trong Pod đang chạy:

```shell
# Nếu Pod mới chưa healthy, hãy chạy lại lệnh này vài lần.
kubectl get pods
```

Kiểm tra log của container để xem các biến môi trường:

```shell
kubectl logs envfile-test-pod -c use-envfile | grep DB_ADDRESS
```

Kết quả hiển thị giá trị của các biến môi trường đã chọn:

```
DB_ADDRESS=address
```

## Cú pháp file env (Env file syntax) {#env-file-syntax}

Định dạng file env mà Kubernetes sử dụng là một tập con được định nghĩa chặt chẽ của ngữ nghĩa
biến môi trường trong bash tuân thủ POSIX. Bất kỳ file env nào được Kubernetes hỗ trợ đều sẽ
tạo ra các biến môi trường giống hệt như khi được diễn dịch bởi bash tuân thủ POSIX. Tuy
nhiên, bash tuân thủ POSIX hỗ trợ thêm một số định dạng mà Kubernetes không chấp nhận.

Ví dụ:

```
MY_VAR='my-literal-value'
```

### Quy tắc (Rules)

* Khai báo biến: dùng dạng `VAR='value'`. Khoảng trắng bao quanh dấu `=` bị bỏ qua; khoảng
  trắng ở đầu dòng bị bỏ qua; dòng trống bị bỏ qua.
* Giá trị trong dấu nháy: giá trị phải được bao trong dấu nháy đơn (`'`).
  * Nội dung bên trong dấu nháy đơn được giữ nguyên đúng như viết (literal). Không có xử lý
    chuỗi thoát (escape sequence), gộp khoảng trắng hay diễn dịch ký tự nào được áp dụng.
  * Ký tự xuống dòng bên trong dấu nháy đơn được giữ nguyên (hỗ trợ giá trị nhiều dòng).
* Chú thích (comment): các dòng bắt đầu bằng `#` được coi là chú thích và bị bỏ qua. Ký tự `#`
  nằm bên trong giá trị được bao bằng nháy đơn thì không phải là chú thích.

Ví dụ:

```
# chú thích
DB_ADDRESS='address'

MULTI='line1
line2'
```

### Các dạng không được hỗ trợ (Unsupported forms)

* Giá trị không có dấu nháy bị **cấm**:
  * `VAR=value` — không được hỗ trợ.
* Giá trị trong dấu nháy kép bị **cấm**:
  * `VAR="value"` — không được hỗ trợ.
* Nhiều chuỗi trong dấu nháy đặt liền kề nhau **không** được hỗ trợ:
  * `VAR='val1''val2'` — không được hỗ trợ.
* Mọi hình thức nội suy (interpolation), khai triển (expansion) hay nối chuỗi (concatenation)
  đều **không** được hỗ trợ:
  * `VAR='a'$OTHER` hoặc `VAR=${OTHER}` — không được hỗ trợ.

Yêu cầu nghiêm ngặt về dấu nháy đơn bảo đảm rằng giá trị được kubelet đọc nguyên văn khi nạp
biến môi trường từ file.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [biến môi trường](336-env-variable-expose-pod-info-vi.md).
* Đọc [Định nghĩa biến môi trường cho một Container](331-define-environment-variable-vi.md)
* Đọc [Cung cấp thông tin Pod cho container thông qua biến môi trường](336-env-variable-expose-pod-info-vi.md)

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Trong manifest `envfile-test-pod`, container `use-envfile` có mount volume `config` không? Vậy
   nó tìm được giá trị của `DB_ADDRESS` bằng cách nào, và ai là người đọc file đó?
2. **Câu bẫy.** Bạn tự viết file env với hai dòng `DB_ADDRESS=address` và
   `REST_ENDPOINT="endpoint"`. Cả hai dòng đều hợp lệ với bash. Kubernetes chấp nhận dòng nào, và
   vì sao bài lại ràng buộc chặt hơn bash?
3. Dòng `# chú thích` và giá trị `'a#b'` khác nhau ở chỗ nào? Còn `MULTI='line1` xuống dòng rồi
   `line2'` thì sinh ra một biến hay hai biến?
4. Trên `lab-k8s-master`, trước khi thử tính năng này bạn phải kiểm tra điều gì và bằng lệnh nào?
   Nếu bạn định dùng chính đường này để đưa mật khẩu database vào một Pod, bài trả lời ra sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không** — bài nói rõ volume chỉ được mount vào container ghi file (`initContainer`), còn
   container tiêu thụ thì không được mount volume này. Nó khai `fileKeyRef` với `volumeName: config`,
   `path: config.env` và `key: DB_ADDRESS`; **kubelet** là bên đọc file trong volume `emptyDir` ở
   quá trình khởi tạo container rồi đưa giá trị vào container dưới dạng biến môi trường.
2. **Không dòng nào được chấp nhận.** Giá trị không có dấu nháy (`VAR=value`) bị cấm, và giá trị
   trong nháy kép (`VAR="value"`) cũng bị cấm. Dạng duy nhất hợp lệ là nháy đơn:
   `DB_ADDRESS='address'`. Lý do bài nêu ở cuối mục cú pháp: **yêu cầu nghiêm ngặt về nháy đơn bảo
   đảm giá trị được kubelet đọc nguyên văn** — không escape, không nội suy, không nối chuỗi, nên
   không có chỗ cho sự khác biệt giữa cách bash diễn dịch và cách kubelet diễn dịch.
3. Dòng bắt đầu bằng `#` là **chú thích và bị bỏ qua**; còn `#` nằm **bên trong** giá trị bao bằng
   nháy đơn thì **không phải chú thích**, nó là ký tự thường của giá trị. Khối `MULTI` hai dòng sinh
   ra **một biến duy nhất** có giá trị hai dòng: ký tự xuống dòng bên trong nháy đơn được giữ
   nguyên, đó chính là cách bài hỗ trợ giá trị nhiều dòng.
4. Kiểm tra **phiên bản server** bằng `kubectl version`; bài yêu cầu server bằng hoặc mới hơn phiên
   bản tối thiểu nó nêu, và tính năng đang ở trạng thái beta (bật mặc định). Phiên bản cluster lab
   đối chiếu ở [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Về
   mật khẩu: **không nên** — ghi chú của bài nói `emptyDir` không cung cấp các cơ chế bảo vệ như đối
   tượng Secret chuyên dụng, nên đưa biến môi trường bí mật vào container theo cách này không được
   coi là thực hành bảo mật tốt.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

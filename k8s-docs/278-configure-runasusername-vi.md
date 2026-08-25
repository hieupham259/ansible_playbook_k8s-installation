# Cấu hình RunAsUserName cho Pod và container Windows (Configure RunAsUserName for Windows pods and containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-runasusername/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 15 — Windows, nếu môi trường có node Windows](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows)
→ dòng **Thực hành**, bài 2/4 · Kiểm chứng ở [Lab 15 — Node Windows](labs/LAB-15-NODE-WINDOWS.md)
phần **B5.1** (apply `runAsUserName` lên cluster toàn Linux, đọc kết quả ở ba tình huống
`.spec.os.name`), phần **B5.2** (ràng buộc định dạng `DOMAIN\USER`), và phần **B11.A7**
(`echo $env:USERNAME` trên node Windows thật — **chỉ nhánh A**).

Nói thẳng ngay từ đầu: cluster lab là ba VM Ubuntu `lab-k8s-master`, `lab-k8s-worker1`,
`lab-k8s-worker2` — **không có node Windows**, nên Lab 15 chạy **nhánh B** (đọc-hiểu, có kiểm chứng
ranh giới trên chính cluster Linux). Đây lại là bài Windows **đo được nhiều nhất** trên cluster đó:
tập ràng buộc định dạng username kiểm chứng được trọn vẹn ở B5.2, và B5.1 cho thấy đúng cái ranh
giới đáng nhớ nhất của bài — trường được lưu vào object và không sinh lỗi, nhưng tiến trình trong
container **không** chạy dưới username đó. Chỉ phần cuối — mở PowerShell trong container Windows
và đọc `$env:USERNAME` — mới cần một VM Windows Server.

**Phải hiểu ở lần đọc này:**

- `runAsUserName` là thiết lập cho Pod và container **sẽ chạy trên node Windows**, **gần tương
  đương** `runAsUser` vốn dành riêng cho Linux: nó cho chạy ứng dụng trong container dưới một
  username khác mặc định. Chỗ đặt là `securityContext` → `windowsOptions` → `runAsUserName`.
- Đặt ở **cấp Pod** (mục *Đặt username cho Pod*): các tùy chọn security context Windows đó áp cho
  **tất cả container và init container** trong Pod.
- Đặt ở **cấp container** (mục *Đặt username cho Container*): chỉ áp cho riêng container đó, và
  **ghi đè** thiết lập đã đặt ở cấp Pod. Ví dụ thứ hai của bài đặt cả hai cấp — Pod
  `ContainerUser`, container `ContainerAdministrator` — và kết quả `echo $env:USERNAME` là
  **`ContainerAdministrator`**.
- Mục *Các giới hạn về username trên Windows*: giá trị phải theo định dạng `DOMAIN\USER`, phần
  `DOMAIN\` **là tùy chọn**; username Windows **không phân biệt hoa thường**; trường không được
  rỗng và không được chứa ký tự điều khiển; `USER` tối đa **20 ký tự** và không được chứa
  `" / \ [ ] : ; | = , + * ? < > @`. Bốn ví dụ hợp lệ bài nêu: `ContainerAdministrator`,
  `ContainerUser`, `NT AUTHORITY\NETWORK SERVICE`, `NT AUTHORITY\LOCAL SERVICE`.
- Cả hai manifest ví dụ đều mang `nodeSelector: kubernetes.io/os: windows`, khớp với điều kiện ở
  mục *Trước khi bạn bắt đầu*: cluster cần có worker node Windows, **nơi các Pod chạy workload
  Windows sẽ được lập lịch**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bốn bước chạy thật của mỗi ví dụ: `kubectl apply -f https://k8s.io/examples/windows/...`, `kubectl get pod`, `kubectl exec -it … -- powershell`, `echo $env:USERNAME` | image `mcr.microsoft.com/windows/servercore:ltsc2019` chỉ chạy trên node Windows, và ba VM Ubuntu không có `powershell` để `exec` vào | [Lab 15](labs/LAB-15-NODE-WINDOWS.md) phần **B11.A7**, chỉ nhánh A. Nhánh B thay bằng **B5.1**, đo đúng vế đo được trên Linux |
| Ràng buộc chi tiết của phần `DOMAIN`: tên NetBios tối đa 15 ký tự, tên DNS tối đa 255 ký tự và bộ ký tự cấm của từng loại | phần `DOMAIN\` là tùy chọn, và nó chỉ có nghĩa khi node Windows đã join một domain Active Directory | domain Active Directory nằm ngoài phạm vi **cả hai nhánh** của Lab 15; lý do ghi ở bài [273](273-configure-gmsa-vi.md) và ở [bảng 1.1 của Lab 15](labs/LAB-15-NODE-WINDOWS.md#11-ánh-xạ-tài-liệu-sang-bài-thực-hành) |
| Hai link đầu của mục *Tiếp theo* về quản lý danh tính workload bằng GMSA | GMSA là một chuỗi phụ thuộc riêng: CRD, hai webhook, RBAC verb `use`, và một domain thật | bài [273](273-configure-gmsa-vi.md) — bài 1/4 của chính dòng Thực hành này |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.18 [stable]`

Trang này hướng dẫn cách sử dụng thiết lập `runAsUserName` cho các Pod và container sẽ chạy trên
node Windows. Thiết lập này gần tương đương với thiết lập `runAsUser` dành riêng cho Linux, cho
phép bạn chạy ứng dụng trong container dưới một tên người dùng (username) khác với mặc định.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Cluster cần có các worker node Windows, nơi các Pod với container chạy
workload Windows sẽ được lập lịch (schedule).

## Đặt username cho Pod (Set the Username for a Pod)

Để chỉ định username dùng để thực thi các tiến trình trong container của Pod, hãy thêm field
`securityContext`
([PodSecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podsecuritycontext-v1-core))
vào đặc tả (specification) của Pod, và bên trong nó là field `windowsOptions`
([WindowsSecurityContextOptions](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#windowssecuritycontextoptions-v1-core))
chứa field `runAsUserName`.

Các tùy chọn security context Windows mà bạn chỉ định cho Pod sẽ áp dụng cho tất cả các
container và init container trong Pod.

Đây là file cấu hình cho một Pod Windows có field `runAsUserName` được thiết lập:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: run-as-username-pod-demo
spec:
  securityContext:
    windowsOptions:
      runAsUserName: "ContainerUser"
  containers:
  - name: run-as-username-demo
    image: mcr.microsoft.com/windows/servercore:ltsc2019
    command: ["ping", "-t", "localhost"]
  nodeSelector:
    kubernetes.io/os: windows
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/windows/run-as-username-pod.yaml
```

Xác nhận rằng container của Pod đang chạy:

```shell
kubectl get pod run-as-username-pod-demo
```

Mở một shell tới container đang chạy:

```shell
kubectl exec -it run-as-username-pod-demo -- powershell
```

Kiểm tra rằng shell đang chạy dưới đúng username:

```powershell
echo $env:USERNAME
```

Kết quả sẽ là:

```
ContainerUser
```

## Đặt username cho Container (Set the Username for a Container)

Để chỉ định username dùng để thực thi các tiến trình của một container, hãy thêm field
`securityContext`
([SecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#securitycontext-v1-core))
vào manifest của container, và bên trong nó là field `windowsOptions`
([WindowsSecurityContextOptions](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#windowssecuritycontextoptions-v1-core))
chứa field `runAsUserName`.

Các tùy chọn security context Windows mà bạn chỉ định cho một container chỉ áp dụng cho riêng
container đó, và chúng ghi đè (override) các thiết lập đã đặt ở cấp Pod.

Đây là file cấu hình cho một Pod có một container, với field `runAsUserName` được thiết lập ở cả
cấp Pod và cấp container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: run-as-username-container-demo
spec:
  securityContext:
    windowsOptions:
      runAsUserName: "ContainerUser"
  containers:
  - name: run-as-username-demo
    image: mcr.microsoft.com/windows/servercore:ltsc2019
    command: ["ping", "-t", "localhost"]
    securityContext:
        windowsOptions:
            runAsUserName: "ContainerAdministrator"
  nodeSelector:
    kubernetes.io/os: windows
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/windows/run-as-username-container.yaml
```

Xác nhận rằng container của Pod đang chạy:

```shell
kubectl get pod run-as-username-container-demo
```

Mở một shell tới container đang chạy:

```shell
kubectl exec -it run-as-username-container-demo -- powershell
```

Kiểm tra rằng shell đang chạy dưới đúng username (username được thiết lập ở cấp container):

```powershell
echo $env:USERNAME
```

Kết quả sẽ là:

```
ContainerAdministrator
```

## Các giới hạn về username trên Windows (Windows Username limitations)

Để sử dụng tính năng này, giá trị đặt trong field `runAsUserName` phải là một username hợp lệ.
Nó phải có định dạng: `DOMAIN\USER`, trong đó phần `DOMAIN\` là tùy chọn. Username trên Windows
không phân biệt chữ hoa chữ thường. Ngoài ra, có một số ràng buộc đối với `DOMAIN` và `USER`:

- Field `runAsUserName` không được rỗng, và không được chứa các ký tự điều khiển (giá trị ASCII:
  `0x00-0x1F`, `0x7F`)
- `DOMAIN` phải là một tên NetBios hoặc một tên DNS, mỗi loại có ràng buộc riêng:
  - Tên NetBios: tối đa 15 ký tự, không được bắt đầu bằng `.` (dấu chấm), và không được chứa
    các ký tự sau: `\ / : * ? " < > |`
  - Tên DNS: tối đa 255 ký tự, chỉ chứa các ký tự chữ và số, dấu chấm và dấu gạch ngang, và
    không được bắt đầu hoặc kết thúc bằng `.` (dấu chấm) hoặc `-` (dấu gạch ngang).
- `USER` có tối đa 20 ký tự, không được chứa *chỉ toàn* dấu chấm hoặc dấu cách, và không được
  chứa các ký tự sau: `" / \ [ ] : ; | = , + * ? < > @`.

Ví dụ về các giá trị chấp nhận được cho field `runAsUserName`: `ContainerAdministrator`,
`ContainerUser`, `NT AUTHORITY\NETWORK SERVICE`, `NT AUTHORITY\LOCAL SERVICE`.

Để biết thêm thông tin về các giới hạn này, hãy xem
[tại đây](https://support.microsoft.com/en-us/help/909264/naming-conventions-in-active-directory-for-computers-domains-sites-and)
và
[tại đây](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.localaccounts/new-localuser?view=powershell-5.1).

## Tiếp theo (What's next)

* [Hướng dẫn lập lịch container Windows trong Kubernetes](176-windows-user-guide-vi.md)
* [Quản lý danh tính workload với Group Managed Service Accounts (GMSA)](176-windows-user-guide-vi.md#managing-workload-identity-with-group-managed-service-accounts)
* [Cấu hình GMSA cho Pod và container Windows](273-configure-gmsa-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 15 — kể cả khi bạn
chạy nhánh B của Lab 15 trên ba VM Linux:

1. Một Pod đặt `runAsUserName: "ContainerUser"` ở cấp Pod, và container duy nhất bên trong đặt
   `runAsUserName: "ContainerAdministrator"` ở cấp container. `echo $env:USERNAME` trong container
   đó in ra gì, theo quy tắc nào? Thiết lập ở cấp Pod, ngoài container thường, còn áp cho loại
   container nào nữa?
2. **Câu bẫy.** Trên cluster ba VM Ubuntu của bạn, một Pod Linux mang
   `securityContext.windowsOptions.runAsUserName: "ContainerUser"` lên `Running` bình thường, và
   `kubectl get pod -o yaml` in ra đúng trường đó. Vậy tiến trình trong container **có** chạy dưới
   `ContainerUser` không, và bài lấy gì để bạn suy ra câu trả lời?
3. Trong bốn giá trị `ContainerUser`, `NT AUTHORITY\NETWORK SERVICE`, `bad/user` và chuỗi rỗng,
   giá trị nào hợp lệ theo mục *Các giới hạn về username trên Windows*? Giá trị không hợp lệ vi
   phạm ràng buộc cụ thể nào?
4. Cả hai manifest ví dụ đều có `nodeSelector: kubernetes.io/os: windows`. Bỏ dòng đó đi thì điều
   kiện nào của bài bị phá vỡ?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. In ra **`ContainerAdministrator`**. Quy tắc: các tùy chọn security context Windows chỉ định cho
   **một container** chỉ áp dụng cho riêng container đó và **ghi đè các thiết lập đã đặt ở cấp
   Pod** — đúng kết quả bài in ra ở ví dụ thứ hai. Còn thiết lập ở cấp Pod thì áp cho **tất cả các
   container và init container** trong Pod, không chỉ container thường.
2. **Không.** Bài nói ngay câu đầu tiên rằng đây là thiết lập cho các Pod và container **sẽ chạy
   trên node Windows**, và mục *Trước khi bạn bắt đầu* đặt điều kiện cluster **cần có các worker
   node Windows** để Pod workload Windows được lập lịch lên đó. `ContainerUser` là một tài khoản
   của Windows container, trên Ubuntu không có gì để ánh xạ tới. Chỗ dễ sai là lấy "apply thành
   công, không lỗi, `-o yaml` in ra đúng giá trị" làm bằng chứng tính năng đã có hiệu lực: object
   được **lưu** trường không có nghĩa là trường được **thi hành** — chỉ container runtime của
   Windows mới đọc `windowsOptions`.
3. Hợp lệ: **`ContainerUser`** và **`NT AUTHORITY\NETWORK SERVICE`** — cái sau đúng dạng
   `DOMAIN\USER` và nằm trong danh sách ví dụ chấp nhận được của bài. Không hợp lệ: **`bad/user`**,
   vì `USER` **không được chứa** ký tự `/` (nằm trong bộ `" / \ [ ] : ; | = , + * ? < > @`); và
   **chuỗi rỗng**, vì bài quy định trường `runAsUserName` **không được rỗng**.
4. Điều kiện ở mục *Trước khi bạn bắt đầu*: cluster cần worker node Windows, **nơi các Pod với
   container chạy workload Windows sẽ được lập lịch**. `nodeSelector: kubernetes.io/os: windows` là
   thứ ràng buộc Pod vào đúng nhóm node đó. Bỏ nó đi thì scheduler được phép đặt Pod lên một node
   Linux — trên `lab-k8s-worker1` hay `lab-k8s-worker2` chẳng hạn — nơi image
   `mcr.microsoft.com/windows/servercore:ltsc2019` không chạy được và `runAsUserName` không có
   hiệu lực.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

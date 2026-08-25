# Xử lý song song bằng cách khai triển template (Parallel Processing using Expansions)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/job/parallel-processing-expansion/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 4 — Workload controller](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller)
→ [4b. StatefulSet, DaemonSet, Job và autoscaling](00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling),
bài 7/7 của dòng **Thực hành** · Kiểm chứng ở
[Lab 4b — StatefulSet, DaemonSet và Job](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md),
**phần B10**.

Đây là **bài cuối của nhóm 4b** và là mẫu song song thứ ba. Đọc nó cạnh hai mẫu kia để chốt ranh
giới: [351](351-coarse-parallel-work-queue-vi.md) là **một Job, nhiều Pod, lấy việc từ hàng đợi**;
[353](353-indexed-parallel-processing-vi.md) là **một Job, nhiều Pod, mỗi Pod một chỉ số**; còn
bài này là **nhiều Job, mỗi Job một mục việc**.

**Phải hiểu ở lần đọc này:**

- Hình dạng của mẫu: chạy **nhiều đối tượng Job dựa trên một template chung**, mỗi Job xử lý một
  mục việc. Ví dụ trong bài có ba mục — `apple`, `banana`, `cherry` — nên có ba Job.
- `job-tmpl.yaml` **không phải manifest Kubernetes hợp lệ**: cú pháp `$ITEM` không có ý nghĩa gì
  với Kubernetes. Phải **khai triển trước** rồi mới apply — bài dùng `sed` trong một vòng lặp để
  sinh một file cho mỗi mục, sau đó `kubectl create -f ./jobs`
  (mục *Tạo các manifest từ template*).
- Label chung `jobgroup: jobexample` đặt ở **cả Job lẫn pod template** là thứ cho phép thao tác cả
  nhóm bằng một lệnh: `get jobs -l`, `get pods -l`, `logs -f -l`, `delete job -l`. Bài nói rõ
  **Kubernetes không quan tâm tới label tên `jobgroup`** — đó là label bạn tự chọn để tiện gom
  nhóm (mục *Label trên Job và Pod*).
- Giới hạn của cách dùng `sed`: tham số cũng nằm trong **tên** của Job, mà tên object chỉ được chứa
  một số ký tự nhất định. Ví dụ Jinja2 ở phần sau là để vượt giới hạn đó và cho **nhiều tham số mỗi
  Job**; nó dựa vào tính chất một file YAML chứa nhiều tài liệu ngăn bằng `---`.
- Khi nào **không** dùng mẫu này, ở mục *Các lựa chọn thay thế*: quá nhiều Job thì quản lý cồng
  kềnh ngay cả khi có label; tạo cả lô lớn thì **gây tải cao lên control plane** và API server có
  thể **rate limit** với mã 429; và bạn còn bị chặn bởi **hạn ngạch tài nguyên** đối với Job.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ví dụ nâng cao bằng Jinja2: file `job.yaml.jinja2`, `pip install --user jinja2`, alias `render_template` | là công cụ sinh manifest ở phía client, không phải cơ chế Kubernetes; nắm ý tưởng khai triển template là đủ | [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) phần B10 chỉ khai triển bằng `sed`, đúng phần làm được bằng công cụ có sẵn |
| Mục *Sử dụng Job trong workload thực tế* — mỗi Pod ghi output vào PersistentVolume hoặc một dịch vụ lưu trữ bên ngoài | PersistentVolume chưa được học, và [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) không tạo PVC | [giai đoạn 6 — Lưu trữ](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) |
| Link *các label được khuyến nghị* (bài [31](31-common-labels-vi.md)) trong ghi chú về khóa `jobgroup` | ở đây chỉ cần biết `jobgroup` là label bạn tự đặt, không có gì đặc biệt | bài [31](31-common-labels-vi.md) đã đọc ở [1b. Làm việc với object và kubectl](00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl) |
| Gợi ý tự viết [controller](25-controllers-vi.md) để quản lý các đối tượng Job một cách tự động | viết controller riêng là chủ đề mở rộng Kubernetes, cần công cụ chưa học | [giai đoạn 14 — Khả năng mở rộng](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) |

---

Tác vụ này minh họa cách chạy nhiều Job dựa trên một template chung. Bạn có thể
dùng cách tiếp cận này để xử lý các lô (batch) công việc song song.

Trong ví dụ này chỉ có ba mục: _apple_, _banana_ và _cherry_.
Các Job mẫu xử lý từng mục bằng cách in ra một chuỗi rồi tạm dừng.

Xem [sử dụng Job trong workload thực tế](#using-jobs-in-real-workloads) để tìm hiểu
mẫu hình (pattern) này phù hợp với các trường hợp sử dụng thực tế hơn như thế nào.

## Trước khi bạn bắt đầu (Before you begin)

Bạn nên đã quen với cách sử dụng cơ bản, không song song, của
[Job](67-job-vi.md).

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được
cấu hình để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một
cluster có ít nhất hai node không đóng vai trò máy chủ control plane. Nếu bạn
chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để tạo template ở mức cơ bản, bạn cần tiện ích dòng lệnh `sed`.

Để làm theo ví dụ tạo template nâng cao, bạn cần một bản cài đặt
[Python](https://www.python.org/) hoạt động được, và thư viện template Jinja2
cho Python.

Sau khi đã thiết lập Python, bạn có thể cài Jinja2 bằng cách chạy:

```shell
pip install --user jinja2
```

## Tạo các Job dựa trên một template (Create Jobs based on a template) {#create-jobs-based-on-a-template}

Trước tiên, tải template Job sau về một file có tên `job-tmpl.yaml`.
Đây là nội dung bạn sẽ tải về:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: process-item-$ITEM
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      name: jobexample
      labels:
        jobgroup: jobexample
    spec:
      containers:
      - name: c
        image: busybox:1.28
        command: ["sh", "-c", "echo Processing item $ITEM && sleep 5"]
      restartPolicy: Never
```

```shell
# Dùng curl để tải job-tmpl.yaml
curl -L -s -O https://k8s.io/examples/application/job/job-tmpl.yaml
```

File bạn vừa tải về chưa phải là một manifest Kubernetes hợp lệ.
Thay vào đó, template này là một biểu diễn YAML của một đối tượng Job với một số
placeholder cần được điền vào trước khi có thể sử dụng. Cú pháp `$ITEM` không có ý nghĩa
gì đối với Kubernetes.

### Tạo các manifest từ template (Create manifests from the template)

Đoạn shell sau dùng `sed` để thay thế chuỗi `$ITEM` bằng biến lặp,
ghi kết quả vào một thư mục tạm tên là `jobs`. Hãy chạy đoạn này ngay:

```shell
# Khai triển template thành nhiều file, mỗi file cho một mục cần xử lý.
mkdir ./jobs
for i in apple banana cherry
do
  cat job-tmpl.yaml | sed "s/\$ITEM/$i/" > ./jobs/job-$i.yaml
done
```

Kiểm tra xem đã hoạt động chưa:

```shell
ls jobs/
```

Kết quả tương tự như sau:

```
job-apple.yaml
job-banana.yaml
job-cherry.yaml
```

Bạn có thể dùng bất kỳ ngôn ngữ template nào (ví dụ: Jinja2; ERB), hoặc
viết một chương trình để sinh ra các manifest Job.

### Tạo các Job từ các manifest (Create Jobs from the manifests)

Tiếp theo, tạo tất cả các Job bằng một lệnh kubectl:

```shell
kubectl create -f ./jobs
```

Kết quả tương tự như sau:

```
job.batch/process-item-apple created
job.batch/process-item-banana created
job.batch/process-item-cherry created
```

Bây giờ, hãy kiểm tra các Job:

```shell
kubectl get jobs -l jobgroup=jobexample
```

Kết quả tương tự như sau:

```
NAME                  COMPLETIONS   DURATION   AGE
process-item-apple    1/1           14s        22s
process-item-banana   1/1           12s        21s
process-item-cherry   1/1           12s        20s
```

Việc dùng tùy chọn `-l` với kubectl chỉ chọn các Job thuộc nhóm Job này
(trong hệ thống có thể có các Job khác không liên quan).

Bạn cũng có thể kiểm tra các Pod bằng cùng label selector đó:

```shell
kubectl get pods -l jobgroup=jobexample
```

Kết quả tương tự như sau:

```
NAME                        READY     STATUS      RESTARTS   AGE
process-item-apple-kixwv    0/1       Completed   0          4m
process-item-banana-wrsf7   0/1       Completed   0          4m
process-item-cherry-dnfu9   0/1       Completed   0          4m
```

Chúng ta có thể dùng một lệnh duy nhất này để kiểm tra output của tất cả các Job cùng lúc:

```shell
kubectl logs -f -l jobgroup=jobexample
```

Kết quả sẽ là:

```
Processing item apple
Processing item banana
Processing item cherry
```

### Dọn dẹp (Clean up) {#cleanup-1}

```shell
# Xóa các Job bạn đã tạo
# Cluster của bạn sẽ tự động dọn dẹp các Pod của chúng
kubectl delete job -l jobgroup=jobexample
```

## Sử dụng tham số template nâng cao (Use advanced template parameters)

Trong [ví dụ thứ nhất](#create-jobs-based-on-a-template), mỗi bản thể hiện (instance) của template
có một tham số, và tham số đó cũng được dùng trong tên của Job. Tuy nhiên,
[tên](17-names-vi.md#names) bị giới hạn
chỉ được chứa một số ký tự nhất định.

Ví dụ phức tạp hơn một chút này dùng
[ngôn ngữ template Jinja](https://palletsprojects.com/p/jinja/) để sinh ra các manifest
rồi từ đó tạo các đối tượng, với nhiều tham số cho mỗi Job.

Trong phần này của tác vụ, bạn sẽ dùng một script Python một dòng để
chuyển đổi template thành một tập các manifest.

Trước tiên, sao chép và dán template sau của một đối tượng Job vào một file có tên `job.yaml.jinja2`:

```liquid
{% set params = [{ "name": "apple", "url": "http://dbpedia.org/resource/Apple", },
                  { "name": "banana", "url": "http://dbpedia.org/resource/Banana", },
                  { "name": "cherry", "url": "http://dbpedia.org/resource/Cherry" }]
%}
{% for p in params %}
{% set name = p["name"] %}
{% set url = p["url"] %}
---
apiVersion: batch/v1
kind: Job
metadata:
  name: jobexample-{{ name }}
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      name: jobexample
      labels:
        jobgroup: jobexample
    spec:
      containers:
      - name: c
        image: busybox:1.28
        command: ["sh", "-c", "echo Processing URL {{ url }} && sleep 5"]
      restartPolicy: Never
{% endfor %}
```

Template trên định nghĩa hai tham số cho mỗi đối tượng Job bằng một danh sách
các dict của Python (dòng 1-4). Một vòng lặp `for` sinh ra một manifest Job cho mỗi
bộ tham số (các dòng còn lại).

Ví dụ này dựa trên một tính năng của YAML. Một file YAML có thể chứa nhiều
tài liệu (trong trường hợp này là các manifest Kubernetes), được phân tách bằng `---`
trên một dòng riêng.
Bạn có thể pipe output trực tiếp sang `kubectl` để tạo các Job.

Tiếp theo, dùng chương trình Python một dòng này để khai triển template:

```shell
alias render_template='python -c "from jinja2 import Template; import sys; print(Template(sys.stdin.read()).render());"'
```

Dùng `render_template` để chuyển đổi các tham số và template thành một
file YAML duy nhất chứa các manifest Kubernetes:

```shell
# Lệnh này cần alias mà bạn đã định nghĩa ở trên
cat job.yaml.jinja2 | render_template > jobs.yaml
```

Bạn có thể xem `jobs.yaml` để xác nhận rằng script `render_template` đã hoạt động
đúng.

Khi bạn đã hài lòng rằng `render_template` hoạt động như bạn mong muốn,
bạn có thể pipe output của nó sang `kubectl`:

```shell
cat job.yaml.jinja2 | render_template | kubectl apply -f -
```

Kubernetes chấp nhận và chạy các Job mà bạn đã tạo.

### Dọn dẹp (Clean up) {#cleanup-2}

```shell
# Xóa các Job bạn đã tạo
# Cluster của bạn sẽ tự động dọn dẹp các Pod của chúng
kubectl delete job -l jobgroup=jobexample
```

## Sử dụng Job trong workload thực tế (Using Jobs in real workloads) {#using-jobs-in-real-workloads}

Trong một trường hợp sử dụng thực tế, mỗi Job thực hiện một khối lượng tính toán đáng kể, chẳng hạn
render một khung hình của một bộ phim, hoặc xử lý một dải hàng (rows) trong cơ sở dữ liệu. Nếu bạn
render một bộ phim, bạn sẽ đặt `$ITEM` là số thứ tự khung hình. Nếu bạn xử lý các hàng từ một bảng
cơ sở dữ liệu, bạn sẽ đặt `$ITEM` biểu diễn dải các hàng cần xử lý.

Trong tác vụ này, bạn đã chạy một lệnh để thu thập output từ các Pod bằng cách lấy
log của chúng. Trong một trường hợp sử dụng thực tế, mỗi Pod của một Job sẽ ghi output của nó vào
bộ lưu trữ bền vững (durable storage) trước khi hoàn thành. Bạn có thể dùng một PersistentVolume cho mỗi Job,
hoặc một dịch vụ lưu trữ bên ngoài. Ví dụ, nếu bạn render các khung hình cho một bộ phim,
hãy dùng HTTP để `PUT` dữ liệu khung hình đã render lên một URL, với mỗi khung hình
dùng một URL khác nhau.

## Label trên Job và Pod (Labels on Jobs and Pods)

Sau khi bạn tạo một Job, Kubernetes tự động thêm các label bổ sung
để phân biệt các Pod của Job này với các Pod của Job khác.

Trong ví dụ này, mỗi Job và Pod template của nó có một label:
`jobgroup=jobexample`.

Bản thân Kubernetes không quan tâm đến các label có tên `jobgroup`. Việc đặt một label
cho tất cả các Job mà bạn tạo từ một template giúp thao tác trên tất cả
các Job đó cùng lúc một cách thuận tiện.
Trong [ví dụ thứ nhất](#create-jobs-based-on-a-template) bạn đã dùng một template để
tạo nhiều Job. Template đảm bảo rằng mỗi Pod cũng nhận được cùng label đó, nên
bạn có thể kiểm tra tất cả các Pod của các Job được tạo từ template này bằng một lệnh duy nhất.

> **Ghi chú:**
> Khóa label `jobgroup` không có gì đặc biệt hay được dành riêng.
> Bạn có thể chọn cách đặt label của riêng mình.
> Có các [label được khuyến nghị](31-common-labels-vi.md#labels)
> mà bạn có thể dùng nếu muốn.

## Các lựa chọn thay thế (Alternatives)

Nếu bạn dự định tạo một số lượng lớn đối tượng Job, bạn có thể thấy rằng:

- Ngay cả khi dùng label, việc quản lý quá nhiều Job vẫn cồng kềnh.
- Nếu bạn tạo nhiều Job trong một lô, bạn có thể gây tải cao
  lên control plane của Kubernetes. Ngoài ra, Kubernetes API
  server có thể giới hạn tần suất (rate limit) của bạn, tạm thời từ chối các yêu cầu với mã trạng thái 429.
- Bạn bị giới hạn bởi hạn ngạch tài nguyên (resource quota)
  đối với Job: API server sẽ từ chối vĩnh viễn một số yêu cầu của bạn
  khi bạn tạo một khối lượng công việc rất lớn trong một lô.

Có các [mẫu hình Job (job patterns)](67-job-vi.md#job-patterns)
khác mà bạn có thể dùng để xử lý khối lượng công việc lớn mà không cần tạo quá nhiều
đối tượng Job.

Bạn cũng có thể cân nhắc viết [controller](25-controllers-vi.md)
của riêng mình để quản lý các đối tượng Job một cách tự động.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ để mở lab của nhóm 4b:

1. Về số lượng đối tượng Job sinh ra, mẫu này khác Indexed Job
   ([353](353-indexed-parallel-processing-vi.md)) và mẫu hàng đợi công việc
   ([351](351-coarse-parallel-work-queue-vi.md)) ở chỗ nào?
2. **Câu bẫy.** Bạn tải `job-tmpl.yaml` về rồi `kubectl apply -f job-tmpl.yaml` thẳng. Có Job nào
   được tạo không? Vì sao?
3. Label `jobgroup: jobexample` xuất hiện ở hai chỗ trong template. Hai chỗ đó là gì, và nhờ nó
   bạn làm được những thao tác nào bằng một lệnh duy nhất?
4. Trên cluster lab ba VM, bạn định sinh vài nghìn Job từ một template rồi apply cả lô. Bài cảnh
   báo ba vấn đề gì?
5. Trong ví dụ dùng `sed`, `$ITEM` xuất hiện cả trong `metadata.name` của Job. Bài nêu ràng buộc
   nào khiến cách đó không dùng được cho tham số bất kỳ?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Mẫu này tạo **nhiều đối tượng Job — mỗi mục việc một Job**. Hai mẫu kia chỉ có **một** đối tượng
   Job: bài [351](351-coarse-parallel-work-queue-vi.md) cho các Pod của một Job **lấy việc từ hàng
   đợi**, còn bài [353](353-indexed-parallel-processing-vi.md) cho mỗi Pod của một Job **một chỉ
   số** để tự biết phần việc. Ở bài này, việc được cố định vào từng Job ngay lúc sinh manifest.
2. **Không, và lệnh sẽ không chạy được.** Đây là chỗ dễ nhầm vì file có đuôi `.yaml` và trông đúng
   như một manifest. Nhưng bài nói thẳng: file bạn tải về **chưa phải là một manifest Kubernetes
   hợp lệ** — nó là biểu diễn YAML của một Job **với placeholder**, và **cú pháp `$ITEM` không có ý
   nghĩa gì đối với Kubernetes**. Phải khai triển trước, ví dụ bằng vòng lặp `sed` sinh ra
   `job-apple.yaml`, `job-banana.yaml`, `job-cherry.yaml`, rồi mới `kubectl create -f ./jobs`.
3. Ở **`metadata.labels` của Job** và ở **`metadata.labels` của pod template** bên trong Job. Vì
   template bảo đảm **mỗi Pod cũng nhận cùng label đó**, bạn thao tác cả nhóm bằng một lệnh:
   `kubectl get jobs -l jobgroup=jobexample`, `kubectl get pods -l jobgroup=jobexample`,
   `kubectl logs -f -l jobgroup=jobexample` và `kubectl delete job -l jobgroup=jobexample`. Bài
   nhấn mạnh **bản thân Kubernetes không quan tâm tới label tên `jobgroup`**; khóa đó không đặc
   biệt và không được dành riêng.
4. Ba vấn đề ở mục *Các lựa chọn thay thế*: **quản lý quá nhiều Job vẫn cồng kềnh ngay cả khi dùng
   label**; **tạo nhiều Job trong một lô gây tải cao lên control plane**, và API server có thể
   **rate limit, tạm thời từ chối yêu cầu với mã trạng thái 429**; và bạn **bị giới hạn bởi hạn
   ngạch tài nguyên đối với Job** — API server sẽ từ chối vĩnh viễn một số yêu cầu khi khối lượng
   trong một lô quá lớn.
5. Ràng buộc về **tên object**: tham số được nhét vào `metadata.name` (`process-item-$ITEM`), mà
   tên trong Kubernetes **bị giới hạn chỉ được chứa một số ký tự nhất định**. Đó là lý do bài
   chuyển sang ví dụ Jinja2 khi cần nhiều tham số cho mỗi Job.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của nhóm
[4b. StatefulSet, DaemonSet, Job và autoscaling](00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling);
trả lời trôi cả năm câu thì mở
[Lab 4b — StatefulSet, DaemonSet và Job](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) và làm từ
phần B0.

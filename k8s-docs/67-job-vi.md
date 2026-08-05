# Jobs

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/controllers/job/>
>
> Job đại diện cho các tác vụ một lần (one-off task): chạy đến khi hoàn thành rồi dừng lại.

Một Job tạo ra một hoặc nhiều Pod và sẽ tiếp tục thử lại việc thực thi các Pod cho đến
khi một số lượng Pod xác định kết thúc thành công. Khi các Pod hoàn thành thành công,
Job theo dõi các lần hoàn thành thành công đó. Khi đạt đủ số lần hoàn thành thành công
đã chỉ định, tác vụ (tức là Job) hoàn tất. Xóa một Job sẽ dọn dẹp các Pod mà nó đã tạo.
Tạm dừng (suspend) một Job sẽ xóa các Pod đang hoạt động của nó cho đến khi Job được
tiếp tục trở lại.

Trường hợp đơn giản là tạo một đối tượng Job để chạy một Pod đến khi hoàn thành một cách
đáng tin cậy. Đối tượng Job sẽ khởi động một Pod mới nếu Pod đầu tiên thất bại hoặc bị
xóa (ví dụ do lỗi phần cứng của node hoặc node khởi động lại).

Bạn cũng có thể dùng Job để chạy nhiều Pod song song.

Nếu bạn muốn chạy một Job (một tác vụ đơn lẻ, hoặc nhiều tác vụ song song) theo lịch,
hãy xem [CronJob](./69-cron-jobs-vi.md).

## Chạy một Job ví dụ (Running an example Job)

Dưới đây là một cấu hình Job ví dụ. Nó tính số π đến 2000 chữ số và in kết quả ra.
Job này mất khoảng 10 giây để hoàn thành.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

Bạn có thể chạy ví dụ này bằng lệnh sau:

```shell
kubectl apply -f https://kubernetes.io/examples/controllers/job.yaml
```

Kết quả tương tự như sau:

```
job.batch/pi created
```

Kiểm tra trạng thái của Job bằng `kubectl`:

#### kubectl describe job pi

```bash
Name:           pi
Namespace:      default
Selector:       batch.kubernetes.io/controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
Labels:         batch.kubernetes.io/controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
                batch.kubernetes.io/job-name=pi
                ...
Annotations:    batch.kubernetes.io/job-tracking: ""
Parallelism:    1
Completions:    1
Start Time:     Mon, 02 Dec 2019 15:20:11 +0200
Completed At:   Mon, 02 Dec 2019 15:21:16 +0200
Duration:       65s
Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:  batch.kubernetes.io/controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
           batch.kubernetes.io/job-name=pi
  Containers:
   pi:
    Image:      perl:5.34.0
    Port:       <none>
    Host Port:  <none>
    Command:
      perl
      -Mbignum=bpi
      -wle
      print bpi(2000)
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  21s   job-controller  Created pod: pi-xf9p4
  Normal  Completed         18s   job-controller  Job completed
```

#### kubectl get job pi -o yaml

```bash
apiVersion: batch/v1
kind: Job
metadata:
  annotations: batch.kubernetes.io/job-tracking: ""
             ...  
  creationTimestamp: "2022-11-10T17:53:53Z"
  generation: 1
  labels:
    batch.kubernetes.io/controller-uid: 863452e6-270d-420e-9b94-53a54146c223
    batch.kubernetes.io/job-name: pi
  name: pi
  namespace: default
  resourceVersion: "4751"
  uid: 204fb678-040b-497f-9266-35ffa8716d14
spec:
  backoffLimit: 4
  completionMode: NonIndexed
  completions: 1
  parallelism: 1
  selector:
    matchLabels:
      batch.kubernetes.io/controller-uid: 863452e6-270d-420e-9b94-53a54146c223
  suspend: false
  template:
    metadata:
      creationTimestamp: null
      labels:
        batch.kubernetes.io/controller-uid: 863452e6-270d-420e-9b94-53a54146c223
        batch.kubernetes.io/job-name: pi
    spec:
      containers:
      - command:
        - perl
        - -Mbignum=bpi
        - -wle
        - print bpi(2000)
        image: perl:5.34.0
        imagePullPolicy: IfNotPresent
        name: pi
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Never
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  active: 1
  ready: 0
  startTime: "2022-11-10T17:53:57Z"
  uncountedTerminatedPods: {}
```

Để xem các Pod đã hoàn thành của một Job, dùng `kubectl get pods`.

Để liệt kê tất cả các Pod thuộc về một Job ở dạng máy có thể đọc được (machine readable),
bạn có thể dùng một lệnh như sau:

```shell
pods=$(kubectl get pods --selector=batch.kubernetes.io/job-name=pi --output=jsonpath='{.items[*].metadata.name}')
echo $pods
```

Kết quả tương tự như sau:

```
pi-5rwd7
```

Ở đây, selector này giống với selector của Job. Tùy chọn `--output=jsonpath` chỉ định một
biểu thức lấy tên từ mỗi Pod trong danh sách trả về.

Xem standard output của một trong các Pod:

```shell
kubectl logs $pods
```

Một cách khác để xem log của một Job:

```shell
kubectl logs jobs/pi
```

Kết quả tương tự như sau:

```
3.1415926535897932384626433832795028841971693993751058209749445923078164062862089986280348253421170679821480865132823066470938446095505822317253594081284811174502841027019385211055596446229489549303819644288109756659334461284756482337867831652712019091456485669234603486104543266482133936072602491412737245870066063155881748815209209628292540917153643678925903600113305305488204665213841469519415116094330572703657595919530921861173819326117931051185480744623799627495673518857527248912279381830119491298336733624406566430860213949463952247371907021798609437027705392171762931767523846748184676694051320005681271452635608277857713427577896091736371787214684409012249534301465495853710507922796892589235420199561121290219608640344181598136297747713099605187072113499999983729780499510597317328160963185950244594553469083026425223082533446850352619311881710100031378387528865875332083814206171776691473035982534904287554687311595628638823537875937519577818577805321712268066130019278766111959092164201989380952572010654858632788659361533818279682303019520353018529689957736225994138912497217752834791315155748572424541506959508295331168617278558890750983817546374649393192550604009277016711390098488240128583616035637076601047101819429555961989467678374494482553797747268471040475346462080466842590694912933136770289891521047521620569660240580381501935112533824300355876402474964732639141992726042699227967823547816360093417216412199245863150302861829745557067498385054945885869269956909272107975093029553211653449872027559602364806654991198818347977535663698074265425278625518184175746728909777727938000816470600161452491921732172147723501414419735685481613611573525521334757418494684385233239073941433345477624168625189835694855620992192221842725502542568876717904946016534668049886272327917860857843838279679766814541009538837863609506800642251252051173929848960841284886269456042419652850222106611863067442786220391949450471237137869609563643719172874677646575739624138908658326459958133904780275901
```

## Viết một Job spec (Writing a Job spec)

Giống như mọi cấu hình Kubernetes khác, một Job cần các trường `apiVersion`, `kind`, và
`metadata`.

Khi control plane tạo các Pod mới cho một Job, `.metadata.name` của Job là một phần cơ sở
để đặt tên cho các Pod đó. Tên của một Job phải là một giá trị
[DNS subdomain](./17-names-vi.md) hợp lệ, nhưng điều này có thể tạo ra kết quả không
mong muốn cho hostname của các Pod. Để tương thích tốt nhất, tên này nên tuân theo các
quy tắc chặt chẽ hơn dành cho một [DNS label](./17-names-vi.md#dns-label-names).
Ngay cả khi tên là một DNS subdomain, tên cũng không được dài quá 63 ký tự.

Một Job cũng cần có
[phần `.spec`](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status).

### Label của Job (Job Labels)

Label của Job sẽ có tiền tố (prefix) `batch.kubernetes.io/` cho `job-name` và `controller-uid`.

### Pod Template

`.spec.template` là trường bắt buộc duy nhất của `.spec`.

`.spec.template` là một [pod template](./46-pods-vi.md#pod-template). Nó có schema
giống hệt một Pod, ngoại trừ việc nó được lồng bên trong và không có `apiVersion` hay
`kind`.

Ngoài các trường bắt buộc của một Pod, một pod template trong Job phải chỉ định các label
phù hợp (xem [pod selector](#pod-selector)) và một chính sách khởi động lại (restart
policy) phù hợp.

Chỉ cho phép [`RestartPolicy`](./47-pod-lifecycle-vi.md#restart-policy) bằng `Never`
hoặc `OnFailure`.

### Pod selector {#pod-selector}

Trường `.spec.selector` là tùy chọn. Trong hầu hết mọi trường hợp, bạn không nên chỉ định
nó. Xem phần [chỉ định pod selector của riêng bạn](#specifying-your-own-pod-selector).

### Thực thi song song cho Job (Parallel execution for Jobs) {#parallel-jobs}

Có ba loại tác vụ chính phù hợp để chạy dưới dạng một Job:

1. Job không song song (non-parallel)
   - thông thường, chỉ một Pod được khởi động, trừ khi Pod đó thất bại.
   - Job hoàn tất ngay khi Pod của nó kết thúc thành công.
1. Job song song với *số lần hoàn thành cố định* (fixed completion count):
   - chỉ định một giá trị dương khác 0 cho `.spec.completions`.
   - Job đại diện cho tác vụ tổng thể, và hoàn tất khi có `.spec.completions` Pod thành công.
   - khi dùng `.spec.completionMode="Indexed"`, mỗi Pod nhận một index khác nhau
     trong khoảng từ 0 đến `.spec.completions-1`.
1. Job song song với *hàng đợi công việc* (work queue):
   - không chỉ định `.spec.completions`, mặc định theo `.spec.parallelism`.
   - các Pod phải tự phối hợp với nhau hoặc thông qua một dịch vụ bên ngoài để xác định
     mỗi Pod sẽ xử lý phần việc nào. Ví dụ, một Pod có thể lấy một lô (batch) tối đa N
     mục từ hàng đợi công việc.
   - mỗi Pod có khả năng tự xác định một cách độc lập liệu tất cả các Pod ngang hàng
     (peer) đã xong việc hay chưa, và nhờ đó biết toàn bộ Job đã xong hay chưa.
   - khi _bất kỳ_ Pod nào của Job kết thúc thành công, không Pod mới nào được tạo thêm.
   - khi có ít nhất một Pod đã kết thúc thành công và tất cả các Pod đều đã kết thúc,
     thì Job hoàn tất thành công.
   - khi bất kỳ Pod nào đã thoát (exit) thành công, không Pod nào khác nên còn đang làm
     bất kỳ phần việc nào cho tác vụ này hay đang ghi bất kỳ output nào. Tất cả chúng
     nên đang trong quá trình thoát.

Với Job _không song song_, bạn có thể để trống cả `.spec.completions` và
`.spec.parallelism`. Khi cả hai không được đặt, cả hai đều mặc định là 1.

Với Job _số lần hoàn thành cố định_, bạn nên đặt `.spec.completions` bằng số lần hoàn
thành cần thiết. Bạn có thể đặt `.spec.parallelism`, hoặc để trống và nó sẽ mặc định là 1.

Với Job _hàng đợi công việc_, bạn phải để trống `.spec.completions`, và đặt
`.spec.parallelism` là một số nguyên không âm.

Để biết thêm thông tin về cách sử dụng các loại Job khác nhau, xem phần
[các mẫu sử dụng Job](#job-patterns).

#### Kiểm soát mức song song (Controlling parallelism)

Mức song song yêu cầu (`.spec.parallelism`) có thể được đặt thành bất kỳ giá trị không
âm nào. Nếu không chỉ định, nó mặc định là 1. Nếu được đặt là 0, Job thực tế bị tạm dừng
cho đến khi giá trị này được tăng lên.

Mức song song thực tế (số pod đang chạy tại bất kỳ thời điểm nào) có thể nhiều hơn hoặc
ít hơn mức song song yêu cầu, vì nhiều lý do:

- Với Job _số lần hoàn thành cố định_, số pod thực tế chạy song song sẽ không vượt quá số
  lần hoàn thành còn lại. Các giá trị `.spec.parallelism` cao hơn thực tế bị bỏ qua.
- Với Job _hàng đợi công việc_, không Pod mới nào được khởi động sau khi có bất kỳ Pod
  nào thành công — tuy nhiên, các Pod còn lại vẫn được phép chạy đến khi hoàn thành.
- Nếu Job controller chưa kịp phản ứng.
- Nếu Job controller không tạo được Pod vì bất kỳ lý do gì (thiếu `ResourceQuota`, thiếu
  quyền, v.v.), thì số pod có thể ít hơn yêu cầu.
- Job controller có thể hạn chế (throttle) việc tạo Pod mới do đã có quá nhiều pod thất
  bại trước đó trong cùng một Job.
- Khi một Pod được tắt một cách êm thấm (gracefully shut down), việc dừng lại cần thời gian.

### Chế độ hoàn thành (Completion mode) {#completion-mode}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Job với _số lần hoàn thành cố định_ — tức là Job có `.spec.completions` khác null — có
thể có một chế độ hoàn thành được chỉ định trong `.spec.completionMode`:

- `NonIndexed` (mặc định): Job được coi là hoàn tất khi đã có `.spec.completions` Pod
  hoàn thành thành công. Nói cách khác, mỗi lần hoàn thành của Pod là tương đồng với
  nhau. Lưu ý rằng các Job có `.spec.completions` là null thì ngầm định là `NonIndexed`.
- `Indexed`: các Pod của một Job nhận một index hoàn thành tương ứng từ 0 đến
  `.spec.completions-1`. Index này khả dụng thông qua bốn cơ chế:
  - Annotation của Pod `batch.kubernetes.io/job-completion-index`.
  - Label của Pod `batch.kubernetes.io/job-completion-index` (từ v1.28 trở đi). Lưu ý
    feature gate `PodIndexLabel` phải được bật để dùng label này, và nó được bật mặc định.
  - Là một phần của hostname của Pod, theo mẫu `$(job-name)-$(index)`.
    Khi bạn dùng một Indexed Job kết hợp với một Service, các Pod trong Job có thể dùng
    các hostname xác định (deterministic) này để liên lạc với nhau qua DNS. Để biết thêm
    thông tin về cách cấu hình, xem
    [Job với giao tiếp Pod-to-Pod](https://kubernetes.io/docs/tasks/job/job-with-pod-to-pod-communication/).
  - Từ tác vụ chạy trong container, thông qua biến môi trường `JOB_COMPLETION_INDEX`.

  Job được coi là hoàn tất khi có một Pod hoàn thành thành công cho mỗi index. Để biết
  thêm thông tin về cách dùng chế độ này, xem
  [Indexed Job cho xử lý song song với phân công việc tĩnh](https://kubernetes.io/docs/tasks/job/indexed-parallel-processing-static/).

> **Ghi chú:**
> Dù hiếm gặp, có thể có nhiều hơn một Pod được khởi động cho cùng một index (do nhiều lý
> do khác nhau như node bị lỗi, kubelet khởi động lại, hoặc Pod bị trục xuất - eviction).
> Trong trường hợp này, chỉ Pod đầu tiên hoàn thành thành công mới được tính vào số lần
> hoàn thành và cập nhật status của Job. Các Pod khác đang chạy hoặc đã hoàn thành cho
> cùng index đó sẽ bị Job controller xóa ngay khi chúng bị phát hiện.

## Tích hợp với các Workload API (Integrate with Workload APIs) {#integrate-with-workload-apis}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Khi feature gate [`WorkloadWithJob`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
được bật, Job controller tự động tạo các đối tượng [Workload](./77-workload-api-vi.md) và
[PodGroup](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/workload-v1alpha1/)
cho [các Job song song đủ điều kiện](#qualifying-criteria) trước khi tạo bất kỳ Pod nào.
Điều này cho phép [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)
nguyên bản (native), trong đó tất cả các Pod của một Job được lập lịch cùng nhau hoặc
không Pod nào được lập lịch cả.

### Tiêu chí đủ điều kiện (Qualifying criteria) {#qualifying-criteria}

Job controller tạo một Workload với một
[chính sách gang scheduling](./79-workload-policies-vi.md)
khi Job thỏa mãn tất cả các điều kiện sau:

- `.spec.parallelism` lớn hơn 1
- `.spec.completionMode` là `Indexed`
- `.spec.parallelism` bằng `.spec.completions`
- `.spec.template.spec.schedulingGroup` không được đặt

Các Job không khớp các tiêu chí này tiếp tục lập lịch các Pod một cách độc lập, không có
`Workload` hay `PodGroup` nào được tạo.

Ví dụ, Job sau đây chạy 8 worker đánh index song song. Khi tính năng được bật, Job
controller tạo một `Workload` và `PodGroup` với `minCount: 8` trước khi tạo bất kỳ Pod
nào, đảm bảo cả 8 worker được lập lịch cùng nhau:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: distributed-training
  namespace: training
spec:
  parallelism: 8
  completions: 8
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: trainer
        image: training-image:latest
        resources:
          limits:
            nvidia.com/gpu: 1
```

Khi Job controller xử lý Job này, nó tự động:

1. Tạo một đối tượng [Workload](./77-workload-api-vi.md) trong cùng namespace. Workload
   chứa một `podGroupTemplate` với một
   [chính sách gang scheduling](./79-workload-policies-vi.md)
   trong đó `minCount` bằng parallelism của Job.
1. Tạo một đối tượng [PodGroup](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/workload-v1alpha1/)
   dựa trên template đó.
   PodGroup là một đơn vị lập lịch runtime độc lập, mang một bản sao inline của chính
   sách gang.
1. Tạo các Pod với `spec.schedulingGroup.podGroupName` được đặt thành tên của PodGroup,
   liên kết mỗi Pod với nhóm lập lịch (scheduling group) của nó.

Việc phát hiện (discovery) các đối tượng này dựa trên các tham chiếu trong spec
(`controllerRef` và `podGroupTemplateRef`).

Workload và PodGroup thuộc sở hữu của Job (thông qua `ownerReferences`) và được tự động
thu gom rác (garbage collect) khi Job bị xóa.

### Cơ chế loại trừ cho các controller cấp cao hơn (Opt-out for higher-level controllers)

Nếu Pod template của một Job đã có sẵn `spec.schedulingGroup`, Job controller sẽ không
tạo các đối tượng `Workload` hay `PodGroup`. Điều này cho phép các controller cấp cao
hơn, chẳng hạn `JobSet`, tự quản lý vòng đời của `Workload` và `PodGroup`.

### Hành vi với CronJob (CronJob behavior)

Các Job do một `CronJob` tạo ra không có `schedulingGroup` được đặt trong `PodTemplate`.
Nếu một `Job` do CronJob tạo ra khớp với các tiêu chí gang scheduling, Job controller
tạo một `Workload` và `PodGroup` riêng cho mỗi thực thể (instance) Job.

### Các hạn chế của bản phát hành Alpha (Limitations for Alpha release) {#workload-integration-limitations}

- Mỗi Job ánh xạ tới đúng một `PodGroup`. Tất cả các Pod trong Job thuộc về cùng một
  nhóm lập lịch.
- `minCount` trong chính sách gang là bất biến (immutable). Các cập nhật vào
  `.spec.parallelism` bị từ chối đối với các Job dùng gang scheduling. Xem
  [Elastic Indexed Jobs](#elastic-indexed-jobs) để biết chi tiết về hạn chế này.
- Các Job bị tạm dừng vẫn giữ lại các đối tượng `Workload` và `PodGroup` của chúng;
  các đối tượng này không bị xóa khi tạm dừng hay được tạo lại khi tiếp tục.

## Xử lý lỗi Pod và container (Handling Pod and container failures)

Một container trong Pod có thể thất bại vì một số lý do, chẳng hạn tiến trình trong nó
thoát với mã thoát (exit code) khác 0, hoặc container bị kill vì vượt quá giới hạn
memory, v.v. Nếu điều này xảy ra và `.spec.template.spec.restartPolicy = "OnFailure"`,
Pod vẫn ở lại trên node nhưng container được chạy lại. Do đó, chương trình của bạn cần
xử lý trường hợp nó được khởi động lại tại chỗ, hoặc nếu không thì hãy chỉ định
`.spec.template.spec.restartPolicy = "Never"`.
Xem [vòng đời pod](./47-pod-lifecycle-vi.md#restart-policy) để biết thêm thông tin về
`restartPolicy`.

Toàn bộ một Pod cũng có thể thất bại, vì một số lý do, chẳng hạn khi pod bị đẩy khỏi
node (node bị nâng cấp, khởi động lại, bị xóa, v.v.), hoặc nếu một container của Pod
thất bại và `.spec.template.spec.restartPolicy = "Never"`. Khi một Pod thất bại, Job
controller khởi động một Pod mới. Điều này có nghĩa là ứng dụng của bạn cần xử lý trường
hợp nó được khởi động lại trong một pod mới. Đặc biệt, nó cần xử lý các file tạm, các
khóa (lock), output dở dang và những thứ tương tự do các lần chạy trước để lại.

Mặc định, mỗi lần pod thất bại được tính vào giới hạn `.spec.backoffLimit`, xem
[chính sách backoff khi Pod lỗi](#pod-backoff-failure-policy). Tuy nhiên, bạn có thể
tùy chỉnh cách xử lý pod thất bại bằng cách đặt [chính sách lỗi Pod](#pod-failure-policy)
của Job.

Ngoài ra, bạn có thể chọn đếm số lần pod thất bại một cách độc lập cho từng index của
một [Indexed](#completion-mode) Job bằng cách đặt trường `.spec.backoffLimitPerIndex`
(để biết thêm thông tin, xem [giới hạn backoff theo từng index](#backoff-limit-per-index)).

Lưu ý rằng ngay cả khi bạn chỉ định `.spec.parallelism = 1` và `.spec.completions = 1`
và `.spec.template.spec.restartPolicy = "Never"`, cùng một chương trình đôi khi vẫn có
thể bị khởi động hai lần.

Nếu bạn chỉ định cả `.spec.parallelism` và `.spec.completions` đều lớn hơn 1, có thể có
nhiều pod chạy cùng một lúc. Do đó, các pod của bạn cũng phải chịu được sự chạy đồng
thời (concurrency).

Nếu bạn chỉ định trường `.spec.podFailurePolicy`, Job controller không coi một Pod đang
kết thúc (pod có trường `.metadata.deletionTimestamp` được đặt) là thất bại cho đến khi
Pod đó ở trạng thái cuối (terminal) — tức `.status.phase` của nó là `Failed` hoặc
`Succeeded`. Tuy nhiên, Job controller tạo một Pod thay thế ngay khi việc kết thúc trở
nên rõ ràng. Khi pod đã kết thúc, Job controller đánh giá `.backoffLimit` và
`.podFailurePolicy` cho Job liên quan, có tính đến Pod vừa kết thúc này.

Nếu một trong hai yêu cầu này không được thỏa mãn, Job controller tính một Pod đang kết
thúc là thất bại ngay lập tức, ngay cả khi Pod đó sau này kết thúc với `phase: "Succeeded"`.

### Chính sách backoff khi Pod lỗi (Pod backoff failure policy) {#pod-backoff-failure-policy}

Có những tình huống bạn muốn cho một Job thất bại sau một số lần thử lại, do có lỗi
logic trong cấu hình, v.v. Để làm vậy, đặt `.spec.backoffLimit` để chỉ định số lần thử
lại trước khi coi Job là thất bại.

`.spec.backoffLimit` mặc định được đặt là 6, trừ khi
[giới hạn backoff theo từng index](#backoff-limit-per-index) (chỉ với Indexed Job) được
chỉ định. Khi `.spec.backoffLimitPerIndex` được chỉ định, `.spec.backoffLimit` mặc định
là 2147483647 (MaxInt32).

Các Pod thất bại gắn với Job được Job controller tạo lại với độ trễ backoff tăng theo
cấp số nhân (10s, 20s, 40s ...) với mức trần là sáu phút.

Số lần thử lại được tính theo hai cách:

- Số Pod có `.status.phase = "Failed"`.
- Khi dùng `restartPolicy = "OnFailure"`, số lần thử lại trong tất cả các container của
  các Pod có `.status.phase` bằng `Pending` hoặc `Running`.

Nếu một trong hai cách tính đạt tới `.spec.backoffLimit`, Job bị coi là thất bại.

> **Ghi chú:**
> Nếu Job của bạn có `restartPolicy = "OnFailure"`, hãy nhớ rằng Pod đang chạy job sẽ bị
> chấm dứt khi đã đạt tới giới hạn backoff của job. Điều này có thể khiến việc debug
> chương trình của Job khó khăn hơn. Chúng tôi khuyên bạn đặt `restartPolicy = "Never"`
> khi debug Job, hoặc dùng một hệ thống logging để đảm bảo output từ các Job thất bại
> không bị mất một cách vô ý.

### Giới hạn backoff theo từng index (Backoff limit per index) {#backoff-limit-per-index}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]`

Khi chạy một [Indexed](#completion-mode) Job, bạn có thể chọn xử lý các lần thử lại khi
pod thất bại một cách độc lập cho từng index. Để làm vậy, đặt
`.spec.backoffLimitPerIndex` để chỉ định số lần pod thất bại tối đa cho mỗi index.

Khi giới hạn backoff theo index bị vượt quá đối với một index, Kubernetes coi index đó
là thất bại và thêm nó vào trường `.status.failedIndexes`. Các index thành công — những
index có pod được thực thi thành công — được ghi vào trường `.status.completedIndexes`,
bất kể bạn có đặt trường `backoffLimitPerIndex` hay không.

Lưu ý rằng một index thất bại không làm gián đoạn việc thực thi các index khác.
Khi tất cả các index đã kết thúc đối với một Job mà bạn chỉ định giới hạn backoff theo
index, nếu có ít nhất một trong các index đó thất bại, Job controller đánh dấu toàn bộ
Job là thất bại, bằng cách đặt condition Failed trong status. Job bị đánh dấu thất bại
ngay cả khi một số — thậm chí có thể gần như tất cả — các index được xử lý thành công.

Bạn có thể giới hạn thêm số index tối đa bị đánh dấu thất bại bằng cách đặt trường
`.spec.maxFailedIndexes`.
Khi số index thất bại vượt quá trường `maxFailedIndexes`, Job controller kích hoạt việc
chấm dứt tất cả các Pod còn đang chạy của Job đó. Khi tất cả các pod đã kết thúc, toàn
bộ Job bị Job controller đánh dấu thất bại, bằng cách đặt condition Failed trong status
của Job.

Dưới đây là một manifest ví dụ cho một Job định nghĩa `backoffLimitPerIndex`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-backoff-limit-per-index-example
spec:
  completions: 10
  parallelism: 3
  completionMode: Indexed  # bắt buộc đối với tính năng này
  backoffLimitPerIndex: 1  # số lần thất bại tối đa cho mỗi index
  maxFailedIndexes: 5      # số index thất bại tối đa trước khi chấm dứt việc thực thi Job
  template:
    spec:
      restartPolicy: Never # bắt buộc đối với tính năng này
      containers:
      - name: example
        image: python
        command:           # Job thất bại vì có ít nhất một index thất bại
                           # (ở đây tất cả các index chẵn đều thất bại),
                           # nhưng tất cả các index đều được thực thi
                           # vì maxFailedIndexes không bị vượt quá.
        - python3
        - -c
        - |
          import os, sys
          print("Hello world")
          if int(os.environ.get("JOB_COMPLETION_INDEX")) % 2 == 0:
            sys.exit(1)
```

Trong ví dụ trên, Job controller cho phép mỗi index được khởi động lại một lần. Khi tổng
số index thất bại vượt quá 5, toàn bộ Job bị chấm dứt.

Khi job kết thúc, status của Job trông như sau:

```sh
kubectl get -o yaml job job-backoff-limit-per-index-example
```

```yaml
  status:
    completedIndexes: 1,3,5,7,9
    failedIndexes: 0,2,4,6,8
    succeeded: 5          # 1 pod thành công cho mỗi index trong 5 index thành công
    failed: 10            # 2 pod thất bại (1 lần thử lại) cho mỗi index trong 5 index thất bại
    conditions:
    - message: Job has failed indexes
      reason: FailedIndexes
      status: "True"
      type: FailureTarget
    - message: Job has failed indexes
      reason: FailedIndexes
      status: "True"
      type: Failed
```

Job controller thêm condition `FailureTarget` vào Job để kích hoạt
[việc kết thúc và dọn dẹp Job](#job-termination-and-cleanup). Khi tất cả các Pod của Job
đã kết thúc, Job controller thêm condition `Failed` với cùng các giá trị `reason` và
`message` như condition `FailureTarget`. Để biết chi tiết, xem
[Chấm dứt các Pod của Job](#termination-of-job-pods).

Ngoài ra, bạn có thể muốn dùng backoff theo từng index cùng với một
[chính sách lỗi Pod](#pod-failure-policy). Khi dùng backoff theo từng index, có một
hành động `FailIndex` mới cho phép bạn tránh các lần thử lại không cần thiết bên trong
một index.

### Chính sách lỗi Pod (Pod failure policy) {#pod-failure-policy}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.31 [stable]`

Chính sách lỗi Pod (Pod failure policy), được định nghĩa bằng trường
`.spec.podFailurePolicy`, cho phép cluster của bạn xử lý các lần Pod thất bại dựa trên
mã thoát của container và các condition của Pod.

Trong một số tình huống, bạn có thể muốn kiểm soát việc xử lý Pod thất bại tốt hơn mức
mà [chính sách backoff khi Pod lỗi](#pod-backoff-failure-policy) — vốn dựa trên
`.spec.backoffLimit` của Job — cung cấp. Dưới đây là một vài ví dụ về trường hợp sử dụng:

* Để tối ưu chi phí chạy workload bằng cách tránh các lần khởi động lại Pod không cần
  thiết, bạn có thể chấm dứt một Job ngay khi một trong các Pod của nó thất bại với mã
  thoát cho thấy có lỗi phần mềm (software bug).
* Để đảm bảo Job của bạn hoàn thành ngay cả khi có gián đoạn (disruption), bạn có thể bỏ
  qua các lần Pod thất bại do gián đoạn gây ra (chẳng hạn chiếm chỗ (preemption), trục
  xuất do API khởi xướng (API-initiated eviction) hoặc trục xuất dựa trên taint) để
  chúng không bị tính vào giới hạn số lần thử lại `.spec.backoffLimit`.

Bạn có thể cấu hình một chính sách lỗi Pod, trong trường `.spec.podFailurePolicy`, để
đáp ứng các trường hợp sử dụng trên. Chính sách này có thể xử lý các lần Pod thất bại
dựa trên mã thoát của container và các condition của Pod.

Dưới đây là một manifest cho một Job định nghĩa `podFailurePolicy`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-pod-failure-policy-example
spec:
  completions: 12
  parallelism: 3
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: main
        image: docker.io/library/bash:5
        command: ["bash"]        # lệnh ví dụ mô phỏng một bug kích hoạt hành động FailJob
        args:
        - -c
        - echo "Hello world!" && sleep 5 && exit 42
  backoffLimit: 6
  podFailurePolicy:
    rules:
    - action: FailJob
      onExitCodes:
        containerName: main      # tùy chọn
        operator: In             # một trong: In, NotIn
        values: [42]
    - action: Ignore             # một trong: Ignore, FailJob, Count
      onPodConditions:
      - type: DisruptionTarget   # biểu thị sự gián đoạn của Pod
```

Trong ví dụ trên, quy tắc đầu tiên của chính sách lỗi Pod chỉ định rằng Job sẽ bị đánh
dấu thất bại nếu container `main` thất bại với mã thoát 42. Sau đây là các quy tắc dành
riêng cho container `main`:

- mã thoát 0 nghĩa là container thành công
- mã thoát 42 nghĩa là **toàn bộ Job** thất bại
- bất kỳ mã thoát nào khác thể hiện rằng container thất bại, và do đó toàn bộ Pod thất
  bại. Pod sẽ được tạo lại nếu tổng số lần khởi động lại dưới `backoffLimit`. Nếu đạt
  tới `backoffLimit`, **toàn bộ Job** thất bại.

> **Ghi chú:**
> Vì Pod template chỉ định `restartPolicy: Never`, kubelet không khởi động lại container
> `main` trong Pod cụ thể đó.

Quy tắc thứ hai của chính sách lỗi Pod, chỉ định hành động `Ignore` cho các Pod thất bại
có condition `DisruptionTarget`, loại các gián đoạn Pod ra khỏi việc bị tính vào giới
hạn số lần thử lại `.spec.backoffLimit`.

> **Ghi chú:**
> Nếu Job thất bại, dù là do chính sách lỗi Pod hay chính sách backoff khi Pod lỗi, và
> Job đang chạy nhiều Pod, Kubernetes chấm dứt tất cả các Pod trong Job đó còn đang
> Pending hoặc Running.

Dưới đây là một số yêu cầu và ngữ nghĩa của API:

- nếu bạn muốn dùng trường `.spec.podFailurePolicy` cho một Job, bạn cũng phải định
  nghĩa pod template của Job đó với `.spec.restartPolicy` được đặt là `Never`.
- các quy tắc chính sách lỗi Pod mà bạn chỉ định trong `spec.podFailurePolicy.rules`
  được đánh giá theo thứ tự. Khi một quy tắc khớp với một lần Pod thất bại, các quy tắc
  còn lại bị bỏ qua. Khi không có quy tắc nào khớp với lần Pod thất bại, cách xử lý mặc
  định được áp dụng.
- bạn có thể muốn giới hạn một quy tắc cho một container cụ thể bằng cách chỉ định tên
  của nó trong `spec.podFailurePolicy.rules[*].onExitCodes.containerName`. Khi không
  chỉ định, quy tắc áp dụng cho tất cả các container. Khi được chỉ định, tên đó phải
  khớp với tên của một container hoặc `initContainer` trong Pod template.
- bạn có thể chỉ định hành động được thực hiện khi một chính sách lỗi Pod khớp qua
  `spec.podFailurePolicy.rules[*].action`. Các giá trị có thể là:
  - `FailJob`: dùng để chỉ ra rằng job của Pod nên bị đánh dấu là Failed và tất cả các
    Pod đang chạy nên bị chấm dứt.
  - `Ignore`: dùng để chỉ ra rằng bộ đếm cho `.spec.backoffLimit` không nên bị tăng và
    một Pod thay thế nên được tạo ra.
  - `Count`: dùng để chỉ ra rằng Pod nên được xử lý theo cách mặc định. Bộ đếm cho
    `.spec.backoffLimit` nên được tăng.
  - `FailIndex`: dùng hành động này cùng với
    [giới hạn backoff theo từng index](#backoff-limit-per-index) để tránh các lần thử
    lại không cần thiết bên trong index của pod thất bại.

> **Ghi chú:**
> Khi bạn dùng `podFailurePolicy`, job controller chỉ khớp các Pod ở phase `Failed`.
> Các Pod có deletion timestamp mà chưa ở phase cuối (`Failed` hoặc `Succeeded`) được
> coi là vẫn đang kết thúc. Điều này ngụ ý rằng các pod đang kết thúc vẫn giữ
> [finalizer theo dõi](#job-tracking-with-finalizers) cho đến khi chúng đạt phase cuối.
> Từ Kubernetes 1.27, kubelet chuyển các pod bị xóa sang phase cuối
> (xem: [Phase của Pod](./47-pod-lifecycle-vi.md#pod-phase)). Điều này đảm bảo các pod
> bị xóa được Job controller gỡ các finalizer.

> **Ghi chú:**
> Bắt đầu từ Kubernetes v1.28, khi chính sách lỗi Pod được sử dụng, Job controller chỉ
> tạo lại các Pod đang kết thúc khi các Pod này đạt tới phase cuối `Failed`. Hành vi này
> tương tự `podReplacementPolicy: Failed`. Để biết thêm thông tin, xem
> [Chính sách thay thế Pod](#pod-replacement-policy).

Khi bạn dùng `podFailurePolicy` và Job thất bại do pod khớp với quy tắc có hành động
`FailJob`, Job controller kích hoạt quá trình kết thúc Job bằng cách thêm condition
`FailureTarget`. Để biết thêm chi tiết, xem
[Kết thúc và dọn dẹp Job](#job-termination-and-cleanup).

## Chính sách thành công (Success policy) {#success-policy}

Khi tạo một Indexed Job, bạn có thể định nghĩa khi nào Job có thể được tuyên bố là thành
công bằng `.spec.successPolicy`, dựa trên các pod đã thành công.

Mặc định, một Job thành công khi số Pod thành công bằng `.spec.completions`.
Dưới đây là một số tình huống mà bạn có thể muốn có sự kiểm soát bổ sung cho việc tuyên
bố Job thành công:

* Khi chạy các mô phỏng (simulation) với các tham số khác nhau, bạn có thể không cần tất
  cả các mô phỏng đều thành công thì Job tổng thể mới thành công.
* Khi theo mẫu leader-worker, chỉ sự thành công của leader mới quyết định sự thành công
  hay thất bại của Job. Ví dụ về điều này là các framework như MPI và PyTorch, v.v.

Bạn có thể cấu hình một chính sách thành công, trong trường `.spec.successPolicy`, để
đáp ứng các trường hợp sử dụng trên. Chính sách này có thể xử lý sự thành công của Job
dựa trên các pod đã thành công. Sau khi Job đạt chính sách thành công, job controller
chấm dứt các Pod còn sót lại. Một chính sách thành công được định nghĩa bằng các quy
tắc. Mỗi quy tắc có thể có một trong các dạng sau:

* Khi bạn chỉ định chỉ mỗi `succeededIndexes`, khi tất cả các index được chỉ định trong
  `succeededIndexes` thành công, job controller đánh dấu Job là thành công.
  `succeededIndexes` phải là một danh sách các khoảng (interval) từ 0 đến
  `.spec.completions-1`.
* Khi bạn chỉ định chỉ mỗi `succeededCount`, khi số index thành công đạt tới
  `succeededCount`, job controller đánh dấu Job là thành công.
* Khi bạn chỉ định cả `succeededIndexes` và `succeededCount`, khi số index thành công
  trong tập con các index được chỉ định tại `succeededIndexes` đạt tới `succeededCount`,
  job controller đánh dấu Job là thành công.

Lưu ý rằng khi bạn chỉ định nhiều quy tắc trong `.spec.successPolicy.rules`, job
controller đánh giá các quy tắc theo thứ tự. Khi Job đạt một quy tắc, job controller bỏ
qua các quy tắc còn lại.

Dưới đây là một manifest cho một Job với `successPolicy`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-success
spec:
  parallelism: 10
  completions: 10
  completionMode: Indexed # Bắt buộc đối với success policy
  successPolicy:
    rules:
      - succeededIndexes: 0,2-3
        succeededCount: 1
  template:
    spec:
      containers:
      - name: main
        image: python
        command:          # Miễn là ít nhất một trong các Pod có index 0, 2 và 3 thành công,
                          # thì Job tổng thể là thành công.
          - python3
          - -c
          - |
            import os, sys
            if os.environ.get("JOB_COMPLETION_INDEX") == "2":
              sys.exit(0)
            else:
              sys.exit(1)
      restartPolicy: Never
```

Trong ví dụ trên, cả `succeededIndexes` và `succeededCount` đều được chỉ định. Do đó,
job controller sẽ đánh dấu Job là thành công và chấm dứt các Pod còn sót lại khi một
trong các index được chỉ định — 0, 2 hoặc 3 — thành công.
Job đạt chính sách thành công sẽ nhận condition `SuccessCriteriaMet` với reason
`SuccessPolicy`. Sau khi lệnh gỡ bỏ các Pod còn sót lại được đưa ra, Job nhận condition
`Complete`.

Lưu ý rằng `succeededIndexes` được biểu diễn dưới dạng các khoảng phân tách bằng dấu
gạch nối (hyphen). Các số được liệt kê bằng phần tử đầu và phần tử cuối của dãy, phân
tách bằng dấu gạch nối.

> **Ghi chú:**
> Khi bạn chỉ định cả chính sách thành công lẫn một số chính sách kết thúc như
> `.spec.backoffLimit` và `.spec.podFailurePolicy`, khi Job đạt một trong hai loại chính
> sách, job controller tuân theo chính sách kết thúc và bỏ qua chính sách thành công.

## Kết thúc và dọn dẹp Job (Job termination and cleanup) {#job-termination-and-cleanup}

Khi một Job hoàn tất, không Pod nào được tạo thêm, nhưng các Pod cũng
[thường](#pod-backoff-failure-policy) không bị xóa. Việc giữ chúng lại cho phép bạn vẫn
xem được log của các pod đã hoàn thành để kiểm tra lỗi, cảnh báo hay các output chẩn
đoán khác. Đối tượng job cũng vẫn còn lại sau khi hoàn tất để bạn có thể xem status của
nó. Người dùng có trách nhiệm xóa các job cũ sau khi đã ghi nhận status của chúng. Xóa
job bằng `kubectl` (ví dụ `kubectl delete jobs/pi` hoặc `kubectl delete -f ./job.yaml`).
Khi bạn xóa job bằng `kubectl`, tất cả các pod mà nó đã tạo cũng bị xóa theo.

Mặc định, một Job sẽ chạy không gián đoạn trừ khi một Pod thất bại
(`restartPolicy=Never`) hoặc một container thoát với lỗi (`restartPolicy=OnFailure`),
lúc đó Job tuân theo `.spec.backoffLimit` được mô tả ở trên. Khi đã đạt tới
`.spec.backoffLimit`, Job sẽ bị đánh dấu là thất bại và mọi Pod đang chạy sẽ bị chấm dứt.

Một cách khác để chấm dứt một Job là đặt một thời hạn hoạt động (active deadline).
Thực hiện bằng cách đặt trường `.spec.activeDeadlineSeconds` của Job thành một số giây.
`activeDeadlineSeconds` áp dụng cho toàn bộ thời gian tồn tại của job, bất kể có bao
nhiêu Pod được tạo. Khi một Job đạt tới `activeDeadlineSeconds`, tất cả các Pod đang
chạy của nó bị chấm dứt và status của Job sẽ trở thành `type: Failed` với
`reason: DeadlineExceeded`.

Lưu ý rằng `.spec.activeDeadlineSeconds` của Job có quyền ưu tiên cao hơn
`.spec.backoffLimit` của nó. Do đó, một Job đang thử lại một hoặc nhiều Pod thất bại sẽ
không triển khai thêm Pod khi đã đạt tới giới hạn thời gian được chỉ định bởi
`activeDeadlineSeconds`, ngay cả khi chưa đạt tới `backoffLimit`.

Ví dụ:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-timeout
spec:
  backoffLimit: 5
  activeDeadlineSeconds: 100
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

Lưu ý rằng cả Job spec lẫn [Pod template spec](./50-init-containers-vi.md) bên trong
Job đều có trường `activeDeadlineSeconds`. Hãy đảm bảo bạn đặt trường này ở đúng cấp.

Hãy nhớ rằng `restartPolicy` áp dụng cho Pod, chứ không áp dụng cho chính Job: không có
việc tự động khởi động lại Job khi status của Job là `type: Failed`. Nghĩa là, các cơ
chế chấm dứt Job được kích hoạt bởi `.spec.activeDeadlineSeconds` và `.spec.backoffLimit`
dẫn đến sự thất bại vĩnh viễn của Job, đòi hỏi can thiệp thủ công để xử lý.

### Các condition kết thúc của Job (Terminal Job conditions) {#terminal-job-conditions}

Một Job có hai trạng thái cuối (terminal state) khả dĩ, mỗi trạng thái có một Job
condition tương ứng:

* Succeeded (thành công): Job condition `Complete`
* Failed (thất bại): Job condition `Failed`

Job thất bại vì các lý do sau:

- Số lần Pod thất bại vượt quá `.spec.backoffLimit` được chỉ định trong Job spec.
  Để biết chi tiết, xem [Chính sách backoff khi Pod lỗi](#pod-backoff-failure-policy).
- Thời gian chạy của Job vượt quá `.spec.activeDeadlineSeconds` được chỉ định.
- Một Indexed Job dùng `.spec.backoffLimitPerIndex` có các index thất bại.
  Để biết chi tiết, xem [Giới hạn backoff theo từng index](#backoff-limit-per-index).
- Số index thất bại trong Job vượt quá `spec.maxFailedIndexes` được chỉ định.
  Để biết chi tiết, xem [Giới hạn backoff theo từng index](#backoff-limit-per-index).
- Một Pod thất bại khớp với một quy tắc trong `.spec.podFailurePolicy` có hành động
  `FailJob`. Để biết chi tiết về cách các quy tắc chính sách lỗi Pod có thể ảnh hưởng
  tới việc đánh giá thất bại, xem [Chính sách lỗi Pod](#pod-failure-policy).

Job thành công vì các lý do sau:

- Số Pod thành công đạt tới `.spec.completions` được chỉ định.
- Các tiêu chí được chỉ định trong `.spec.successPolicy` được thỏa mãn. Để biết chi
  tiết, xem [Chính sách thành công](#success-policy).

Trong Kubernetes v1.31 trở đi, Job controller trì hoãn việc thêm các condition cuối —
`Failed` hoặc `Complete` — cho đến khi tất cả các Pod của Job đã kết thúc.

Trong Kubernetes v1.30 trở về trước, Job controller thêm condition cuối `Complete` hoặc
`Failed` của Job ngay khi quá trình kết thúc Job được kích hoạt và tất cả các finalizer
của Pod đã được gỡ. Tuy nhiên, một số Pod vẫn có thể đang chạy hoặc đang kết thúc vào
thời điểm condition cuối được thêm.

Trong Kubernetes v1.31 trở đi, controller chỉ thêm các condition cuối của Job _sau khi_
tất cả các Pod đã kết thúc. Bạn có thể kiểm soát hành vi này bằng các
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`JobManagedBy` và `JobPodReplacementPolicy` (cả hai được bật mặc định).

### Chấm dứt các Pod của Job (Termination of Job pods) {#termination-of-job-pods}

Job controller thêm condition `FailureTarget` hoặc condition `SuccessCriteriaMet` vào
Job để kích hoạt việc chấm dứt Pod sau khi Job đạt tiêu chí thành công hoặc thất bại.

Các yếu tố như `terminationGracePeriodSeconds` có thể làm tăng khoảng thời gian từ lúc
Job controller thêm condition `FailureTarget` hoặc condition `SuccessCriteriaMet` đến
lúc tất cả các Pod của Job kết thúc và Job controller thêm một
[condition cuối](#terminal-job-conditions) (`Failed` hoặc `Complete`).

Bạn có thể dùng condition `FailureTarget` hoặc `SuccessCriteriaMet` để đánh giá Job đã
thất bại hay thành công mà không phải chờ controller thêm condition cuối.

Ví dụ, bạn có thể muốn quyết định khi nào tạo một Job thay thế cho một Job đã thất bại.
Nếu bạn thay thế Job thất bại ngay khi condition `FailureTarget` xuất hiện, Job thay
thế sẽ chạy sớm hơn, nhưng có thể dẫn tới việc các Pod của Job thất bại và của Job thay
thế chạy cùng lúc, tiêu tốn thêm tài nguyên tính toán.

Ngược lại, nếu cluster của bạn có dung lượng tài nguyên hạn chế, bạn có thể chọn chờ
đến khi condition `Failed` xuất hiện trên Job — điều này làm Job thay thế chạy chậm hơn
nhưng đảm bảo bạn tiết kiệm tài nguyên bằng cách chờ đến khi tất cả các Pod thất bại
đã bị gỡ bỏ.

## Tự động dọn dẹp các Job đã hoàn tất (Clean up finished jobs automatically) {#clean-up-finished-jobs-automatically}

Các Job đã hoàn tất thường không còn cần thiết trong hệ thống nữa. Giữ chúng lại trong
hệ thống sẽ gây áp lực lên API server. Nếu các Job được quản lý trực tiếp bởi một
controller cấp cao hơn, chẳng hạn [CronJob](./69-cron-jobs-vi.md), các Job có thể được
CronJob dọn dẹp dựa trên chính sách dọn dẹp theo dung lượng (capacity-based cleanup
policy) đã được chỉ định.

### Cơ chế TTL cho các Job đã hoàn tất (TTL mechanism for finished Jobs)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [stable]`

Một cách khác để tự động dọn dẹp các Job đã hoàn tất (`Complete` hoặc `Failed`) là dùng
cơ chế TTL do [TTL controller](./68-ttlafterfinished-vi.md) cung cấp cho các tài nguyên
đã hoàn tất, bằng cách chỉ định trường `.spec.ttlSecondsAfterFinished` của Job.

Khi TTL controller dọn dẹp Job, nó sẽ xóa Job theo tầng (cascading), tức là xóa các đối
tượng phụ thuộc của Job, chẳng hạn các Pod, cùng với Job. Lưu ý rằng khi Job bị xóa,
các đảm bảo về vòng đời của nó, chẳng hạn các finalizer, sẽ được tuân thủ.

Ví dụ:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-ttl
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

Job `pi-with-ttl` sẽ đủ điều kiện để được tự động xóa `100` giây sau khi nó hoàn tất.

Nếu trường này được đặt là `0`, Job sẽ đủ điều kiện được tự động xóa ngay sau khi hoàn
tất. Nếu trường này không được đặt, Job sẽ không được TTL controller dọn dẹp sau khi nó
hoàn tất.

> **Ghi chú:**
> Nên đặt trường `ttlSecondsAfterFinished` vì các job không được quản lý (unmanaged job
> — các Job bạn tạo trực tiếp, chứ không phải gián tiếp thông qua các workload API khác
> như CronJob) có chính sách xóa mặc định là `orphanDependents`, khiến các Pod do một
> unmanaged Job tạo ra bị bỏ lại sau khi Job đó đã bị xóa hoàn toàn.
> Dù control plane cuối cùng cũng
> [thu gom rác](./47-pod-lifecycle-vi.md#pod-garbage-collection) các Pod của một Job đã
> bị xóa sau khi chúng thất bại hoặc hoàn thành, đôi khi những pod còn sót lại đó có thể
> gây suy giảm hiệu năng của cluster, hoặc trong trường hợp xấu nhất khiến cluster
> ngừng hoạt động do sự suy giảm này.
>
> Bạn có thể dùng [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/) và
> [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) để đặt
> mức trần cho lượng tài nguyên mà một namespace cụ thể có thể tiêu thụ.

## Các mẫu sử dụng Job (Job patterns) {#job-patterns}

Đối tượng Job có thể được dùng để xử lý một tập các *hạng mục công việc* (work item)
độc lập nhưng có liên quan với nhau. Đó có thể là các email cần gửi, các frame cần
render, các file cần chuyển mã (transcode), các dải khóa (range of keys) trong cơ sở dữ
liệu NoSQL cần quét, v.v.

Trong một hệ thống phức tạp, có thể có nhiều tập hạng mục công việc khác nhau. Ở đây
chúng ta chỉ xét một tập hạng mục công việc mà người dùng muốn quản lý cùng nhau — một
*batch job*.

Có vài mẫu (pattern) khác nhau cho tính toán song song, mỗi mẫu có điểm mạnh và điểm
yếu riêng. Các đánh đổi (tradeoff) là:

- Một đối tượng Job cho mỗi hạng mục công việc, so với một đối tượng Job duy nhất cho
  tất cả các hạng mục. Một Job cho mỗi hạng mục tạo ra chi phí quản lý (overhead) cho
  người dùng và cho hệ thống khi phải quản lý số lượng lớn đối tượng Job.
  Một Job duy nhất cho tất cả các hạng mục sẽ tốt hơn khi số lượng hạng mục lớn.
- Số Pod được tạo bằng số hạng mục công việc, so với việc mỗi Pod có thể xử lý nhiều
  hạng mục. Khi số Pod bằng số hạng mục, các Pod thường ít phải sửa đổi code và
  container hiện có hơn. Để mỗi Pod xử lý nhiều hạng mục sẽ tốt hơn khi số lượng hạng
  mục lớn.
- Một số cách tiếp cận dùng hàng đợi công việc (work queue). Điều này đòi hỏi phải chạy
  một dịch vụ hàng đợi, và sửa đổi chương trình hoặc container hiện có để nó dùng hàng
  đợi công việc. Các cách tiếp cận khác thì dễ thích ứng hơn với một ứng dụng đã được
  container hóa sẵn.
- Khi Job được gắn với một
  [headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services),
  bạn có thể cho phép các Pod trong một Job giao tiếp với nhau để cùng phối hợp trong
  một phép tính toán.

Các đánh đổi được tóm tắt ở đây, với cột 2 đến 4 tương ứng với các đánh đổi bên trên.
Tên các mẫu cũng là link tới các ví dụ và mô tả chi tiết hơn.

|                  Mẫu (Pattern)                  | Một đối tượng Job duy nhất | Số pod ít hơn số hạng mục công việc? | Dùng ứng dụng không cần sửa đổi? |
| ----------------------------------------------- |:-----------------:|:---------------------------:|:-------------------:|
| [Hàng đợi với một Pod cho mỗi work item]        |         ✓         |                             |      đôi khi        |
| [Hàng đợi với số lượng Pod thay đổi]            |         ✓         |             ✓               |                     |
| [Indexed Job với phân công việc tĩnh]           |         ✓         |                             |          ✓          |
| [Job với giao tiếp Pod-to-Pod]                  |         ✓         |         đôi khi             |      đôi khi        |
| [Mở rộng từ Job template]                       |                   |                             |          ✓          |

Khi bạn chỉ định số lần hoàn thành bằng `.spec.completions`, mỗi Pod do Job controller
tạo ra có một
[`spec`](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)
giống hệt nhau. Điều này có nghĩa là tất cả các pod của một tác vụ sẽ có cùng command
line và cùng image, cùng các volume, và (gần như) cùng các biến môi trường. Các mẫu này
là những cách khác nhau để sắp xếp cho các pod làm việc trên những phần việc khác nhau.

Bảng này cho thấy các thiết lập bắt buộc cho `.spec.parallelism` và `.spec.completions`
đối với mỗi mẫu. Ở đây, `W` là số hạng mục công việc.

|             Mẫu (Pattern)                       | `.spec.completions` |  `.spec.parallelism` |
| ----------------------------------------------- |:-------------------:|:--------------------:|
| [Hàng đợi với một Pod cho mỗi work item]        |          W          |        bất kỳ        |
| [Hàng đợi với số lượng Pod thay đổi]            |         null        |        bất kỳ        |
| [Indexed Job với phân công việc tĩnh]           |          W          |        bất kỳ        |
| [Job với giao tiếp Pod-to-Pod]                  |          W          |         W            |
| [Mở rộng từ Job template]                       |          1          |       nên là 1       |

## Sử dụng nâng cao (Advanced usage)

### Tạm dừng một Job (Suspending a Job) {#suspending-a-job}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Khi một Job được tạo, Job controller sẽ lập tức bắt đầu tạo các Pod để đáp ứng yêu cầu
của Job và tiếp tục làm như vậy cho đến khi Job hoàn tất. Tuy nhiên, bạn có thể muốn
tạm dừng việc thực thi một Job và tiếp tục nó sau đó, hoặc khởi tạo các Job ở trạng
thái tạm dừng và để một controller tùy chỉnh quyết định sau đó khi nào bắt đầu chúng.

Để tạm dừng một Job, bạn có thể cập nhật trường `.spec.suspend` của Job thành true;
sau này, khi muốn tiếp tục nó, cập nhật lại thành false. Tạo một Job với `.spec.suspend`
được đặt là true sẽ tạo Job ở trạng thái tạm dừng.

Trong Kubernetes 1.35 trở đi, trường `.status.startTime` được xóa khi Job bị tạm dừng
nếu feature gate
[MutableSchedulingDirectivesForSuspendedJobs](#mutable-scheduling-directives-for-suspended-jobs)
được bật.

Khi một Job được tiếp tục sau khi tạm dừng, trường `.status.startTime` của nó sẽ được
đặt lại thành thời gian hiện tại. Điều này có nghĩa là bộ đếm thời gian
`.spec.activeDeadlineSeconds` sẽ bị dừng và đặt lại khi Job bị tạm dừng rồi được tiếp
tục.

Khi bạn tạm dừng một Job, mọi Pod đang chạy chưa có trạng thái `Completed` sẽ bị
[chấm dứt](./47-pod-lifecycle-vi.md#pod-termination) bằng tín hiệu SIGTERM. Thời gian
kết thúc êm thấm (graceful termination period) của Pod sẽ được tuân thủ và Pod của bạn
phải xử lý tín hiệu này trong khoảng thời gian đó. Việc này có thể bao gồm lưu lại tiến
độ để dùng sau hoặc hoàn tác các thay đổi. Các Pod bị chấm dứt theo cách này sẽ không
được tính vào số `completions` của Job.

Một định nghĩa Job ở trạng thái tạm dừng có thể trông như sau:

```shell
kubectl get job myjob -o yaml
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
spec:
  suspend: true
  parallelism: 1
  completions: 5
  template:
    spec:
      ...
```

Bạn cũng có thể bật/tắt việc tạm dừng Job bằng cách patch Job qua dòng lệnh.

Tạm dừng một Job đang hoạt động:

```shell
kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":true}}'
```

Tiếp tục một Job đã tạm dừng:

```shell
kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":false}}'
```

Status của Job có thể được dùng để xác định một Job có đang bị tạm dừng hay đã từng bị
tạm dừng trong quá khứ hay không:

```shell
kubectl get jobs/myjob -o yaml
```

```yaml
apiVersion: batch/v1
kind: Job
# .metadata và .spec được lược bỏ
status:
  conditions:
  - lastProbeTime: "2021-02-05T13:14:33Z"
    lastTransitionTime: "2021-02-05T13:14:33Z"
    status: "True"
    type: Suspended
  startTime: "2021-02-05T13:13:48Z"
```

Job condition kiểu "Suspended" với status "True" nghĩa là Job đang bị tạm dừng; trường
`lastTransitionTime` có thể được dùng để xác định Job đã bị tạm dừng trong bao lâu. Nếu
status của condition đó là "False", thì Job trước đó từng bị tạm dừng và hiện đang
chạy. Nếu một condition như vậy không tồn tại trong status của Job, Job chưa bao giờ bị
dừng.

Các event cũng được tạo ra khi Job bị tạm dừng và được tiếp tục:

```shell
kubectl describe jobs/myjob
```

```
Name:           myjob
...
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  12m   job-controller  Created pod: myjob-hlrpl
  Normal  SuccessfulDelete  11m   job-controller  Deleted pod: myjob-hlrpl
  Normal  Suspended         11m   job-controller  Job suspended
  Normal  SuccessfulCreate  3s    job-controller  Created pod: myjob-jvb44
  Normal  Resumed           3s    job-controller  Job resumed
```

Bốn event cuối, cụ thể là các event "Suspended" và "Resumed", là kết quả trực tiếp của
việc thay đổi trường `.spec.suspend`. Trong khoảng thời gian giữa hai event này, chúng
ta thấy không có Pod nào được tạo, nhưng việc tạo Pod đã khởi động lại ngay khi Job
được tiếp tục.

### Chỉ thị lập lịch có thể thay đổi (Mutable Scheduling Directives)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.27 [stable]`

Trong hầu hết các trường hợp, một job song song sẽ muốn các pod chạy với các ràng buộc,
như tất cả trong cùng một zone, hoặc tất cả hoặc trên GPU model x hoặc y nhưng không
trộn lẫn cả hai.

Trường [suspend](#suspending-a-job) là bước đầu tiên để đạt được các ngữ nghĩa đó.
Suspend cho phép một controller hàng đợi tùy chỉnh (custom queue controller) quyết định
khi nào một job nên bắt đầu; tuy nhiên, một khi job đã được bỏ tạm dừng (unsuspend),
controller hàng đợi tùy chỉnh không còn ảnh hưởng gì đến việc các pod của job sẽ thực
sự được đặt ở đâu.

Tính năng này cho phép cập nhật các chỉ thị lập lịch (scheduling directives) của một
Job trước khi nó bắt đầu, trao cho các controller hàng đợi tùy chỉnh khả năng ảnh hưởng
đến việc bố trí pod, đồng thời vẫn giao việc gán pod-vào-node thực tế cho
kube-scheduler.

Các trường trong pod template của Job có thể được cập nhật là node affinity, node
selector, toleration, label, annotation và
[scheduling gate](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/).

#### Chỉ thị lập lịch có thể thay đổi cho các Job bị tạm dừng (Mutable Scheduling Directives for suspended Jobs) {#mutable-scheduling-directives-for-suspended-jobs}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Trong Kubernetes 1.34 trở về trước, việc thay đổi (mutate) các chỉ thị lập lịch của Pod
chỉ được phép đối với các Job bị tạm dừng chưa từng được bỏ tạm dừng trước đó. Trong
Kubernetes 1.35, điều này được phép với bất kỳ Job bị tạm dừng nào khi feature gate
`MutableSchedulingDirectivesForSuspendedJobs` được bật.

Ngoài ra, feature gate này còn cho phép xóa trường `.status.startTime` khi
[Job bị tạm dừng](#suspending-a-job).

### Tài nguyên Pod có thể thay đổi cho các Job bị tạm dừng (Mutable Pod resources for suspended Jobs)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Một quản trị viên cluster có thể định nghĩa các kiểm soát admission trong Kubernetes để
sửa đổi các yêu cầu (request) hoặc giới hạn (limit) tài nguyên cho một Job, dựa trên
các quy tắc chính sách.

Với tính năng này, Kubernetes cũng cho phép bạn sửa đổi pod template của một
[job bị tạm dừng](#suspending-a-job) để thay đổi yêu cầu tài nguyên của các Pod trong
Job. Điều này khác với _thay đổi kích thước Pod tại chỗ_ (in-place Pod resize) — cơ chế
cho phép bạn cập nhật tài nguyên, từng Pod một, cho các Pod đang chạy.

Client đặt các request hoặc limit tài nguyên mới có thể khác với client đã tạo Job ban
đầu, và không cần phải là quản trị viên cluster.

### Chỉ định Pod selector của riêng bạn (Specifying your own Pod selector) {#specifying-your-own-pod-selector}

Thông thường, khi tạo một đối tượng Job, bạn không chỉ định `.spec.selector`.
Logic gán mặc định của hệ thống sẽ thêm trường này khi Job được tạo.
Nó chọn một giá trị selector không trùng lặp với bất kỳ job nào khác.

Tuy nhiên, trong một số trường hợp, bạn có thể cần ghi đè selector được đặt tự động
này. Để làm vậy, bạn có thể chỉ định `.spec.selector` của Job.

Hãy hết sức cẩn thận khi làm điều này. Nếu bạn chỉ định một label selector không phải
duy nhất cho các pod của Job đó, và selector này khớp với các Pod không liên quan, thì
các pod của job không liên quan có thể bị xóa, hoặc Job này có thể tính các Pod khác
vào số lần hoàn thành của nó, hoặc một hoặc cả hai Job có thể từ chối tạo Pod hay chạy
đến khi hoàn thành. Nếu một selector không duy nhất được chọn, các controller khác (ví
dụ ReplicationController) và các Pod của chúng cũng có thể hành xử theo những cách
không thể đoán trước. Kubernetes sẽ không ngăn bạn mắc sai lầm khi chỉ định
`.spec.selector`.

Dưới đây là một ví dụ về trường hợp bạn có thể muốn dùng tính năng này.

Giả sử Job `old` đang chạy. Bạn muốn các Pod hiện có tiếp tục chạy, nhưng muốn phần còn
lại của các Pod mà nó tạo ra dùng một pod template khác và muốn Job có tên mới. Bạn
không thể cập nhật Job vì các trường này không thể cập nhật được. Do đó, bạn xóa Job
`old` nhưng _để các pod của nó tiếp tục chạy_, bằng lệnh
`kubectl delete jobs/old --cascade=orphan`.
Trước khi xóa nó, bạn ghi lại selector mà nó đang dùng:

```shell
kubectl get job old -o yaml
```

Kết quả tương tự như sau:

```yaml
kind: Job
metadata:
  name: old
  ...
spec:
  selector:
    matchLabels:
      batch.kubernetes.io/controller-uid: a8f3d00d-c6d2-11e5-9f87-42010af00002
  ...
```

Sau đó bạn tạo một Job mới với tên `new` và chỉ định tường minh cùng selector đó.
Vì các Pod hiện có mang label
`batch.kubernetes.io/controller-uid=a8f3d00d-c6d2-11e5-9f87-42010af00002`,
chúng cũng chịu sự điều khiển của Job `new`.

Bạn cần chỉ định `manualSelector: true` trong Job mới, vì bạn không dùng selector mà hệ
thống thường tự động sinh cho bạn.

```yaml
kind: Job
metadata:
  name: new
  ...
spec:
  manualSelector: true
  selector:
    matchLabels:
      batch.kubernetes.io/controller-uid: a8f3d00d-c6d2-11e5-9f87-42010af00002
  ...
```

Bản thân Job mới sẽ có một uid khác với `a8f3d00d-c6d2-11e5-9f87-42010af00002`. Đặt
`manualSelector: true` nói với hệ thống rằng bạn biết mình đang làm gì và cho phép sự
không khớp này.

### Theo dõi Job bằng finalizer (Job tracking with finalizers) {#job-tracking-with-finalizers}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

Control plane theo dõi các Pod thuộc về bất kỳ Job nào và nhận biết nếu một Pod như vậy
bị gỡ khỏi API server. Để làm điều đó, Job controller tạo các Pod với finalizer
`batch.kubernetes.io/job-tracking`. Controller chỉ gỡ finalizer này sau khi Pod đã được
ghi nhận trong status của Job, cho phép Pod được gỡ bỏ bởi các controller khác hoặc
người dùng.

> **Ghi chú:**
> Xem [Pod của tôi bị kẹt ở trạng thái terminating](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
> nếu bạn thấy các pod của một Job bị kẹt với finalizer theo dõi.

### Elastic Indexed Jobs {#elastic-indexed-jobs}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.31 [stable]`

Bạn có thể scale các Indexed Job lên hoặc xuống bằng cách thay đổi đồng thời cả
`.spec.parallelism` và `.spec.completions` sao cho
`.spec.parallelism == .spec.completions`.
Khi scale xuống, Kubernetes gỡ bỏ các Pod có index cao hơn.

Các trường hợp sử dụng elastic Indexed Job bao gồm các workload dạng batch cần scale
một Indexed Job, chẳng hạn các job huấn luyện MPI, Horovod, Ray và PyTorch.

> **Ghi chú:**
> Khi feature gate [`WorkloadWithJob`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
> được bật và một Job khớp với
> [các tiêu chí gang scheduling](#integrate-with-workload-apis),
> các cập nhật vào `.spec.parallelism` sẽ bị từ chối vì trường `minCount` của `Workload`
> là bất biến (immutable). Để scale một Job dùng gang scheduling, hãy xóa và tạo lại nó
> với giá trị parallelism mới.

### Trì hoãn việc tạo Pod thay thế (Delayed creation of replacement pods) {#pod-replacement-policy}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [stable]`

Mặc định, Job controller tạo lại các Pod ngay khi chúng thất bại hoặc đang kết thúc (có
deletion timestamp). Điều này có nghĩa là, tại một thời điểm nhất định, khi một số Pod
đang kết thúc, số Pod đang chạy của một Job có thể lớn hơn `parallelism` hoặc lớn hơn
một Pod cho mỗi index (nếu bạn dùng Indexed Job).

Bạn có thể chọn chỉ tạo các Pod thay thế khi Pod đang kết thúc đã hoàn toàn ở trạng
thái cuối (có `status.phase: Failed`). Để làm vậy, đặt
`.spec.podReplacementPolicy: Failed`.
Chính sách thay thế mặc định phụ thuộc vào việc Job có đặt `podFailurePolicy` hay
không. Khi không có chính sách lỗi Pod nào được định nghĩa cho một Job, việc bỏ qua
trường `podReplacementPolicy` sẽ chọn chính sách thay thế `TerminatingOrFailed`:
control plane tạo các Pod thay thế ngay khi Pod bị xóa (ngay khi control plane thấy một
Pod của Job này có `deletionTimestamp` được đặt).
Với các Job có đặt chính sách lỗi Pod, `podReplacementPolicy` mặc định là `Failed`, và
không giá trị nào khác được phép.
Xem [Chính sách lỗi Pod](#pod-failure-policy) để tìm hiểu thêm về chính sách lỗi Pod
cho Job.

```yaml
kind: Job
metadata:
  name: new
  ...
spec:
  podReplacementPolicy: Failed
  ...
```

Miễn là cluster của bạn đã bật feature gate này, bạn có thể kiểm tra trường
`.status.terminating` của một Job. Giá trị của trường này là số Pod thuộc sở hữu của
Job hiện đang kết thúc.

```shell
kubectl get jobs/myjob -o yaml
```

```yaml
apiVersion: batch/v1
kind: Job
# .metadata và .spec được lược bỏ
status:
  terminating: 3 # ba Pod đang kết thúc và chưa đạt tới phase Failed
```

### Ủy quyền quản lý đối tượng Job cho controller bên ngoài (Delegation of managing a Job object to external controller)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

Tính năng này cho phép bạn vô hiệu hóa Job controller tích hợp sẵn (built-in) cho một
Job cụ thể, và ủy quyền việc điều phối (reconcile) Job đó cho một controller bên ngoài.

Bạn chỉ định controller sẽ điều phối Job bằng cách đặt một giá trị tùy chỉnh cho trường
`spec.managedBy` — bất kỳ giá trị nào khác `kubernetes.io/job-controller`. Giá trị của
trường này là bất biến (immutable).

> **Ghi chú:**
> Khi dùng tính năng này, hãy chắc chắn rằng controller được chỉ định bởi trường này đã
> được cài đặt, nếu không Job có thể hoàn toàn không được điều phối.

> **Ghi chú:**
> Khi phát triển một Job controller bên ngoài, hãy lưu ý rằng controller của bạn cần
> hoạt động theo cách tuân thủ các định nghĩa về spec và các trường status của API đối
> tượng Job.
>
> Vui lòng xem xét chi tiết những điều này trong
> [Job API](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/job-v1/).
> Chúng tôi cũng khuyến nghị bạn chạy các bài kiểm tra tuân thủ e2e (e2e conformance
> test) cho đối tượng Job để xác minh phần triển khai của mình.
>
> Cuối cùng, khi phát triển một Job controller bên ngoài, hãy đảm bảo nó không dùng
> finalizer `batch.kubernetes.io/job-tracking`, vốn được dành riêng cho controller tích
> hợp sẵn.

## Các lựa chọn thay thế (Alternatives)

### Pod trần (Bare Pods)

Khi node mà một Pod đang chạy trên đó khởi động lại hoặc gặp sự cố, pod đó bị chấm dứt
và sẽ không được khởi động lại. Tuy nhiên, một Job sẽ tạo các Pod mới để thay thế các
Pod đã bị chấm dứt. Vì lý do này, chúng tôi khuyến nghị bạn dùng Job thay vì một Pod
trần (bare Pod), ngay cả khi ứng dụng của bạn chỉ cần một Pod duy nhất.

### Replication Controller

Job bổ trợ cho [Replication Controller](./70-replicationcontroller-vi.md).
Một Replication Controller quản lý các Pod không được kỳ vọng sẽ kết thúc (ví dụ các
web server), còn Job quản lý các Pod được kỳ vọng sẽ kết thúc (ví dụ các tác vụ batch).

Như đã thảo luận trong [Vòng đời Pod](./47-pod-lifecycle-vi.md), `Job` *chỉ* phù hợp
cho các pod có `RestartPolicy` bằng `OnFailure` hoặc `Never`.

> **Ghi chú:**
> Nếu `RestartPolicy` không được đặt, giá trị mặc định là `Always`.

### Một Job đơn lẻ khởi động Pod đóng vai trò controller (Single Job starts controller Pod)

Một mẫu khác là một Job đơn lẻ tạo ra một Pod, Pod này sau đó tạo ra các Pod khác —
đóng vai trò như một dạng controller tùy chỉnh cho các Pod đó. Cách này cho phép sự
linh hoạt cao nhất, nhưng có thể hơi phức tạp để bắt đầu và ít tích hợp với Kubernetes
hơn.

Một ưu điểm của cách tiếp cận này là toàn bộ quy trình có được sự đảm bảo hoàn thành
của một đối tượng Job, nhưng vẫn duy trì toàn quyền kiểm soát việc những Pod nào được
tạo ra và cách phân công công việc cho chúng.

## Tiếp theo (What's next)

* Tìm hiểu về [Pod](./46-pods-vi.md).
* Đọc về các cách chạy Job khác nhau:
  * [Xử lý song song thô bằng một hàng đợi công việc (Coarse Parallel Processing Using a Work Queue)](https://kubernetes.io/docs/tasks/job/coarse-parallel-processing-work-queue/)
  * [Xử lý song song mịn bằng một hàng đợi công việc (Fine Parallel Processing Using a Work Queue)](https://kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/)
  * Dùng một [Indexed Job cho xử lý song song với phân công việc tĩnh](https://kubernetes.io/docs/tasks/job/indexed-parallel-processing-static/)
  * Tạo nhiều Job dựa trên một template: [Xử lý song song bằng cách mở rộng (Parallel Processing using Expansions)](https://kubernetes.io/docs/tasks/job/parallel-processing-expansion/)
* Theo các link trong phần [Tự động dọn dẹp các Job đã hoàn tất](#clean-up-finished-jobs-automatically)
  để tìm hiểu thêm về cách cluster của bạn có thể dọn dẹp các tác vụ đã hoàn thành
  và/hoặc thất bại.
* `Job` là một phần của Kubernetes REST API.
  Đọc định nghĩa đối tượng
  [Job](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/job-v1/)
  để hiểu API cho job.
* Đọc về [`CronJob`](./69-cron-jobs-vi.md), thứ mà bạn có thể dùng để định nghĩa một
  chuỗi các Job sẽ chạy theo lịch, tương tự công cụ `cron` của UNIX.
* Thực hành cách cấu hình việc xử lý các lần pod thất bại có thể thử lại và không thể
  thử lại bằng `podFailurePolicy`, dựa trên các
  [ví dụ](https://kubernetes.io/docs/tasks/job/pod-failure-policy/) từng bước.
* Tìm hiểu về [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)
  cho việc lập lịch tất-cả-hoặc-không (all-or-nothing) đối với các Job song song.

[Indexed Job với phân công việc tĩnh]: https://kubernetes.io/docs/tasks/job/indexed-parallel-processing-static/
[Mở rộng từ Job template]: https://kubernetes.io/docs/tasks/job/parallel-processing-expansion/
[Job với giao tiếp Pod-to-Pod]: https://kubernetes.io/docs/tasks/job/job-with-pod-to-pod-communication/
[Hàng đợi với một Pod cho mỗi work item]: https://kubernetes.io/docs/tasks/job/coarse-parallel-processing-work-queue/
[Hàng đợi với số lượng Pod thay đổi]: https://kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/

# Lab 11b — HPA và VPA

> **Điểm bắt đầu:** snapshot `04-metrics-ready` — mốc do [Lab 11a](LAB-11A-OBSERVABILITY.md) tạo,
> xem [chuỗi snapshot](README.md#3-chuỗi-snapshot). Mốc này khác mọi mốc trước ở đúng một thứ:
> **metrics-server đang chạy** và API `metrics.k8s.io` đã `Available`.
> **Điểm kết thúc:** cleanup trả cluster về đúng `04-metrics-ready`, **không tạo snapshot mới**.
> Lab không cài thêm bất cứ thành phần nào và không sửa gì trong `kube-system`.
> **Lab trước:** [Lab 11a — Observability](LAB-11A-OBSERVABILITY.md) đã cài metrics-server và chốt
> gate `kubectl top` ở [B12.3](LAB-11A-OBSERVABILITY.md#b123-gate-trạng-thái-của-mốc-mới).
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Đây là một **lab trả nợ**, không phải một nhóm bài mới. Nó trả
[nợ #1 của sổ nợ lab](README.md#5-sổ-nợ-lab) — *thực hành HPA và VPA* — phát sinh ở
[giai đoạn 4](../00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller) tại bài
[72](../72-horizontal-pod-autoscale-vi.md) và [73](../73-vertical-pod-autoscale-vi.md), bị hoãn ở
[Lab 4b](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) vì HPA cần metrics-server của giai đoạn 11. Lab
nằm ở mục [Giai đoạn 11 — Observability](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability), và
bài thực hành chính của nó là [342 — Hướng dẫn từng bước về
HorizontalPodAutoscaler](../342-hpa-walkthrough-vi.md). Bài [71](../71-autoscaling-vi.md) là bản đồ
hai trục co giãn; lab đi cả hai trục nhưng **chỉ một trục làm được** — lý do nằm ở
[B2](#b2-hai-trục-co-giãn-và-năng-lực-thật-của-cluster-này).

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không chép
lại con số phiên bản nào** và **không kéo image mới nào** — toàn bộ workload dùng `busybox` đã khóa
ở bảng đó và đã có sẵn trên ba node. Thành phần ngoài baseline duy nhất lab dựa vào là
metrics-server, version của nó nằm ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00), và lab **chỉ đọc**, không
đụng vào.

Lab dùng Deployment, Service, Job, `kubectl scale`, `kubectl apply` và `requests`/`limits` — tất cả
thuộc giai đoạn 3 đến 5 đã học. **Không** tạo CRD, **không** dùng DRA, **không** cài Operator: đó là
nội dung của [giai đoạn 13](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao) và
[giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), đứng sau lab này.

**Trước khi bắt đầu**, chạy [quy trình mở đầu
A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab) trên `lab-k8s-master`, rồi thêm ba
lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Ba lệnh riêng của lab 11b: moc dau vao phai co san nguon metric, va chua co HPA nao.
kubectl -n kube-system get deployment metrics-server
kubectl top node
kubectl get hpa -A
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod; **`metrics-server` có `READY 1/1`**,
**`kubectl top node` in ra ba dòng có số thật**, và `kubectl get hpa -A` trả `No resources found`.
Nếu `kubectl top node` báo lỗi thì cluster của bạn **không** ở mốc `04-metrics-ready`: restore cả ba
VM về đúng mốc đó, hoặc quay lại [Lab 11a](LAB-11A-OBSERVABILITY.md) — đừng chạy tiếp lab này.

**Lab trả nợ thì phải đọc lại bài gốc trước.** Mở lại hai bài dưới đây và đọc hết phần *Phải hiểu ở
lần đọc này* của chúng, rồi mới chạy phần B:

- [72 — Tự động co giãn Pod theo chiều ngang](../72-horizontal-pod-autoscale-vi.md): công thức
  `desiredReplicas`, dung sai, `Utilization` tính theo phần trăm của `requests`, cửa sổ ổn định.
- [73 — Tự động co giãn Pod theo chiều dọc](../73-vertical-pod-autoscale-vi.md): VPA là CRD phải cài
  riêng, ba thành phần, các `updateMode`.

```bash
# Doi bien nay thanh 'roi' SAU KHI da doc lai xong hai bai 72 va 73.
DA_DOC_LAI_72_73='chua'

case "$DA_DOC_LAI_72_73" in
  roi) echo 'PASS: da doc lai bai 72 va 73 truoc khi mo lab tra no' ;;
  *)   echo 'FAIL: mo lai bai 72 va 73, doc xong roi dat bien nay = roi va chay lai' ;;
esac
```

**PASS:** dòng `PASS: da doc lai bai 72 va 73…` xuất hiện. Đây là gate tự khai báo, và nó là quy
định của [sổ nợ lab](README.md#5-sổ-nợ-lab): lab trả nợ phải nhắc lại bài gốc. Bỏ qua nó thì phần B
vẫn chạy, nhưng bạn sẽ nhìn số mà không biết mình đang nhìn cái gì.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- **Hai trục co giãn của bài [71](../71-autoscaling-vi.md) không đối xứng nhau trên cluster thật:**
  chỉ ra bằng lệnh rằng trục ngang tự động (`HorizontalPodAutoscaler`) là **API lõi có sẵn**, còn
  trục dọc tự động (`VerticalPodAutoscaler`) **không tồn tại** trên cluster baseline, và nói đúng
  hai thứ còn thiếu của trục dọc.
- **`Utilization` là một phép chia, và mẫu số là `requests`:** dựng được một workload có
  `requests.cpu` đọc từ dung lượng thật của node, rồi chứng minh bằng chính `status` của HPA rằng
  `averageUtilization` khớp với `averageValue` chia cho `requests.cpu`.
- **Không có `requests` thì không có phép chia:** tạo được một HPA hiển thị `<unknown>`, chỉ ra
  condition `ScalingActive` là `False`, và chứng minh rằng thứ thiếu là **mẫu số** chứ không phải
  số đo — metrics-server vẫn báo mức tiêu thụ của chính Pod đó.
- **Một HPA, hai cách tạo:** tạo bằng `kubectl autoscale` và bằng manifest `autoscaling/v2`, rồi
  chứng minh bằng `diff` rằng hai cách cho ra cùng một `spec`.
- **HPA ghi vào subresource `scale`:** đọc được subresource đó trên Deployment, và chứng minh
  DaemonSet **không có** subresource đó — đó là lý do cơ học khiến HPA không áp dụng cho nó.
- **Đọc ba condition** `AbleToScale`, `ScalingActive`, `ScalingLimited`: nói đúng mỗi condition trả
  lời câu hỏi gì, và tạo được một tình huống làm `ScalingLimited` chuyển sang `True`.
- **Mở rộng nhanh, thu hẹp chậm:** sinh tải thật có trần, đo **bằng số vòng quan sát** rằng HPA tăng
  replica sớm hơn hẳn lúc nó giảm, và chỉ ra cơ chế tạo ra sự bất đối xứng đó nằm ở đâu — trong
  object, hay trong controller.
- **`behavior` là chỗ sự bất đối xứng thành cấu hình:** chứng minh object HPA mặc định **không mang**
  trường `behavior`, đặt `scaleDown.stabilizationWindowSeconds` rồi đo lại và chứng minh chu kỳ thu
  hẹp thứ hai ngắn hơn chu kỳ thứ nhất.
- **HPA xung đột với bàn tay người:** chứng minh `kubectl scale` bị HPA kéo ngược lại, chứng minh
  `spec.replicas` còn trong manifest làm `kubectl apply` giật số Pod về, và làm được cách xử lý mà
  bài [72](../72-horizontal-pod-autoscale-vi.md) khuyến nghị.
- **Trục dọc dừng ở đâu và vì sao:** chứng minh bằng lệnh rằng CRD của VPA không có trên cluster,
  giải thích vì sao **không** cài nó trong lab này, và nói đúng phần nào của [nợ
  #1](README.md#5-sổ-nợ-lab) đã trả xong, phần nào còn treo cùng điều kiện để trả.
- **Cleanup đúng phạm vi:** xóa sạch object của bài học nhưng **giữ nguyên metrics-server**, và
  chứng minh bằng checksum rằng lab không đụng vào cấu hình node lẫn Deployment của metrics-server.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài | Kiểm chứng ở |
| --- | --- |
| [71 — Tự động co giãn Workload](../71-autoscaling-vi.md) | B2 — hai trục và **năng lực thật** của cluster: B2.1 chứng minh trục ngang tự động có sẵn trong API lõi, B2.2 chứng minh trục dọc tự động không có. B9 quay lại trục dọc. Phần *co giãn ngang thủ công* đã làm ở [Lab 4b B2](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md#b2-scale-statefulset--hai-chiều-hai-thứ-tự) và [Lab 4a B3](LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md#b3-scale-không-phải-là-rollout); phần *co giãn dọc thủ công* đã làm ở [Lab 3c B6](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md#b6-resize-tài-nguyên-của-container-đang-chạy) — lab này không lặp lại, chỉ nối chúng vào bản đồ ở B9.2 |
| [72 — Tự động co giãn Pod theo chiều ngang](../72-horizontal-pod-autoscale-vi.md) | B4 — đối tượng API, `kubectl autoscale` so với manifest `autoscaling/v2`, subresource `scale`, và ranh giới "không áp dụng cho đối tượng không co giãn được"; B5 — hệ quả của việc container không đặt resource request; B6 — công thức `desiredReplicas` đo trên số thật, dung sai, và cửa sổ ổn định bất đối xứng; B7 — trường `behavior` và `scaleDown.stabilizationWindowSeconds`; B8 — thrashing khi `spec.replicas` còn trong manifest, cùng quy trình chuyển Deployment sang tự động co giãn |
| [73 — Tự động co giãn Pod theo chiều dọc](../73-vertical-pod-autoscale-vi.md) | B2.2 — chứng minh VPA là CRD **chưa được cài**, API server từ chối `kind: VerticalPodAutoscaler`; B9 — vì sao lab không cài nó, mẫu số dùng chung khiến VPA và HPA cùng theo CPU trên một workload là xung đột, và phần `updateMode` (đọc hiểu, ghi rõ không có gate) |
| [342 — Hướng dẫn từng bước về HorizontalPodAutoscaler](../342-hpa-walkthrough-vi.md) | B3 — dựng workload đích và Service theo đúng hình dạng của bài, thay image ví dụ bằng image baseline; B4 — *Tạo HorizontalPodAutoscaler* và *Tạo autoscaler theo cách khai báo*; B6 — *Tăng tải* và *Dừng tạo tải* bằng một Job có trần; B4.2 và B8 — phụ lục *Các status condition*; B6.2 — mục *Đại lượng*, đọc `averageValue` dạng mili và đối chiếu với `averageUtilization` |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| **Cài add-on VPA và chạy một `VerticalPodAutoscaler` thật** — phần thực hành của bài [73](../73-vertical-pod-autoscale-vi.md) | Ba lý do cộng lại, không phải một. **Một:** VPA không có trong [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00), tức chưa có version nào được chốt cho baseline. **Hai:** cài nó là **đổi hạ tầng vĩnh viễn** — thêm ba Deployment, một mutating webhook và một bộ CRD vào `kube-system` — trong khi lab này được khai báo là *trả về* `04-metrics-ready` chứ không tạo mốc mới; một add-on cài rồi mà không chụp mốc sẽ biến mất hoặc lệch state ở lab sau. **Ba:** VPA là CRD cộng controller, tức đúng mô hình Operator của [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) — cài ở đây là nhảy cóc. Hệ quả: **phần VPA của [nợ #1](README.md#5-sổ-nợ-lab) chưa được trả**; điều kiện để trả nằm ở B9.3 |
| Ba thành phần *Recommender* / *Updater* / *admission controller*, `resourcePolicy`, `minAllowed`/`maxAllowed`/`controlledValues` | Chúng chỉ quan sát được khi add-on VPA đang chạy. Không có object nào trên cluster để `kubectl get`, nên không viết được gate `PASS:` — theo đúng quy ước của thư mục lab thì không đưa vào. Phần **đọc hiểu** về chúng nằm ở B9.2 và được kiểm ở [checkpoint](#3-checkpoint-11b) |
| Chế độ `InPlace` của VPA và feature gate `InPlacePodVerticalScaling` trên hai thành phần VPA | Bài [73](../73-vertical-pod-autoscale-vi.md) tự xếp mục này vào *Đọc lướt*: alpha, và phải bật feature gate ở cả cluster lẫn add-on. Bật feature gate là việc của [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| *Co giãn dựa trên metric tùy chỉnh*, *metric bên ngoài*, *co giãn dựa trên nhiều metric* của bài [72](../72-horizontal-pod-autoscale-vi.md) và [342](../342-hpa-walkthrough-vi.md) | Cần đăng ký `custom.metrics.k8s.io` hoặc `external.metrics.k8s.io`, mà hai API đó do **adapter của bên thứ ba** cung cấp chứ không phải metrics-server. [Lab 11a](LAB-11A-OBSERVABILITY.md#11-ánh-xạ-tài-liệu-sang-bài-thực-hành) đã ghi lý do không dựng stack giám sát trên ba VM lab; phần này thuộc [giai đoạn 23](../00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) |
| *Metric tài nguyên theo container* (`type: ContainerResource`) | Chỉ có nghĩa khi Pod có nhiều container với vai trò khác nhau. Workload của lab là một container; dựng thêm sidecar chỉ để đổi một trường metric là kéo dài lab mà không thêm cơ chế mới nào |
| Trường `tolerance` trong `behavior` | Bài [72](../72-horizontal-pod-autoscale-vi.md) ghi trường này ở mức **beta** trong baseline. Bật và kiểm một trường beta là thao tác feature gate ở phạm vi cluster — [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). Dung sai **mặc định** vẫn quan sát được gián tiếp: ở B6 bạn thấy HPA không nhúc nhích với các dao động nhỏ quanh mục tiêu |
| Bốn tùy chọn dòng lệnh `--horizontal-pod-autoscaler-*` của kube-controller-manager | Đặt chúng nghĩa là sửa `/etc/kubernetes/manifests/kube-controller-manager.yaml` rồi chờ kubelet dựng lại static Pod — [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). Lab này **chỉ đọc** manifest đó ở B6.5 để biết cluster đang dùng mặc định hay giá trị tùy chỉnh, và B10.3 chứng minh bằng checksum rằng file không bị sửa |
| Image `registry.k8s.io/hpa-example` mà bài [342](../342-hpa-walkthrough-vi.md) dùng | Không nằm trong [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và kéo nó về là thêm một image ngoài baseline. B3 thay bằng `httpd` của `busybox` đã khóa; B6 thay vòng lặp `kubectl run` bằng một **Job có trần và có deadline**, an toàn hơn hẳn cho ba VM 4/2/2 vCPU |
| `minikube addons enable metrics-server` ở phần *Trước khi bạn bắt đầu* của bài [342](../342-hpa-walkthrough-vi.md) | Chuỗi lab không dùng minikube. metrics-server đã được cài bằng tay ở [Lab 11a B5](LAB-11A-OBSERVABILITY.md#b5-cài-metrics-server-và-chữa-lỗi-certificate), kèm phần chẩn đoán lỗi certificate mà `addons enable` giấu đi |
| *Cluster Proportional Autoscaler*, *KEDA*, *co giãn theo lịch* của bài [71](../71-autoscaling-vi.md) | Đều là dự án ngoài lõi Kubernetes; chính bài xếp chúng vào *Đọc lướt* và lộ trình không cài dự án nào trong số đó |
| *Co giãn hạ tầng cluster* / node autoscaling của bài [71](../71-autoscaling-vi.md) | Thêm bớt node là một tầng khác hẳn, thuộc [giai đoạn 12](../00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao) và [giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node). Ba VM của lab là cố định |

### 1.2. Thời lượng

2–3 giờ, tính từ lúc gate mở đầu đã PASS. Ba khoảng thời gian đáng kể đều là **chờ có kiểm soát**:
chờ metrics-server hoàn thành chu kỳ thu thập đầu tiên cho workload mới (B3.3), chờ HPA thu hẹp sau
khi tải dừng (B6.4), và chu kỳ tải thứ hai của B7.3. Mọi bước phải chờ đều viết dưới dạng vòng lặp
có điều kiện thoát và **đếm số vòng**, không phải một con số cố định — chu kỳ đồng bộ của HPA và cửa
sổ ổn định đều **phụ thuộc cấu hình** của kube-controller-manager, và B6.5 kiểm chính điều đó.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi rõ
  node khác**. Lệnh cần `sudo` để đọc file trên node chạy trên chính node đó.
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Gần như mọi gate so sánh với biến đặt ở B0
  (`REQ_CPU_M`, `LIM_CPU_M`, `MAX_REP`, `TARGET_PCT`, `POLL`, `WK`, `EV`) và hai hàm chuẩn hóa
  (`cpu_m`, `mem_mi`); mở shell mới giữa chừng là mất hết.
- **Trần tài nguyên của lab được sinh từ `allocatable` thật, không viết cứng.** Ba VM lab là 4/2/2
  vCPU theo [bảng A1.2](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm), nên B0.3 tính mọi ngưỡng từ dung
  lượng thật của hai worker và gate rằng **tổng `limits` của toàn bộ replica cộng toàn bộ Pod sinh
  tải không vượt một nửa** tổng `allocatable` của hai worker đó. Không tự sửa các ngưỡng này lên cho
  "thấy rõ hơn".
- **Cách dừng tải khẩn cấp.** Tải trong lab do một Job sinh ra và Job đó có `activeDeadlineSeconds`,
  nên nó tự chết kể cả khi bạn quên. Nếu cluster ì đi hoặc bạn muốn dừng ngay, chạy theo đúng thứ tự
  dưới đây trên `lab-k8s-master`:

  ```bash
  # 1. Cat nguon tai truoc — day la thu dang sinh CPU.
  kubectl -n lab-11b delete job --all --ignore-not-found --wait=false

  # 2. Go HPA de no thoi day so replica len.
  kubectl -n lab-11b delete hpa --all --ignore-not-found

  # 3. Ha workload dich ve mot replica.
  kubectl -n lab-11b scale deployment --all --replicas=1

  # 4. Neu van chua yen: xoa ca namespace cua lab. Khong dong toi kube-system.
  kubectl delete namespace lab-11b --wait=false
  ```

  Bốn lệnh này chỉ đụng vào namespace `lab-11b`. **Không** bao giờ `kubectl delete` trong
  `kube-system` để "giải phóng tài nguyên" — metrics-server nằm ở đó và nó là định nghĩa của mốc.
- **Lab không cài gì và không tạo snapshot mới.** Nó không kéo image mới: `busybox` đã có trên cả ba
  node từ Lab 00, và metrics-server đã có từ Lab 11a. Nếu bạn thấy mình sắp gõ `curl` một manifest
  từ mạng thì bạn đã đi lạc.
- **Lab chỉ ĐỌC cấu hình node và cấu hình control plane.** Không sửa `/var/lib/kubelet/config.yaml`,
  không sửa file nào trong `/etc/kubernetes/manifests`, không restart kubelet, không đụng vào
  Deployment `metrics-server`. B0.4 ghi checksum của những thứ đó và B10.3 đối chiếu lại.
- **Bước cố ý cấu hình sai duy nhất là B5** — một Deployment không khai `resources` để HPA báo
  `<unknown>`. Theo quy ước của thư mục lab, nó được ghim vào `lab-k8s-worker2` bằng `nodeName`.
  Lab này không có fault injection ở tầng node.
- Mọi object của lab nằm trong namespace `lab-11b` và luôn được gọi kèm `-n lab-11b`. Lab **không**
  tạo object phạm vi cluster nào: không ClusterRole, không CRD, không PV, không StorageClass.
- **Mọi con số trong lab đọc từ cluster thật** — `allocatable` của hai worker, số replica, mức tiêu
  thụ CPU, giá trị `requests`. Gate so **giá trị đã chuẩn hóa**, không so chuỗi: `1` và `1000m` là
  cùng một thứ, và hai hàm ở B0.2 lo việc đó.
- **Không nhớ con số thời gian như một cam kết.** Chu kỳ đồng bộ của HPA và cửa sổ ổn định là cấu
  hình của kube-controller-manager, không phải hằng số của Kubernetes. Lab đo **số vòng quan sát**
  trên cluster của bạn và so hai con số đó với nhau, chứ không so với một con số in sẵn.
- Manifest tạm ghi vào `~/lab-work/11b/`; bằng chứng ghi vào `~/lab-evidence/11b/`.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Sự cố phát sinh trong bài học nằm
  ở [mục 4](#4-troubleshooting-của-lab-này); sự cố dựng môi trường nằm ở [mục 4 của Lab
  00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).

---

# Phần B — Thực hành kiến thức 11b

## B0. Chuẩn bị workspace, ngưỡng sinh từ cluster và ảnh chụp "trước"

**Mục đích:** dựng chỗ làm việc, khóa tên node vào biến, **tính mọi ngưỡng của lab từ
`.status.allocatable` thật** thay vì bịa số, và chụp checksum của những thứ lab hứa không đụng vào
để B10.3 biến lời hứa đó thành thứ kiểm chứng được.

### B0.1. Workspace, namespace và biến tên node

```bash
mkdir -p ~/lab-work/11b ~/lab-evidence/11b
WK=~/lab-work/11b
EV=~/lab-evidence/11b

kubectl config current-context
kubectl create namespace lab-11b

MASTER='lab-k8s-master'
W1='lab-k8s-worker1'
W2='lab-k8s-worker2'
NODES="$MASTER $W1 $W2"

for n in $NODES; do
  kubectl get node "$n" -o jsonpath='{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}'
done | tee "$EV/b0-nodes.txt"

test "$(wc -l < "$EV/b0-nodes.txt")" -eq 3 \
  && test "$(kubectl get namespace lab-11b -o jsonpath='{.status.phase}')" = 'Active' \
  && echo 'PASS: ba node doc duoc va namespace lab-11b da Active'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B0.2. Hai hàm chuẩn hóa

Kubernetes viết cùng một giá trị theo nhiều dạng (`1` và `1000m`, `1Gi` và `1024Mi`), và `status`
của HPA cũng vậy — bài [342](../342-hpa-walkthrough-vi.md) có hẳn một mục *Đại lượng* nói rằng bạn
sẽ thấy giá trị dao động giữa `1` và `1500m`. Mọi so sánh trong lab đi qua hai hàm dưới đây. Chúng
giống hệt [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) và
[Lab 11a](LAB-11A-OBSERVABILITY.md) — giữ nguyên để ba lab đọc số theo cùng một cách. Định nghĩa
chúng **một lần** và giữ tới hết phần B:

```bash
cpu_m()  { case "$1" in ''|'<none>') echo 0 ;; *m) echo "${1%m}" ;; *) echo "$(( $1 * 1000 ))" ;; esac; }
mem_mi() { case "$1" in ''|'<none>') echo 0 ;; *Ki) echo "$(( ${1%Ki} / 1024 ))" ;; *Mi) echo "${1%Mi}" ;; *Gi) echo "$(( ${1%Gi} * 1024 ))" ;; *) echo 0 ;; esac; }

test "$(cpu_m 500m)" -eq 500 && test "$(cpu_m 2)" -eq 2000 \
  && test "$(cpu_m 0)" -eq 0 \
  && test "$(mem_mi 1Gi)" -eq 1024 && test "$(mem_mi 2048Ki)" -eq 2 \
  && echo 'PASS: hai ham chuan hoa hoat dong dung'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B0.3. Sinh bộ ngưỡng của lab từ `allocatable` thật

Chỉ đọc hai worker: `lab-k8s-master` mang taint `NoSchedule` nên không nhận Pod thường, và một trần
tính cả phần không dùng được là một trần sai.

```bash
MIN_W_CPU_M=0; MIN_W_MEM_MI=0; SUM_W_CPU_M=0; SUM_W_MEM_MI=0
for n in "$W1" "$W2"; do
  V_CPU="$(cpu_m  "$(kubectl get node "$n" -o jsonpath='{.status.allocatable.cpu}')")"
  V_MEM="$(mem_mi "$(kubectl get node "$n" -o jsonpath='{.status.allocatable.memory}')")"
  echo "$n allocatable: ${V_CPU}m CPU / ${V_MEM}Mi memory"
  SUM_W_CPU_M=$(( SUM_W_CPU_M + V_CPU ));  SUM_W_MEM_MI=$(( SUM_W_MEM_MI + V_MEM ))
  if [ "$MIN_W_CPU_M" -eq 0 ] || [ "$V_CPU" -lt "$MIN_W_CPU_M" ]; then MIN_W_CPU_M="$V_CPU"; fi
  if [ "$MIN_W_MEM_MI" -eq 0 ] || [ "$V_MEM" -lt "$MIN_W_MEM_MI" ]; then MIN_W_MEM_MI="$V_MEM"; fi
done
echo "worker nho nhat: ${MIN_W_CPU_M}m / ${MIN_W_MEM_MI}Mi"
echo "tong hai worker: ${SUM_W_CPU_M}m / ${SUM_W_MEM_MI}Mi"

test "$MIN_W_CPU_M" -gt 0 && test "$MIN_W_MEM_MI" -gt 0 \
  && echo 'PASS: doc duoc allocatable that cua ca hai worker'
```

Từ hai con số đó sinh ra toàn bộ tham số của lab:

```bash
TARGET_PCT=50
POLL=15

REQ_CPU_M=$((  MIN_W_CPU_M / 100 ))
LIM_CPU_M=$((  REQ_CPU_M * 5 ))
REQ_MEM_MI=$(( MIN_W_MEM_MI / 200 ))
LIM_MEM_MI=$(( REQ_MEM_MI * 4 ))

MAX_REP=$(( SUM_W_CPU_M / 4 / LIM_CPU_M ))
if [ "$MAX_REP" -gt 10 ]; then MAX_REP=10; fi

LOAD_N=2
LOAD_LIM_CPU_M=$(( MIN_W_CPU_M / 8 ))
LOAD_SEC=240
LOAD_SEC_2=120
LOAD_DEADLINE=$(( LOAD_SEC + 120 ))

{
  echo "=== $(date -Is) — tham so lab 11b sinh tu allocatable that ==="
  echo "workload dich : requests cpu=${REQ_CPU_M}m memory=${REQ_MEM_MI}Mi"
  echo "workload dich : limits   cpu=${LIM_CPU_M}m memory=${LIM_MEM_MI}Mi"
  echo "HPA           : target=${TARGET_PCT}% min=1 max=${MAX_REP}"
  echo "nguong tuyet doi cua HPA = ${TARGET_PCT}% x ${REQ_CPU_M}m = $(( REQ_CPU_M * TARGET_PCT / 100 ))m moi Pod"
  echo "sinh tai      : ${LOAD_N} Pod, limit cpu=${LOAD_LIM_CPU_M}m, chay ${LOAD_SEC}s, deadline ${LOAD_DEADLINE}s"
} | tee "$EV/b0-tham-so.txt"
```

Gate **giá trị**, không gate chuỗi — bộ tham số phải nằm trong trần an toàn của ba VM lab:

```bash
test "$REQ_CPU_M" -ge 10 \
  && echo "PASS: requests.cpu = ${REQ_CPU_M}m, du lon de metrics-server phan giai duoc"
test "$LIM_CPU_M" -gt "$REQ_CPU_M" && test "$LIM_MEM_MI" -gt "$REQ_MEM_MI" \
  && echo 'PASS: limits lon hon requests o ca hai truc'
test "$REQ_MEM_MI" -ge 16 \
  && echo "PASS: requests.memory = ${REQ_MEM_MI}Mi, du cho busybox httpd"
test "$MAX_REP" -ge 3 \
  && echo "PASS: maxReplicas = ${MAX_REP}, du cho vung de HPA co gian va cham tran"

TRAN_M=$(( MAX_REP * LIM_CPU_M + LOAD_N * LOAD_LIM_CPU_M ))
echo "tran CPU cua toan lab = ${TRAN_M}m tren tong ${SUM_W_CPU_M}m allocatable cua hai worker"
test $(( TRAN_M * 2 )) -le "$SUM_W_CPU_M" \
  && echo 'PASS: tran CPU cua toan lab khong vuot mot nua allocatable cua hai worker'
```

**Ý nghĩa:** `requests.cpu` được đặt **thấp** là có chủ đích. Bài
[72](../72-horizontal-pod-autoscale-vi.md) định nghĩa `Utilization` là tỷ lệ giữa lượng đang dùng và
lượng được `requests`, nên `requests` nhỏ làm ngưỡng tuyệt đối nhỏ theo — dòng `nguong tuyet doi`
trong evidence chính là con số mà mỗi Pod phải vượt để HPA mở rộng. Nhờ vậy một tải khiêm tốn, có
trần rõ ràng, vẫn đủ kích HPA mà không làm nghẹt ba VM lab. Trần ở gate cuối là thứ giữ cho điều đó
đúng: kể cả khi HPA chạm `maxReplicas` **và** tải chạy hết công suất, tổng `limits` vẫn dưới một nửa
dung lượng hai worker.

**PASS:** năm dòng `PASS:` của bước này xuất hiện. Nếu dòng `maxReplicas` hoặc dòng `requests.memory`
fail thì VM của bạn nhỏ hơn [bảng A1.2](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) — dừng lại và cấp
đúng phần cứng trước, đừng hạ ngưỡng để đi tiếp.

### B0.4. Ảnh chụp "trước" của những thứ lab hứa không đụng vào

```bash
{
  echo "$MASTER kubelet $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in "$W1" "$W2"; do
    echo "$n kubelet $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
  for f in kube-apiserver kube-scheduler kube-controller-manager; do
    echo "$MASTER $f $(sudo sha256sum /etc/kubernetes/manifests/$f.yaml | awk '{print $1}')"
  done
} | tee "$EV/b0-config-sha.txt"

test "$(wc -l < "$EV/b0-config-sha.txt")" -eq 6 \
  && test "$(awk '{print $3}' "$EV/b0-config-sha.txt" | grep -c '^[0-9a-f]\{64\}$')" -eq 6 \
  && echo 'PASS: ghi duoc checksum cua 3 cau hinh kubelet va 3 manifest control plane'
```

metrics-server là thứ lab này **dựa vào** chứ không sở hữu. Chụp lại đúng hai thứ định nghĩa nó —
image và danh sách `args` — để B10.3 chứng minh lab không vá thêm cờ nào vào nó:

```bash
{
  echo "image=$(kubectl -n kube-system get deploy metrics-server \
    -o jsonpath='{.spec.template.spec.containers[0].image}')"
  kubectl -n kube-system get deploy metrics-server \
    -o jsonpath='{range .spec.template.spec.containers[0].args[*]}arg={@}{"\n"}{end}'
} | tee "$EV/b0-metrics-server.txt"

{
  echo "=== $(date -Is) — trang thai truoc Lab 11b ==="
  echo '--- hpa toan cluster';  kubectl get hpa -A 2>&1
  echo '--- crd autoscaling';   kubectl get crd 2>/dev/null | grep 'autoscaling.k8s.io' || echo 'khong co'
  echo '--- namespaces';        kubectl get namespaces
  echo '--- pv';                kubectl get pv 2>&1
} | tee "$EV/b0-truoc.txt"

test -s "$EV/b0-metrics-server.txt" && test -s "$EV/b0-truoc.txt" \
  && grep -q '^image=' "$EV/b0-metrics-server.txt" \
  && echo 'PASS: da ghi anh chup truoc cua metrics-server va cua cluster'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B1. Gate điều kiện đầu vào — nguồn metric phải chạy thật

**Mục đích:** đây là **gate quan trọng nhất của lab**, và nó đứng trước mọi thứ khác vì một lý do
đơn giản: HPA theo CPU **không tồn tại như một chức năng** nếu API `metrics.k8s.io` không trả số.
Bài [72](../72-horizontal-pod-autoscale-vi.md) nói controller lấy metric tài nguyên từ API đó, và
quản trị viên phải bảo đảm API đó đã được đăng ký. Ba bước dưới đây kiểm đúng chuỗi ấy, từ dưới lên.

> Nếu bất kỳ gate nào của B1 fail thì **dừng lại**. Đừng tạo HPA, đừng sinh tải, đừng "thử xem sao"
> — mọi thứ phía sau sẽ hiển thị `<unknown>` và bạn sẽ chẩn đoán nhầm nguyên nhân. Quay lại
> [Lab 11a B5](LAB-11A-OBSERVABILITY.md#b5-cài-metrics-server-và-chữa-lỗi-certificate), sửa xong rồi
> mới mở lại lab này.

### B1.1. metrics-server và APIService

```bash
kubectl -n kube-system get deploy metrics-server -o wide
kubectl get apiservice v1beta1.metrics.k8s.io -o wide

MS_READY="$(kubectl -n kube-system get deploy metrics-server -o jsonpath='{.status.readyReplicas}')"
AV="$(kubectl get apiservice v1beta1.metrics.k8s.io \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
MR="$(kubectl api-resources --api-group=metrics.k8s.io --no-headers | wc -l)"
echo "metrics-server readyReplicas=${MS_READY:-0} | APIService Available=$AV | tai nguyen metrics.k8s.io=$MR"

test "${MS_READY:-0}" -ge 1 && test "$AV" = 'True' \
  && echo 'PASS: metrics-server dang chay va Metrics API da Available'
test "$MR" -eq 2 \
  && echo 'PASS: nhom metrics.k8s.io co dung hai tai nguyen — nodes va pods'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B1.2. `kubectl top node` trả số thật, không chỉ trả bảng

```bash
kubectl top node | tee "$EV/b1-top-node.txt"
kubectl top node --no-headers > "$EV/b1-top-node-raw.txt"

OK_T=0
while read -r n c cp m mp; do
  CM="$(cpu_m "$c")"; MM="$(mem_mi "$m")"
  echo "$n -> ${CM}m CPU / ${MM}Mi memory"
  if [ "$CM" -gt 0 ] && [ "$MM" -gt 0 ]; then OK_T=$(( OK_T + 1 )); fi
done < "$EV/b1-top-node-raw.txt"
echo "so node co so lieu hop le = $OK_T/3"

test "$(wc -l < "$EV/b1-top-node-raw.txt")" -eq 3 \
  && echo 'PASS: kubectl top node liet ke du ba node'
test "$OK_T" -eq 3 \
  && echo 'PASS: ca ba node bao muc su dung that, khac 0'
```

**Ý nghĩa:** gate này so **giá trị đã chuẩn hóa** chứ không so chuỗi. Một bảng in ra đủ ba dòng
nhưng cột CPU là `0m` nghĩa là metrics-server đang chạy mà chưa thu được gì — và HPA cũng sẽ chẳng
thu được gì.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B1.3. Cùng số liệu đó ở tầng Pod, và cửa sổ của nó

```bash
kubectl top pod -n kube-system --sort-by=cpu | head -5 | tee "$EV/b1-top-pod.txt"
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes/$W1" > "$EV/b1-raw-node.txt"

TP="$(kubectl top pod -n kube-system --no-headers | wc -l)"
WINDOW="$(grep -o '"window":"[^"]*"' "$EV/b1-raw-node.txt" | head -1 | cut -d'"' -f4)"
echo "so dong top pod trong kube-system=$TP | window=$WINDOW"

test "$TP" -gt 0 && echo 'PASS: kubectl top pod chay duoc'
test -n "$WINDOW" && echo 'PASS: moi mau metric kem theo mot cua so thoi gian'
```

**Ý nghĩa:** trường `window` nhắc lại ranh giới mà [Lab 11a
B6.3](LAB-11A-OBSERVABILITY.md#b6-kubectl-top-và-ranh-giới-của-metrics-server) đã dựng: mỗi phép đo
chỉ có nghĩa trong một cửa sổ vừa qua. HPA cũng chỉ nhìn được đúng chừng ấy — nó không biết "một giờ
trước tải thế nào", và đó là lý do sâu xa khiến nó cần một **cửa sổ ổn định** để khỏi giật cục.

**PASS:** hai dòng `PASS:` của bước này xuất hiện. **Điều kiện đầu vào của Lab 11b đã đủ.**

---

## B2. Hai trục co giãn và năng lực thật của cluster này

**Mục đích:** bài [71](../71-autoscaling-vi.md) đặt tên hai trục và nói một câu quyết định toàn bộ
hình dạng của lab này: HPA là **tài nguyên API và controller có sẵn**, còn VPA **không có sẵn** —
nó là add-on phải cài riêng. Ở đây bạn kiểm câu đó trên chính cluster của mình, trước khi làm bất cứ
việc gì, để biết ngay phần nào của [nợ #1](README.md#5-sổ-nợ-lab) trả được và phần nào không.

### B2.1. Trục ngang tự động: có sẵn trong API lõi

```bash
kubectl api-resources --api-group=autoscaling | tee "$EV/b2-api-autoscaling.txt"
kubectl api-versions | grep '^autoscaling/' | tee "$EV/b2-api-versions.txt"

HPA_N="$(kubectl api-resources --api-group=autoscaling --no-headers | grep -c 'horizontalpodautoscalers')"
V2_N="$(grep -c '^autoscaling/v2$' "$EV/b2-api-versions.txt")"
echo "tai nguyen horizontalpodautoscalers=$HPA_N | co autoscaling/v2=$V2_N"

test "$HPA_N" -ge 1 \
  && echo 'PASS: HorizontalPodAutoscaler la tai nguyen co san, khong phai add-on'
test "$V2_N" -eq 1 \
  && echo 'PASS: phien ban on dinh autoscaling/v2 duoc phuc vu san'
```

**Ý nghĩa:** không có bước cài nào ở đây, và đó chính là điều phải thấy. Cái duy nhất mà giai đoạn 4
còn thiếu để chạy HPA là **nguồn metric**, chứ không phải bản thân HPA — B1 vừa lấp nốt chỗ đó.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B2.2. Trục dọc tự động: không có mặt trên cluster này

```bash
kubectl get crd 2>/dev/null | grep 'autoscaling.k8s.io' | tee "$EV/b2-crd-vpa.txt"
CRD_VPA="$(wc -l < "$EV/b2-crd-vpa.txt")"
API_VPA="$(kubectl api-resources 2>/dev/null | grep -c 'verticalpodautoscalers' || true)"
echo "CRD ten mien autoscaling.k8s.io=$CRD_VPA | tai nguyen verticalpodautoscalers=$API_VPA"

test "$CRD_VPA" -eq 0 \
  && echo 'PASS: khong co CRD verticalpodautoscalers.autoscaling.k8s.io tren cluster'
test "$API_VPA" -eq 0 \
  && echo 'PASS: API server khong biet tai nguyen verticalpodautoscalers'
```

Thử tạo đúng object mà bài [73](../73-vertical-pod-autoscale-vi.md) đưa ra làm ví dụ — nó phải bị
từ chối:

```bash
cat > "$WK/b2-vpa.yaml" <<'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web
  namespace: lab-11b
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Off"
EOF

if kubectl apply -f "$WK/b2-vpa.yaml" > "$EV/b2-vpa-loi.txt" 2>&1; then
  echo 'FAIL: cluster nay dang co VPA — no khong con o baseline 04-metrics-ready'
else
  echo 'PASS: API server tu choi kind VerticalPodAutoscaler'
fi
cat "$EV/b2-vpa-loi.txt"

grep -qiE 'no matches for kind|unable to recognize|could not find the requested resource' "$EV/b2-vpa-loi.txt" \
  && echo 'PASS: ly do tu choi dung la khong nhan ra kind, khong phai loi quyen hay loi cu phap'
```

**Ý nghĩa:** đây là **hai kiểu "không dùng được" hoàn toàn khác nhau**, và phân biệt được chúng là
nửa bài học của cặp 72–73. Với HPA, object tạo được ngay vì kind đã có sẵn — nó chỉ nằm im nếu
thiếu nguồn metric. Với VPA, **object thậm chí không tạo được**, vì kind chưa tồn tại: bài
[73](../73-vertical-pod-autoscale-vi.md) nói VPA được định nghĩa dưới dạng CustomResourceDefinition
và "không giống HorizontalPodAutoscaler vốn là một phần của API lõi, VPA phải được cài đặt riêng".
Cài nó đòi thêm CRD, ba Deployment và một mutating webhook — lý do đầy đủ vì sao lab **không** làm
điều đó nằm ở [bảng mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành), và hệ quả với sổ nợ được ghi
ở B9.3.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

---

## B3. Workload đích: Deployment có `requests.cpu` và một Service

**Mục đích:** dựng lại đúng hình dạng của bài [342](../342-hpa-walkthrough-vi.md) — một Deployment
phục vụ HTTP, một Service đứng trước nó — bằng image baseline. Điểm mấu chốt không phải ứng dụng mà
là **`requests.cpu`**: nó là mẫu số của mọi phép tính mà HPA sắp làm.

### B3.1. Deployment và Service

```bash
cat > "$WK/b3-web.yaml" <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: lab-11b
  labels:
    app: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: httpd
        image: busybox:1.37
        command: ["sh", "-c"]
        args:
          - |
            mkdir -p /www
            echo lab-11b-web-ok > /www/index.html
            httpd -f -p 8080 -h /www
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "${REQ_CPU_M}m"
            memory: "${REQ_MEM_MI}Mi"
          limits:
            cpu: "${LIM_CPU_M}m"
            memory: "${LIM_MEM_MI}Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: lab-11b
  labels:
    app: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
EOF

kubectl apply -f "$WK/b3-web.yaml"
kubectl -n lab-11b rollout status deploy/web --timeout=180s
kubectl -n lab-11b get deploy,svc,pod -o wide
```

Gate **giá trị** của mẫu số, không gate chuỗi:

```bash
G_REQ="$(kubectl -n lab-11b get deploy web \
  -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')"
G_LIM="$(kubectl -n lab-11b get deploy web \
  -o jsonpath='{.spec.template.spec.containers[0].resources.limits.cpu}')"
G_REP="$(kubectl -n lab-11b get deploy web -o jsonpath='{.status.readyReplicas}')"
echo "requests.cpu=$G_REQ limits.cpu=$G_LIM readyReplicas=${G_REP:-0}"

test "$(cpu_m "$G_REQ")" -eq "$REQ_CPU_M" \
  && test "$(cpu_m "$G_LIM")" -eq "$LIM_CPU_M" \
  && echo 'PASS: Deployment mang dung requests va limits sinh o B0.3'
test "${G_REP:-0}" -eq 1 && echo 'PASS: mot replica dang Ready'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B3.2. Service thật sự trả lời

```bash
kubectl -n lab-11b get endpointslice -l kubernetes.io/service-name=web -o wide

kubectl run b3-client -n lab-11b --image=busybox:1.37 --restart=Never --rm -i --command -- \
  wget -q -O- http://web/ > "$EV/b3-goi-service.txt" 2>&1
cat "$EV/b3-goi-service.txt"

grep -q 'lab-11b-web-ok' "$EV/b3-goi-service.txt" \
  && echo 'PASS: Service web tra ve trang cua Deployment'
```

**Ý nghĩa:** bước này trông thừa nhưng không thừa. Toàn bộ B6 dựa vào việc các Pod sinh tải gọi được
`http://web/`; nếu DNS hoặc Service hỏng thì HPA sẽ **không bao giờ** mở rộng và bạn sẽ đi tìm lỗi ở
đúng chỗ sai. Kiểm nó bây giờ, khi còn dễ đọc.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B3.3. metrics-server đã nhìn thấy workload mới

Số liệu của một Pod mới chỉ xuất hiện sau khi metrics-server chạy xong một chu kỳ thu thập — thời
gian **phụ thuộc cấu hình**, nên viết dưới dạng vòng lặp có điều kiện thoát:

```bash
for i in $(seq 1 20); do
  kubectl top pod -n lab-11b --no-headers 2>/dev/null | grep -q '^web-' && break
  sleep "$POLL"
done
kubectl top pod -n lab-11b | tee "$EV/b3-top-pod.txt"
echo "so vong da cho = $i"

test "$(kubectl top pod -n lab-11b --no-headers | wc -l)" -ge 1 \
  && echo 'PASS: metrics-server da bao cao muc tieu thu cua Pod web'
```

**Ý nghĩa:** giờ cả **tử số** (mức tiêu thụ đo được, do metrics-server cung cấp) lẫn **mẫu số**
(`requests.cpu`, do bạn khai trong Pod spec) đều có mặt. Đó là đúng hai thứ mà HPA cần để tính
`Utilization`. B5 sẽ tháo mẫu số ra để bạn thấy chuyện gì xảy ra.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B4. HPA: một object, hai cách tạo

**Mục đích:** làm mục *Tạo HorizontalPodAutoscaler* và mục *Tạo autoscaler theo cách khai báo* của
bài [342](../342-hpa-walkthrough-vi.md) cạnh nhau, đọc ba condition ở trạng thái nghỉ, và kiểm cơ
chế mà bài [72](../72-horizontal-pod-autoscale-vi.md) mô tả: HPA ghi số replica **qua subresource
`scale`**.

### B4.1. Tạo bằng `kubectl autoscale`

```bash
kubectl autoscale deployment web -n lab-11b \
  --cpu="${TARGET_PCT}%" --min=1 --max="$MAX_REP"

kubectl -n lab-11b get hpa
kubectl -n lab-11b describe hpa web | tee "$EV/b4-describe-nghi.txt"
```

```bash
M_NAME="$(kubectl -n lab-11b get hpa web -o jsonpath='{.spec.metrics[0].resource.name}')"
M_TYPE="$(kubectl -n lab-11b get hpa web -o jsonpath='{.spec.metrics[0].resource.target.type}')"
M_VAL="$( kubectl -n lab-11b get hpa web -o jsonpath='{.spec.metrics[0].resource.target.averageUtilization}')"
H_MIN="$( kubectl -n lab-11b get hpa web -o jsonpath='{.spec.minReplicas}')"
H_MAX="$( kubectl -n lab-11b get hpa web -o jsonpath='{.spec.maxReplicas}')"
echo "metric=$M_NAME target=$M_TYPE=$M_VAL min=$H_MIN max=$H_MAX"

test "$M_NAME" = 'cpu' && test "$M_TYPE" = 'Utilization' \
  && test "$M_VAL" -eq "$TARGET_PCT" \
  && echo 'PASS: HPA theo CPU dang Utilization, dung muc tieu da chon'
test "$H_MIN" -eq 1 && test "$H_MAX" -eq "$MAX_REP" \
  && echo 'PASS: khoang replica dung bo tham so sinh o B0.3'
```

**Ý nghĩa:** `--cpu=50%` được lưu lại thành một khối `metrics` dạng `Resource`/`Utilization` chứ
không phải một trường phần trăm rời — đó là hình dạng của `autoscaling/v2` mà bài
[342](../342-hpa-walkthrough-vi.md) chỉ ra khi nó bảo bạn dump object ra YAML. Trường
`averageUtilization` **là một phần trăm của `requests`**, không phải một số mili-core.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B4.2. Ba condition ở trạng thái nghỉ

```bash
for i in $(seq 1 20); do
  test -n "$(kubectl -n lab-11b get hpa web -o jsonpath='{.status.conditions[0].type}')" && break
  sleep "$POLL"
done

hpa_cond() {
  kubectl -n lab-11b get hpa "$1" \
    -o jsonpath="{.status.conditions[?(@.type=='$2')].status}={.status.conditions[?(@.type=='$2')].reason}"
}

{
  echo "=== $(date -Is) — condition cua HPA web o trang thai nghi ==="
  echo "AbleToScale    $(hpa_cond web AbleToScale)"
  echo "ScalingActive  $(hpa_cond web ScalingActive)"
  echo "ScalingLimited $(hpa_cond web ScalingLimited)"
} | tee "$EV/b4-condition-nghi.txt"

C_ABLE="$(kubectl -n lab-11b get hpa web \
  -o jsonpath='{.status.conditions[?(@.type=="AbleToScale")].status}')"
C_ACT="$(kubectl -n lab-11b get hpa web \
  -o jsonpath='{.status.conditions[?(@.type=="ScalingActive")].status}')"

test "$C_ABLE" = 'True' \
  && echo 'PASS: AbleToScale=True — HPA doc va ghi duoc scale cua doi tuong dich'
test "$C_ACT" = 'True' \
  && echo 'PASS: ScalingActive=True — HPA tinh duoc so replica tu metric'
```

**Ý nghĩa:** phụ lục của bài [342](../342-hpa-walkthrough-vi.md) chia ba condition theo ba câu hỏi
khác nhau. `AbleToScale` hỏi *có với tới được đối tượng đích không*; `ScalingActive` hỏi *có tính
được khuyến nghị không* — nó là chỗ lỗi metric lộ ra, và B5 sẽ cho bạn thấy nó ở trạng thái `False`;
`ScalingLimited` hỏi *khuyến nghị có bị `minReplicas`/`maxReplicas` cắt cụt không*. Hai condition
đầu là gate; giá trị của `ScalingLimited` lúc này được **ghi lại** vào evidence chứ không gate, vì
nó phụ thuộc mức tiêu thụ ngay tại thời điểm đo — B8.2 sẽ tạo ra một tình huống làm nó `True` một
cách chắc chắn.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B4.3. HPA ghi vào subresource `scale` — và thứ gì không có subresource đó

```bash
kubectl -n lab-11b get deploy web --subresource=scale | tee "$EV/b4-scale-deploy.txt"
S_REP="$(kubectl -n lab-11b get deploy web --subresource=scale -o jsonpath='{.spec.replicas}')"
D_REP="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
echo "scale.spec.replicas=$S_REP | deployment.spec.replicas=$D_REP"

test -n "$S_REP" && test "$S_REP" -eq "$D_REP" \
  && echo 'PASS: Deployment co subresource scale, va no khop voi spec.replicas'
```

```bash
DS_ANY="$(kubectl -n kube-system get daemonset -o jsonpath='{.items[0].metadata.name}')"
echo "DaemonSet duoc chon de doi chung: $DS_ANY"

if kubectl -n kube-system get daemonset "$DS_ANY" --subresource=scale > "$EV/b4-scale-ds.txt" 2>&1; then
  echo 'FAIL: DaemonSet lai co subresource scale — doc lai bai 72 truoc khi di tiep'
else
  echo 'PASS: DaemonSet khong co subresource scale'
fi
cat "$EV/b4-scale-ds.txt"
```

**Ý nghĩa:** đây là **lý do cơ học** đằng sau câu "tự động co giãn Pod theo chiều ngang không áp
dụng cho các đối tượng không thể co giãn (ví dụ: một DaemonSet)" của bài
[72](../72-horizontal-pod-autoscale-vi.md). Không phải HPA từ chối DaemonSet vì một luật nào đó, mà
là **không có chỗ để ghi**: số Pod của DaemonSet do số node quyết định, nên nó không có trường
`replicas` và không phơi ra subresource `scale`. Lệnh này chỉ đọc, không tạo object nào trong
`kube-system`.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B4.4. Cùng nội dung đó ở dạng khai báo

Kỹ thuật ở đây giống [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md): chụp lại thứ cần so vào một
file, rồi `diff` thay vì đọc bằng mắt.

```bash
hpa_spec() {
  echo "scaleTargetRef=$(kubectl -n lab-11b get hpa web \
    -o jsonpath='{.spec.scaleTargetRef.apiVersion}/{.spec.scaleTargetRef.kind}/{.spec.scaleTargetRef.name}')"
  echo "minReplicas=$(kubectl -n lab-11b get hpa web -o jsonpath='{.spec.minReplicas}')"
  echo "maxReplicas=$(kubectl -n lab-11b get hpa web -o jsonpath='{.spec.maxReplicas}')"
  echo "metric=$(kubectl -n lab-11b get hpa web \
    -o jsonpath='{.spec.metrics[0].type}/{.spec.metrics[0].resource.name}')"
  echo "target=$(kubectl -n lab-11b get hpa web \
    -o jsonpath='{.spec.metrics[0].resource.target.type}={.spec.metrics[0].resource.target.averageUtilization}')"
}

hpa_spec | tee "$EV/b4-hpa-menh-lenh.txt"
kubectl -n lab-11b delete hpa web
```

```bash
cat > "$WK/b4-hpa.yaml" <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
  namespace: lab-11b
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 1
  maxReplicas: ${MAX_REP}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: ${TARGET_PCT}
EOF

kubectl apply -f "$WK/b4-hpa.yaml"
for i in $(seq 1 20); do
  test -n "$(kubectl -n lab-11b get hpa web -o jsonpath='{.status.conditions[0].type}')" && break
  sleep "$POLL"
done

hpa_spec | tee "$EV/b4-hpa-khai-bao.txt"

diff -u "$EV/b4-hpa-menh-lenh.txt" "$EV/b4-hpa-khai-bao.txt" \
  && echo 'PASS: hai cach tao cho ra cung mot spec, khong lech mot truong nao' \
  || echo 'FAIL: spec lech giua hai cach tao — doc diff o tren'

C_ACT="$(kubectl -n lab-11b get hpa web \
  -o jsonpath='{.status.conditions[?(@.type=="ScalingActive")].status}')"
test "$C_ACT" = 'True' && echo 'PASS: HPA khai bao cung dang hoat dong'
```

**Ý nghĩa:** `kubectl autoscale` là lối tắt, không phải một cơ chế khác. Cùng một object ra đời, chỉ
khác cách gõ. Từ đây trở đi lab dùng bản khai báo, vì nó là thứ bạn giữ trong quản lý mã nguồn —
và B8.3 sẽ cho thấy manifest của **Deployment** mới là chỗ có bẫy, không phải manifest của HPA.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

---

## B5. Không có `requests.cpu` thì HPA không có mẫu số

**Mục đích:** kiểm chứng câu cảnh báo trực tiếp nhất của bài
[72](../72-horizontal-pod-autoscale-vi.md): nếu một số container của Pod không đặt yêu cầu tài
nguyên tương ứng, mức sử dụng CPU của Pod đó **không xác định được** và autoscaler **không thực hiện
hành động nào** theo metric đó. Đây là lỗi cấu hình kinh điển — HPA hiện diện, `kubectl get hpa`
hiện `<unknown>`, và không có gì xảy ra cả.

### B5.1. Một Deployment không khai `resources` và một HPA trên nó

Theo quy ước của thư mục lab, bước cố ý sai này được ghim vào `lab-k8s-worker2`:

```bash
cat > "$WK/b5-khong-request.yaml" <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-khong-request
  namespace: lab-11b
  labels:
    app: web-khong-request
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-khong-request
  template:
    metadata:
      labels:
        app: web-khong-request
    spec:
      nodeName: ${W2}
      containers:
      - name: httpd
        image: busybox:1.37
        command: ["sh", "-c"]
        args:
          - |
            mkdir -p /www
            echo khong-request-ok > /www/index.html
            httpd -f -p 8080 -h /www
        ports:
        - containerPort: 8080
EOF

kubectl apply -f "$WK/b5-khong-request.yaml"
kubectl -n lab-11b rollout status deploy/web-khong-request --timeout=180s

kubectl autoscale deployment web-khong-request -n lab-11b \
  --cpu="${TARGET_PCT}%" --min=1 --max=3
```

```bash
R_NONE="$(kubectl -n lab-11b get deploy web-khong-request \
  -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')"
echo "requests.cpu cua web-khong-request = '${R_NONE:-<trong>}'"

test -z "$R_NONE" \
  && echo 'PASS: container nay that su khong khai requests.cpu'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B5.2. HPA báo `<unknown>` và `ScalingActive` là `False`

```bash
for i in $(seq 1 20); do
  test -n "$(kubectl -n lab-11b get hpa web-khong-request \
    -o jsonpath='{.status.conditions[?(@.type=="ScalingActive")].status}')" && break
  sleep "$POLL"
done

kubectl -n lab-11b get hpa | tee "$EV/b5-get-hpa.txt"
kubectl -n lab-11b describe hpa web-khong-request | tee "$EV/b5-describe.txt"

SA="$(kubectl -n lab-11b get hpa web-khong-request \
  -o jsonpath='{.status.conditions[?(@.type=="ScalingActive")].status}')"
SR="$(kubectl -n lab-11b get hpa web-khong-request \
  -o jsonpath='{.status.conditions[?(@.type=="ScalingActive")].reason}')"
echo "ScalingActive=$SA reason=$SR"

test "$SA" = 'False' \
  && echo "PASS: ScalingActive=False, ly do ghi lai la $SR"
grep -i 'unknown' "$EV/b5-get-hpa.txt" >/dev/null \
  && echo 'PASS: cot muc tieu hien unknown thay vi mot phan tram'
```

Và số replica không nhúc nhích, dù chờ:

```bash
R_BEFORE="$(kubectl -n lab-11b get deploy web-khong-request -o jsonpath='{.spec.replicas}')"
for i in $(seq 1 6); do sleep "$POLL"; done
R_AFTER="$(kubectl -n lab-11b get deploy web-khong-request -o jsonpath='{.spec.replicas}')"
echo "replicas truoc=$R_BEFORE sau=$R_AFTER"

test "$R_BEFORE" -eq "$R_AFTER" \
  && echo 'PASS: HPA khong thuc hien hanh dong nao theo metric no khong tinh duoc'
```

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B5.3. Thứ thiếu là mẫu số, không phải số đo

```bash
kubectl top pod -n lab-11b -l app=web-khong-request | tee "$EV/b5-top-pod.txt"
TPN="$(kubectl top pod -n lab-11b -l app=web-khong-request --no-headers | wc -l)"
echo "so dong top pod cua workload khong khai requests = $TPN"

test "$TPN" -ge 1 \
  && echo 'PASS: metrics-server VAN bao muc tieu thu cua Pod nay'
```

**Ý nghĩa:** đây là chỗ trực giác hay sai. metrics-server **vẫn đo được** Pod đó — tử số có đủ. Cái
thiếu là **mẫu số**: `type: Utilization` là một phép chia cho `requests`, và không có `requests` thì
không có phép chia nào để làm. Hệ quả thực tế: `<unknown>` trong `kubectl get hpa` **không** có
nghĩa là "metrics-server hỏng"; hai nguyên nhân đó cho ra cùng một triệu chứng và được phân biệt
bằng đúng lệnh `kubectl top pod` ở trên. [Mục 4](#4-troubleshooting-của-lab-này) ghi lại cách phân
biệt này.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B5.4. Dọn phần workload của B5

Trả cluster về đúng một workload trước khi sinh tải, để B6 không đo nhầm:

```bash
kubectl -n lab-11b delete hpa web-khong-request
kubectl -n lab-11b delete -f "$WK/b5-khong-request.yaml"

for i in $(seq 1 20); do
  test "$(kubectl -n lab-11b get pods -l app=web-khong-request --no-headers 2>/dev/null | wc -l)" -eq 0 && break
  sleep 3
done

HPA_LEFT="$(kubectl -n lab-11b get hpa --no-headers | wc -l)"
DEP_LEFT="$(kubectl -n lab-11b get deploy --no-headers | wc -l)"
echo "hpa con lai=$HPA_LEFT | deployment con lai=$DEP_LEFT"

test "$HPA_LEFT" -eq 1 && test "$DEP_LEFT" -eq 1 \
  && echo 'PASS: chi con dung mot Deployment va mot HPA — la cap web'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B6. Tải thật: mở rộng nhanh, thu hẹp chậm

**Mục đích:** làm mục *Tăng tải* và *Dừng tạo tải* của bài [342](../342-hpa-walkthrough-vi.md), rồi
**đo** sự bất đối xứng mà bài [72](../72-horizontal-pod-autoscale-vi.md) mô tả — mở rộng gần như
tức thì, thu hẹp thì từ từ.

> **An toàn.** Nguồn tải là một Job có `parallelism` cố định, có `limits.cpu` cho từng Pod, và có
> `activeDeadlineSeconds` để nó tự chết kể cả khi bạn bỏ đi mất. Nếu cần dừng ngay, chạy bốn lệnh ở
> [mục 2](#2-quy-ước-và-an-toàn).

### B6.1. Nguồn tải có trần và có deadline

```bash
REP_START="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
echo "so replica truoc khi sinh tai = $REP_START"

cat > "$WK/b6-load.yaml" <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: load
  namespace: lab-11b
spec:
  parallelism: ${LOAD_N}
  completions: ${LOAD_N}
  backoffLimit: 0
  activeDeadlineSeconds: ${LOAD_DEADLINE}
  template:
    metadata:
      labels:
        app: load
    spec:
      restartPolicy: Never
      containers:
      - name: gen
        image: busybox:1.37
        command: ["sh", "-c"]
        args:
          - |
            timeout ${LOAD_SEC} sh -c 'while true; do wget -q -O /dev/null http://web/; done'
            echo het-tai
        resources:
          requests:
            cpu: "${REQ_CPU_M}m"
            memory: "${REQ_MEM_MI}Mi"
          limits:
            cpu: "${LOAD_LIM_CPU_M}m"
            memory: "${LIM_MEM_MI}Mi"
EOF

kubectl apply -f "$WK/b6-load.yaml"
kubectl -n lab-11b wait --for=condition=Ready pod \
  -l batch.kubernetes.io/job-name=load --timeout=120s
kubectl -n lab-11b get pods -l batch.kubernetes.io/job-name=load -o wide
```

```bash
L_RUN="$(kubectl -n lab-11b get pods -l batch.kubernetes.io/job-name=load \
  --field-selector=status.phase=Running --no-headers | wc -l)"
L_DL="$(kubectl -n lab-11b get job load -o jsonpath='{.spec.activeDeadlineSeconds}')"
L_LIM="$(kubectl -n lab-11b get job load \
  -o jsonpath='{.spec.template.spec.containers[0].resources.limits.cpu}')"
echo "Pod sinh tai dang chay=$L_RUN | deadline=${L_DL}s | limit moi Pod=$L_LIM"

test "$L_RUN" -eq "$LOAD_N" && echo 'PASS: dung so Pod sinh tai da khai, khong hon'
test "$L_DL" -eq "$LOAD_DEADLINE" && test "$(cpu_m "$L_LIM")" -eq "$LOAD_LIM_CPU_M" \
  && echo 'PASS: nguon tai co ca tran CPU lan deadline tu tat'
```

**Ý nghĩa:** bài [342](../342-hpa-walkthrough-vi.md) sinh tải bằng một `kubectl run` chạy vòng lặp
vô hạn ở terminal khác, dừng bằng `Ctrl+C`. Trên ba VM lab thì đó là một cái bẫy: quên một terminal
là cluster chạy hết đêm. Job làm đúng việc ấy nhưng có ba lớp phanh — số Pod cố định, `limits.cpu`
cho từng Pod, và `activeDeadlineSeconds` cho cả Job.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B6.2. Đo vòng HPA mở rộng

```bash
UP_ROUND=0
for i in $(seq 1 20); do
  R="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
  U="$(kubectl -n lab-11b get hpa web \
    -o jsonpath='{.status.currentMetrics[0].resource.current.averageUtilization}')"
  echo "$(date +%T) vong=$i replicas=$R utilization=${U:-na}%"
  if [ "$R" -gt "$REP_START" ]; then UP_ROUND="$i"; break; fi
  sleep "$POLL"
done
echo "HPA mo rong o vong thu $UP_ROUND"

test "$UP_ROUND" -gt 0 \
  && echo 'PASS: HPA da mo rong duoi tai'
```

Giữ tải thêm vài vòng cho số replica ổn định, rồi chốt đỉnh:

```bash
for i in $(seq 1 6); do
  R="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
  U="$(kubectl -n lab-11b get hpa web \
    -o jsonpath='{.status.currentMetrics[0].resource.current.averageUtilization}')"
  echo "$(date +%T) giu tai vong=$i replicas=$R utilization=${U:-na}%"
  sleep "$POLL"
done

REP_PEAK="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
U_PEAK="$(kubectl -n lab-11b get hpa web \
  -o jsonpath='{.status.currentMetrics[0].resource.current.averageUtilization}')"
AV_PEAK="$(kubectl -n lab-11b get hpa web \
  -o jsonpath='{.status.currentMetrics[0].resource.current.averageValue}')"
kubectl -n lab-11b get hpa web | tee "$EV/b6-hpa-dinh.txt"
kubectl top pod -n lab-11b -l app=web | tee "$EV/b6-top-dinh.txt"
echo "dinh: replicas=$REP_PEAK utilization=${U_PEAK:-0}% averageValue=$AV_PEAK"

test "$REP_PEAK" -gt "$REP_START" && test "$REP_PEAK" -le "$MAX_REP" \
  && echo 'PASS: so replica dinh nam trong khoang min–max da khai'
```

Kiểm **phép chia** bằng chính hai số mà HPA tự báo — đây là bằng chứng trực tiếp cho câu
"`Utilization` là phần trăm của `requests`":

```bash
AV_M="$(cpu_m "${AV_PEAK:-0}")"
LHS=$(( ${U_PEAK:-0} * REQ_CPU_M ))
RHS=$(( AV_M * 100 ))
D=$(( LHS - RHS ))
if [ "$D" -lt 0 ]; then D=$(( 0 - D )); fi
echo "utilization x requests = $LHS | averageValue x 100 = $RHS | lech = $D"

test "$D" -le "$REQ_CPU_M" \
  && echo 'PASS: averageUtilization dung bang averageValue chia cho requests.cpu'
```

**Ý nghĩa:** hai vế chỉ khớp vì mẫu số là `requests.cpu` của chính Deployment này. Nhớ kỹ điều đó
cho B9: **đổi `requests.cpu` là đổi mẫu số**, và mọi ngưỡng của HPA dịch chuyển theo mà spec của HPA
không hề thay đổi một ký tự.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B6.3. Sự kiện là nơi HPA kể lại lý do

```bash
kubectl -n lab-11b describe hpa web | tee "$EV/b6-describe-dinh.txt"

grep -q 'SuccessfulRescale' "$EV/b6-describe-dinh.txt" \
  && echo 'PASS: HPA ghi su kien SuccessfulRescale khi no doi so replica'
grep -A 6 '^Events:' "$EV/b6-describe-dinh.txt"
```

**Ý nghĩa:** mỗi lần đổi số replica, HPA ghi một sự kiện kèm **lý do** — kích thước mới và metric
nào đã đẩy nó tới đó. Khi vận hành thật, đây là chỗ đọc đầu tiên để trả lời "vì sao đêm qua workload
này phình lên", chứ không phải log của controller.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B6.4. Dừng tải và đo vòng HPA thu hẹp

```bash
kubectl -n lab-11b delete job load --wait=false
for i in $(seq 1 20); do
  test "$(kubectl -n lab-11b get pods -l batch.kubernetes.io/job-name=load --no-headers 2>/dev/null | wc -l)" -eq 0 && break
  sleep 3
done
kubectl -n lab-11b get pods -o wide

test "$(kubectl -n lab-11b get pods -l batch.kubernetes.io/job-name=load --no-headers 2>/dev/null | wc -l)" -eq 0 \
  && echo 'PASS: nguon tai da tat han'
```

```bash
DOWN_ROUND=0
for i in $(seq 1 60); do
  R="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
  U="$(kubectl -n lab-11b get hpa web \
    -o jsonpath='{.status.currentMetrics[0].resource.current.averageUtilization}')"
  echo "$(date +%T) vong=$i replicas=$R utilization=${U:-na}%"
  if [ "$R" -le "$REP_START" ]; then DOWN_ROUND="$i"; break; fi
  sleep "$POLL"
done
echo "HPA thu hep xong o vong thu $DOWN_ROUND (moi vong ${POLL}s)"

{
  echo "=== $(date -Is) — chu ky 1, hanh vi mac dinh ==="
  echo "REP_START=$REP_START REP_PEAK=$REP_PEAK"
  echo "UP_ROUND=$UP_ROUND DOWN_ROUND=$DOWN_ROUND POLL=${POLL}s"
} | tee "$EV/b6-chu-ky-1.txt"

test "$DOWN_ROUND" -gt 0 \
  && echo 'PASS: HPA da thu hep ve minReplicas sau khi tai dung'
test "$DOWN_ROUND" -gt "$UP_ROUND" \
  && echo 'PASS: thu hep ton nhieu vong hon mo rong — bat doi xung do duoc bang so'
```

**Ý nghĩa:** hai con số `UP_ROUND` và `DOWN_ROUND` là **số vòng đo được trên cluster của bạn**,
không phải một cam kết của Kubernetes. Điều lab khẳng định là **quan hệ giữa chúng**, và quan hệ đó
đến từ cơ chế mà bài [72](../72-horizontal-pod-autoscale-vi.md) mô tả: trước khi co giãn, khuyến
nghị được ghi lại, và controller lấy **giá trị cao nhất trong cửa sổ ổn định** — một phép lấy cực
đại trượt. Chiều mở rộng không có cửa sổ đó nên phản ứng ngay; chiều thu hẹp có, nên một đợt tải vừa
qua vẫn "giữ" số Pod ở mức cũ thêm một lúc.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B6.5. Con số đó do ai quyết định — đọc, không sửa

```bash
sudo grep -nE 'horizontal-pod-autoscaler' /etc/kubernetes/manifests/kube-controller-manager.yaml \
  > "$EV/b6-kcm-co.txt" 2>&1 || true
cat "$EV/b6-kcm-co.txt"
KCM_FLAG="$(grep -c 'horizontal-pod-autoscaler' "$EV/b6-kcm-co.txt" || true)"
echo "so co --horizontal-pod-autoscaler-* duoc dat tuong minh = $KCM_FLAG"

test -f /etc/kubernetes/manifests/kube-controller-manager.yaml \
  && echo "PASS: doc duoc cau hinh HPA hieu luc cua kube-controller-manager (so co tuy chinh = $KCM_FLAG)"
```

**Ý nghĩa:** `0` nghĩa là cluster của bạn **không đặt cờ nào**, tức chu kỳ đồng bộ và cửa sổ ổn định
đang là mặc định dựng sẵn trong controller — bảng mặc định nằm ở mục *Hành vi mặc định* của bài
[72](../72-horizontal-pod-autoscale-vi.md), và bạn tra nó khi cần chứ không thuộc lòng. Khác `0`
nghĩa là ai đó đã chỉnh, và **giá trị thật của cluster bạn là những dòng vừa in ra**, không phải con
số trong tài liệu. Đổi chúng là sửa manifest static Pod — việc của [giai đoạn
20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), và B10.3 sẽ chứng minh
bằng checksum rằng lab này không đụng vào file đó.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B7. `behavior` — nơi sự bất đối xứng thành cấu hình

**Mục đích:** B6 đo được sự bất đối xứng nhưng chưa chỉ ra nó **nằm ở đâu**. Ở đây bạn chứng minh nó
không nằm trong object HPA, rồi kéo nó vào object bằng trường `behavior` và đo lại.

### B7.1. Object HPA mặc định không mang `behavior`

```bash
BEH="$(kubectl -n lab-11b get hpa web -o jsonpath='{.spec.behavior}')"
echo "spec.behavior = '${BEH:-<khong co>}'"
kubectl -n lab-11b get hpa web -o yaml | grep -c 'behavior' | tee "$EV/b7-behavior-truoc.txt"

test -z "$BEH" \
  && echo 'PASS: HPA khong he mang truong behavior khi ban khong dat no'
```

**Ý nghĩa:** không có `behavior` trong object **không** có nghĩa là không có hành vi. Giá trị mặc
định sống trong controller, đúng như B6.5 vừa cho thấy, chứ không được API server viết vào object.
Hệ quả vận hành: `kubectl get hpa -o yaml` **không** trả lời được câu hỏi "cửa sổ ổn định của HPA
này là bao nhiêu"; muốn biết chắc thì hoặc đặt tường minh, hoặc đọc cấu hình controller.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B7.2. Đặt tường minh một cửa sổ thu hẹp ngắn

```bash
SD_FAST=15

cat > "$WK/b7-behavior.yaml" <<EOF
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: ${SD_FAST}
EOF

kubectl -n lab-11b patch hpa web --type=merge --patch-file "$WK/b7-behavior.yaml"
kubectl -n lab-11b get hpa web -o yaml | grep -A 4 'behavior:' | tee "$EV/b7-behavior-sau.txt"

SDW="$(kubectl -n lab-11b get hpa web \
  -o jsonpath='{.spec.behavior.scaleDown.stabilizationWindowSeconds}')"
SUW="$(kubectl -n lab-11b get hpa web \
  -o jsonpath='{.spec.behavior.scaleUp.stabilizationWindowSeconds}')"
echo "scaleDown.stabilizationWindowSeconds=$SDW | scaleUp.stabilizationWindowSeconds='${SUW:-<khong dat>}'"

test "$SDW" -eq "$SD_FAST" \
  && echo 'PASS: cua so on dinh chieu thu hep gio nam trong chinh object'
test -z "$SUW" \
  && echo 'PASS: chieu mo rong khong bi dat gi — gia tri tuy chinh chi merge vao phan ban khai'
```

**Ý nghĩa:** bài [72](../72-horizontal-pod-autoscale-vi.md) nói rõ các giá trị tùy chỉnh được **hợp
nhất** với mặc định: bạn chỉ khai thứ muốn đổi. Ở đây bạn đổi đúng một trường, và `scaleUp` vẫn để
controller lo.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B7.3. Chu kỳ tải thứ hai, và so hai con số

```bash
REP_START2="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
sed -e "s/timeout ${LOAD_SEC} /timeout ${LOAD_SEC_2} /" "$WK/b6-load.yaml" > "$WK/b7-load.yaml"
grep -n 'timeout' "$WK/b7-load.yaml"

kubectl apply -f "$WK/b7-load.yaml"
kubectl -n lab-11b wait --for=condition=Ready pod \
  -l batch.kubernetes.io/job-name=load --timeout=120s

UP_ROUND2=0
for i in $(seq 1 20); do
  R="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
  echo "$(date +%T) mo rong vong=$i replicas=$R"
  if [ "$R" -gt "$REP_START2" ]; then UP_ROUND2="$i"; break; fi
  sleep "$POLL"
done

test "$UP_ROUND2" -gt 0 && echo 'PASS: chu ky hai da mo rong duoc'
```

```bash
kubectl -n lab-11b delete job load --wait=false
for i in $(seq 1 20); do
  test "$(kubectl -n lab-11b get pods -l batch.kubernetes.io/job-name=load --no-headers 2>/dev/null | wc -l)" -eq 0 && break
  sleep 3
done

DOWN_ROUND2=0
for i in $(seq 1 60); do
  R="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
  echo "$(date +%T) thu hep vong=$i replicas=$R"
  if [ "$R" -le "$REP_START2" ]; then DOWN_ROUND2="$i"; break; fi
  sleep "$POLL"
done

{
  echo "=== $(date -Is) — chu ky 2, scaleDown.stabilizationWindowSeconds=${SD_FAST} ==="
  echo "chu ky 1: UP_ROUND=$UP_ROUND DOWN_ROUND=$DOWN_ROUND"
  echo "chu ky 2: UP_ROUND=$UP_ROUND2 DOWN_ROUND=$DOWN_ROUND2"
  echo "moi vong = ${POLL}s"
} | tee "$EV/b7-chu-ky-2.txt"

test "$DOWN_ROUND2" -gt 0 \
  && echo 'PASS: chu ky hai da thu hep ve muc ban dau'
test "$DOWN_ROUND2" -lt "$DOWN_ROUND" \
  && echo 'PASS: cua so on dinh ngan hon thi thu hep ton it vong hon — do duoc bang so'
```

**Ý nghĩa:** lab không hứa "bao nhiêu giây". Nó chứng minh **quan hệ nhân quả**: cùng một workload,
cùng một kiểu tải, đổi đúng một trường trong `spec.behavior` thì số vòng cần để thu hẹp giảm hẳn.
Đó là thứ bạn mang đi được sang cluster khác, còn con số cụ thể thì không.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

---

## B8. HPA gặp bàn tay người

**Mục đích:** HPA sở hữu trường `replicas` của đối tượng đích. Mọi thứ khác cũng ghi vào trường đó —
`kubectl scale`, `kubectl apply`, một script CI — đều là **hai người cùng viết một ô**. Ở đây bạn
dựng lại đúng ba tình huống mà bài [72](../72-horizontal-pod-autoscale-vi.md) cảnh báo.

### B8.1. `kubectl scale` bị kéo ngược lại

```bash
kubectl -n lab-11b scale deployment web --replicas="$MAX_REP"
R_JUST="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
echo "ngay sau kubectl scale: replicas=$R_JUST"

test "$R_JUST" -eq "$MAX_REP" \
  && echo 'PASS: kubectl scale ghi duoc so replica ngay lap tuc'
```

```bash
BACK=0
for i in $(seq 1 40); do
  R="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
  echo "$(date +%T) vong=$i replicas=$R"
  if [ "$R" -eq 1 ]; then BACK="$i"; break; fi
  sleep "$POLL"
done
kubectl -n lab-11b describe hpa web | grep -A 6 '^Events:' | tee "$EV/b8-events-keo-ve.txt"

test "$BACK" -gt 0 \
  && echo 'PASS: HPA keo so replica ve minReplicas ma ban khong cham vao gi them'
```

**Ý nghĩa:** [Lab 4a B3](LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md#b3-scale-không-phải-là-rollout)
dạy `kubectl scale` là lệnh dứt khoát: gõ xong là xong. Có HPA rồi thì nó chỉ còn là một **đề nghị
sống được vài chục giây**. Lần thu hẹp này nhanh vì `scaleDown.stabilizationWindowSeconds` bạn đặt ở
B7.2 vẫn còn hiệu lực — một minh chứng phụ nữa cho việc trường đó có tác dụng thật.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B8.2. `minReplicas` cắt khuyến nghị, và `ScalingLimited` nói ra điều đó

```bash
MIN_HOLD=3
cat > "$WK/b8-min.yaml" <<EOF
spec:
  minReplicas: ${MIN_HOLD}
EOF

kubectl -n lab-11b patch hpa web --type=merge --patch-file "$WK/b8-min.yaml"

for i in $(seq 1 20); do
  R="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
  echo "$(date +%T) vong=$i replicas=$R"
  test "$R" -eq "$MIN_HOLD" && break
  sleep "$POLL"
done

kubectl -n lab-11b describe hpa web | tee "$EV/b8-describe-min.txt"
SL="$(kubectl -n lab-11b get hpa web \
  -o jsonpath='{.status.conditions[?(@.type=="ScalingLimited")].status}')"
SLR="$(kubectl -n lab-11b get hpa web \
  -o jsonpath='{.status.conditions[?(@.type=="ScalingLimited")].reason}')"
echo "replicas=$R | ScalingLimited=$SL reason=$SLR"

test "$R" -eq "$MIN_HOLD" \
  && echo 'PASS: HPA giu so replica o dung minReplicas du khong co tai'
test "$SL" = 'True' \
  && echo "PASS: ScalingLimited=True, ly do ghi lai la $SLR"
```

**Ý nghĩa:** không có tải thì khuyến nghị của HPA là **thấp hơn `minReplicas`**, và nó bị cắt ở sàn.
`ScalingLimited=True` chính là HPA nói với bạn: "tôi muốn ít hơn nhưng anh không cho". Cùng condition
đó chuyển sang lý do trần khi khuyến nghị vượt `maxReplicas` — đó là dấu hiệu cần **nâng trần**, chứ
không phải dấu hiệu HPA hỏng.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B8.3. Cái bẫy thật: `spec.replicas` còn trong manifest

```bash
grep -n 'replicas:' "$WK/b3-web.yaml"
kubectl apply -f "$WK/b3-web.yaml"

R_APPLY="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
echo "ngay sau kubectl apply: replicas=$R_APPLY (manifest dang ghi replicas: 1)"

test "$R_APPLY" -eq 1 \
  && echo 'PASS: apply giat so Pod ve gia tri trong manifest, bat ke HPA dang giu bao nhieu'
```

```bash
BACK2=0
for i in $(seq 1 20); do
  R="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
  echo "$(date +%T) vong=$i replicas=$R"
  if [ "$R" -eq "$MIN_HOLD" ]; then BACK2="$i"; break; fi
  sleep "$POLL"
done

test "$BACK2" -gt 0 \
  && echo 'PASS: HPA lai day len minReplicas — day dung la thrashing bai 72 canh bao'
```

**Ý nghĩa:** đây là vòng lặp phá hoại kinh điển. Mỗi lần CI chạy `kubectl apply -f deployment.yaml`,
số Pod rơi về giá trị trong file; vài chục giây sau HPA kéo lại. Ứng dụng bị xóa và tạo Pod liên tục
mà chẳng ai chạm vào cluster.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B8.4. Cách xử lý mà bài 72 khuyến nghị

Bài [72](../72-horizontal-pod-autoscale-vi.md) bảo **gỡ `spec.replicas` khỏi manifest**, và cảnh báo
rằng gỡ thẳng sẽ gây một lần tụt số Pod, vì `kubectl apply` phía client so với annotation
`last-applied-configuration` — thứ [Lab 4a
B9.2](LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md#b92-last-applied-configuration-kubectl-diff-và-apply-ghi-đè-scale-thủ-công)
đã mổ xẻ. Cách làm không cần trình soạn thảo:

```bash
grep -v '^  replicas: 1$' "$WK/b3-web.yaml" > "$WK/b8-web-khong-replicas.yaml"
grep -c 'replicas:' "$WK/b8-web-khong-replicas.yaml"

# Buoc 1: cap nhat rieng annotation last-applied, KHONG cham vao object.
kubectl apply set-last-applied -f "$WK/b8-web-khong-replicas.yaml"
R_STEP1="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
echo "sau set-last-applied: replicas=$R_STEP1"

# Buoc 2: gio moi apply file khong con replicas.
kubectl apply -f "$WK/b8-web-khong-replicas.yaml"
R_STEP2="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
echo "sau apply file khong co replicas: replicas=$R_STEP2"
```

```bash
LAST="$(kubectl -n lab-11b get deploy web \
  -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}')"
echo "$LAST" > "$EV/b8-last-applied.txt"

test "$R_STEP1" -eq "$MIN_HOLD" && test "$R_STEP2" -eq "$MIN_HOLD" \
  && echo 'PASS: apply khong con giat so Pod — HPA giu nguyen quyen so huu replicas'
echo "$LAST" | grep -q '"replicas"' \
  && echo 'FAIL: annotation last-applied van con replicas' \
  || echo 'PASS: annotation last-applied khong con truong replicas'
```

Trả `minReplicas` về `1` trước khi sang bước sau:

```bash
cat > "$WK/b8-min-lai.yaml" <<'EOF'
spec:
  minReplicas: 1
EOF
kubectl -n lab-11b patch hpa web --type=merge --patch-file "$WK/b8-min-lai.yaml"

for i in $(seq 1 20); do
  R="$(kubectl -n lab-11b get deploy web -o jsonpath='{.spec.replicas}')"
  test "$R" -eq 1 && break
  sleep "$POLL"
done
echo "replicas sau khi ha minReplicas ve 1 = $R"

test "$R" -eq 1 && echo 'PASS: workload da ve mot replica, san sang cho B9 va B10'
```

**Ý nghĩa:** `kubectl apply set-last-applied` làm đúng việc mà bài 72 mô tả bằng
`apply edit-last-applied`, nhưng không cần mở trình soạn thảo nên chạy được trong runbook. Sau bước
này, **không file nào và không annotation nào còn nhắc tới `replicas`** — quyền sở hữu trường đó
thuộc trọn về HPA, và đó mới là trạng thái đúng khi một workload được tự động co giãn.

**PASS:** ba dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

---

## B9. Trục dọc: VPA không cài được, và phần vẫn hiểu được

**Mục đích:** đóng nốt trục thứ hai của bài [71](../71-autoscaling-vi.md) một cách trung thực. Phần
kiểm chứng được thì kiểm bằng lệnh; phần không kiểm chứng được thì nói rõ là đọc hiểu và ghi vào sổ
nợ, chứ không giả vờ đã làm.

### B9.1. Chốt lại năng lực và ghi vào sổ nợ

```bash
{
  echo "=== $(date -Is) — nang luc autoscaling cua cluster tai Lab 11b ==="
  echo "--- truc ngang tu dong"
  echo "hpa dang chay: $(kubectl -n lab-11b get hpa web -o jsonpath='{.metadata.name}')"
  echo "so tai nguyen horizontalpodautoscalers: $(kubectl api-resources --api-group=autoscaling --no-headers | grep -c 'horizontalpodautoscalers')"
  echo "--- truc doc tu dong"
  echo "so CRD ten mien autoscaling.k8s.io: $(kubectl get crd 2>/dev/null | grep -c 'autoscaling.k8s.io' || true)"
  echo "so tai nguyen verticalpodautoscalers: $(kubectl api-resources 2>/dev/null | grep -c 'verticalpodautoscalers' || true)"
  echo "--- so no #1"
  echo "phan HPA: DA TRA tai Lab 11b (B3 den B8)"
  echo "phan VPA: CHUA TRA — thieu add-on VPA, xem ly do o muc 1.1"
} | tee "$EV/b9-so-no.txt"

grep -q 'so CRD ten mien autoscaling.k8s.io: 0' "$EV/b9-so-no.txt" \
  && echo 'PASS: ghi vao evidence bang chung rang cluster khong co VPA'
grep -q 'phan VPA: CHUA TRA' "$EV/b9-so-no.txt" \
  && echo 'PASS: so no ghi ro phan nao cua no #1 con treo'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B9.2. Vì sao VPA và HPA cùng theo CPU trên một workload là xung đột

> **Mục này không có gate.** Trên cluster baseline không có object VPA nào để đo, nên đây là phần
> **đọc hiểu**, không phải thực nghiệm. Nó được kiểm ở [checkpoint](#3-checkpoint-11b) bằng vấn đáp,
> không phải bằng lệnh. Lập luận dưới đây chỉ dùng những gì bạn **đã đo được** ở B6.2 cộng những gì
> bài [73](../73-vertical-pod-autoscale-vi.md) trình bày.

Nối ba mảnh lại:

1. **B6.2 đã đo:** `averageUtilization` bằng `averageValue` chia cho `requests.cpu`. Ngưỡng hành
   động của HPA — con số `nguong tuyet doi` ghi ở `b0-tham-so.txt` — là `TARGET_PCT` phần trăm của
   **`requests.cpu`**, chứ không phải một mức mili-core cố định.
2. **Bài [73](../73-vertical-pod-autoscale-vi.md) nói:** việc của VPA là **viết lại chính
   `requests`** đó, dựa trên mức sử dụng quá khứ.
3. **Ghép lại:** cho VPA quản `cpu` của cùng workload mà HPA đang theo `Utilization` của `cpu` nghĩa
   là để một controller đổi **mẫu số** trong khi controller kia đang chia cho nó. Tải không đổi một
   chút nào, nhưng phần trăm nhảy, nên HPA đổi số replica; số replica đổi làm mức tiêu thụ trên mỗi
   Pod đổi, nên VPA lại đổi `requests`. Đó là một vòng lặp tự kích, đúng hiện tượng *thrashing* mà
   bài [72](../72-horizontal-pod-autoscale-vi.md) gọi tên ở mục *Tính ổn định của quy mô workload*.

Hai lối thoát mà tài liệu để lại, và cả hai đều đọc được từ chính hai bài:

- Cho hai autoscaler **hai tài nguyên khác nhau**: HPA theo `cpu`, VPA chỉ quản `memory` qua
  `controlledResources`. Không còn mẫu số dùng chung thì không còn vòng lặp.
- Hoặc để VPA ở `updateMode: Off` — nó vẫn phân tích và vẫn ghi khuyến nghị vào `.status`, nhưng
  **không áp dụng gì**. Bạn đọc khuyến nghị bằng mắt rồi tự quyết. Đây là cách dùng VPA phổ biến
  nhất trên cluster đã có HPA.

Còn `updateMode` nghĩa là gì, tóm đúng theo bài [73](../73-vertical-pod-autoscale-vi.md): nó quyết
định **khuyến nghị được áp dụng lúc nào và Pod có bị làm phiền hay không** — `Off` chỉ khuyến nghị;
`Initial` chỉ đặt lúc Pod được tạo lần đầu; `Recreate` **evict Pod** để giá trị mới có hiệu lực, và
thành phần thật sự ghi giá trị lên Pod mới là **admission controller** của VPA chứ không phải
updater; `InPlaceOrRecreate` thử sửa tại chỗ rồi mới quay về evict; `InPlace` **không bao giờ
evict**, nó hoãn và thử lại; `Auto` đã lỗi thời và hiện chỉ là bí danh của `Recreate`.

Cuối cùng, đừng nhầm hai thứ: **co giãn dọc thủ công thì cluster này làm được**, và bạn đã làm rồi ở
[Lab 3c B6](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md#b6-resize-tài-nguyên-của-container-đang-chạy) —
đổi `resources` của một container đang chạy, đúng như bài
[289](../289-resize-container-resources-vi.md) mô tả. Thứ vắng mặt là **cái quyết định thay bạn**:
recommender và updater của VPA. Bài [71](../71-autoscaling-vi.md) xếp hai thứ đó vào cùng một trục
nhưng khác hẳn nhau ở chỗ ai bấm nút.

### B9.3. Điều kiện để trả nốt nợ #1

Phần VPA của [nợ #1](README.md#5-sổ-nợ-lab) chỉ trả được khi **cả ba** điều kiện dưới đây có mặt
cùng lúc — thiếu một là lại vi phạm một nguyên tắc của thư mục lab:

1. **Bảng [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) có một dòng
   `vertical-pod-autoscaler` với version đã chốt**, đối chiếu theo bảng tương thích của chính dự án
   với minor Kubernetes ở [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) — đúng cách
   Lab 11a đã làm với metrics-server.
2. **Người học đã qua [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng)**,
   tức đã hiểu CRD và mẫu Operator. Cài VPA là cài đúng một Operator; làm trước khi học nó là nhảy
   cóc.
3. **Có một mốc snapshot để chứa nó.** VPA thêm CRD, ba Deployment và một mutating webhook vào
   cluster; hoặc lab trả nợ đó tự gỡ sạch trước khi kết thúc, hoặc nó phải là một lab **tạo mốc
   mới**. Lab 11b không phải cái nào trong hai loại đó.

```bash
grep -q 'phan HPA: DA TRA' "$EV/b9-so-no.txt" \
  && echo 'PASS: evidence ghi ro phan HPA cua no #1 da tra tai lab nay'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B10. Cleanup và gate cuối

**Mục đích:** xóa sạch object của bài học, chứng minh **metrics-server và cấu hình node không hề bị
lab này đụng vào**, và chứng minh cluster đã trở về đúng `04-metrics-ready`. Lab **không chụp
snapshot mới**, nên gate cuối là thứ duy nhất bảo đảm lab sau mở được.

### B10.1. Xóa object của bài học

```bash
kubectl delete namespace lab-11b --wait=true --timeout=300s

rm -f "$WK/b2-vpa.yaml" "$WK/b3-web.yaml" "$WK/b4-hpa.yaml" \
      "$WK/b5-khong-request.yaml" "$WK/b6-load.yaml" \
      "$WK/b7-behavior.yaml" "$WK/b7-load.yaml" \
      "$WK/b8-min.yaml" "$WK/b8-min-lai.yaml" "$WK/b8-web-khong-replicas.yaml"
rmdir "$WK"
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` bên dưới biến điều đó
thành gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/11b/` **giữ lại** — đó là bằng chứng.

```bash
NS_LEFT="$(kubectl get namespace lab-11b --ignore-not-found -o name | wc -l)"
HPA_ALL="$(kubectl get hpa -A --no-headers 2>/dev/null | wc -l)"
JOB_ALL="$(kubectl get job -A --no-headers 2>/dev/null | wc -l)"
echo "namespace lab-11b con=$NS_LEFT | hpa toan cluster=$HPA_ALL | job toan cluster=$JOB_ALL"

test "$NS_LEFT" -eq 0 && echo 'PASS: namespace lab-11b da bien mat'
test "$HPA_ALL" -eq 0 \
  && echo 'PASS: khong con HorizontalPodAutoscaler nao trong cluster'
test "$JOB_ALL" -eq 0 \
  && echo 'PASS: khong con Job sinh tai nao sot lai'
test ! -e "$WK" && echo 'PASS: manifest tam da xoa'
```

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B10.2. Gate toàn vẹn: metrics-server không bị lab này đụng vào

```bash
{
  echo "image=$(kubectl -n kube-system get deploy metrics-server \
    -o jsonpath='{.spec.template.spec.containers[0].image}')"
  kubectl -n kube-system get deploy metrics-server \
    -o jsonpath='{range .spec.template.spec.containers[0].args[*]}arg={@}{"\n"}{end}'
} | tee "$EV/b10-metrics-server.txt"

diff -u "$EV/b0-metrics-server.txt" "$EV/b10-metrics-server.txt" \
  && echo 'PASS: image va args cua metrics-server khong doi mot ky tu nao' \
  || echo 'FAIL: Deployment metrics-server da bi sua — xem muc 4'
```

```bash
{
  echo "$MASTER kubelet $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in "$W1" "$W2"; do
    echo "$n kubelet $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
  for f in kube-apiserver kube-scheduler kube-controller-manager; do
    echo "$MASTER $f $(sudo sha256sum /etc/kubernetes/manifests/$f.yaml | awk '{print $1}')"
  done
} | tee "$EV/b10-config-sha.txt"

diff -u "$EV/b0-config-sha.txt" "$EV/b10-config-sha.txt" \
  && echo 'PASS: cau hinh kubelet va manifest control plane khong he doi trong suot lab' \
  || echo 'FAIL: co file cau hinh da bi sua — xem muc 4'
```

**Ý nghĩa:** B6.5 dừng ngay trước cửa manifest của kube-controller-manager, và cám dỗ "hạ cửa sổ ổn
định xuống cho lab chạy nhanh hơn" là có thật. Gate này biến lời hứa "chỉ đọc" thành thứ kiểm chứng
được. Nó quan trọng gấp đôi vì lab **không chụp snapshot mới**: một sửa đổi lén lút sẽ nằm lại trên
cluster mà không mốc nào ghi nhận, và lab sau sẽ chạy trên một baseline không ai biết đã lệch.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B10.3. Gate trạng thái của mốc `04-metrics-ready`

```bash
MS_READY="$(kubectl -n kube-system get deploy metrics-server -o jsonpath='{.status.readyReplicas}')"
AV="$(kubectl get apiservice v1beta1.metrics.k8s.io \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
TN="$(kubectl top node --no-headers | wc -l)"
TP="$(kubectl top pod -n kube-system --no-headers | wc -l)"
echo "metrics-server readyReplicas=${MS_READY:-0} | APIService Available=$AV"
echo "kubectl top node=$TN dong | kubectl top pod kube-system=$TP dong"

test "${MS_READY:-0}" -ge 1 && test "$AV" = 'True' \
  && echo 'PASS: metrics-server van chay va Metrics API van san sang'
test "$TN" -eq 3 && test "$TP" -gt 0 \
  && echo 'PASS: kubectl top node va kubectl top pod van chay duoc'
```

Tầng lưu trữ phải giữ nguyên định nghĩa thừa kế từ mốc trước, không nhiễm gì từ lab này:

```bash
PV_ALL="$(kubectl get pv --no-headers 2>/dev/null | wc -l)"
PVC_ALL="$(kubectl get pvc -A --no-headers 2>/dev/null | wc -l)"
SC_ALL="$(kubectl get sc --no-headers | wc -l)"
SC_NOW="$(kubectl get sc -o jsonpath='{.items[0].metadata.name}')"
SC_DEF="$(kubectl get sc "$SC_NOW" \
  -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}')"
echo "pv=$PV_ALL pvc=$PVC_ALL storageclass=$SC_ALL ($SC_NOW, default=$SC_DEF)"

test "$PV_ALL" -eq 0 && test "$PVC_ALL" -eq 0 \
  && echo 'PASS: khong con PV hay PVC nao'
test "$SC_ALL" -eq 1 && test "$SC_DEF" = 'true' \
  && echo 'PASS: van dung mot StorageClass mac dinh nhu moc quy dinh'
```

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B10.4. Gate ngắn A5.5 và kết thúc

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
kubectl get pods -A -o wide | tee "$EV/b10-pods.txt"

{
  echo "=== $(date -Is) — trang thai sau Lab 11b ==="
  echo '--- hpa toan cluster';  kubectl get hpa -A 2>&1
  echo '--- crd autoscaling';   kubectl get crd 2>/dev/null | grep 'autoscaling.k8s.io' || echo 'khong co'
  echo '--- namespaces';        kubectl get namespaces
  echo '--- pv';                kubectl get pv 2>&1
} | tee "$EV/b10-sau.txt"

diff -u "$EV/b0-truoc.txt" "$EV/b10-sau.txt" > "$EV/b10-diff.txt" 2>&1 || true

grep -qi 'lab-11b' "$EV/b10-pods.txt" \
  && echo 'FAIL: con Pod cua lab trong cluster' \
  || echo 'PASS: khong con Pod nao cua lab 11b'
grep -q 'metrics-server' "$EV/b10-pods.txt" \
  && echo 'PASS: metrics-server van nam trong danh sach Pod dang chay'
```

**PASS:** ba node `Ready`; dòng `PASS: readyz ok` xuất hiện; lệnh field selector trả
`No resources found`; CoreDNS đủ replica `READY`; `default` không có Pod; hai dòng `PASS:` ở trên
xuất hiện, không dòng `FAIL:` nào. File `b10-diff.txt` phải **không có khác biệt đáng kể** ngoài
dòng thời gian — nếu nó cho thấy thứ gì khác, dừng lại và tìm nguyên nhân trước khi sang lab sau.

Cluster đã trở về `04-metrics-ready`. **Lab 11b không tạo snapshot mới** — mốc cũ vẫn là điểm bắt
đầu của các lab 12, 13, 14 và 15. Để ba VM đang chạy hoặc tắt tùy bạn; lab sau tự bật máy theo
[A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab).

---

## 3. Checkpoint 11b

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Bài 71 đặt tên hai trục co giãn. Với **mỗi trục**, nói rõ: bản thủ công làm bằng lệnh gì, bản
      tự động do cái gì làm, và trên cluster lab của bạn thì bản tự động đó **có mặt hay không**.
      Bạn chứng minh phần "không có mặt" bằng lệnh nào?
- [ ] Bạn `kubectl apply` một `kind: HorizontalPodAutoscaler` và một `kind: VerticalPodAutoscaler`
      lên cluster lab. Hai lệnh đó hỏng theo **hai kiểu khác nhau** — kể tên từng kiểu và giải thích
      vì sao sự khác nhau đó bắt nguồn từ chỗ hai API được định nghĩa ở đâu.
- [ ] Một HPA đặt `averageUtilization: 50`. Nó so mức tiêu thụ với **cái gì**? Nếu container không
      khai `requests.cpu` thì triệu chứng ở `kubectl get hpa` là gì, condition nào chuyển sang
      `False`, và làm sao phân biệt tình huống đó với tình huống metrics-server chết? Bạn đã chứng
      minh sự phân biệt ấy bằng lệnh nào?
- [ ] `kubectl get hpa` báo `TARGETS` là `300% / 50%` và Deployment đang có 2 replica. Theo công
      thức của bài 72, HPA đặt bao nhiêu replica? Con số đó có thể bị cắt bởi hai thứ — kể tên cả
      hai, và nói condition nào sẽ nói cho bạn biết chuyện cắt đã xảy ra.
- [ ] Bạn muốn HPA tự tăng số Pod của một DaemonSet. Được không, và **lý do cơ học** là gì? Bạn đã
      kiểm điều đó bằng đúng một lệnh — lệnh nào, và nó trả về gì cho Deployment so với cho
      DaemonSet?
- [ ] Vì sao HPA mở rộng nhanh hơn hẳn lúc nó thu hẹp? Cơ chế nằm ở đâu — trong object HPA hay trong
      controller? Bạn chứng minh vị trí đó bằng lệnh nào, và bạn đã làm gì để kéo nó vào object rồi
      đo lại?
- [ ] Trên cluster của bạn, chu kỳ đồng bộ và cửa sổ ổn định của HPA là bao nhiêu? Câu trả lời đúng
      **không phải một con số** — giải thích vì sao, và nói bạn tra ở đâu khi cần biết chắc.
- [ ] Bạn `kubectl scale deployment web --replicas=8` trên một Deployment đang có HPA. Chuyện gì xảy
      ra trong vòng vài chục giây tiếp theo, và vì sao? Câu trả lời đổi thế nào nếu bạn thay lệnh đó
      bằng `kubectl apply -f deployment.yaml` mà file còn `spec.replicas: 8`?
- [ ] Manifest Deployment của bạn còn `spec.replicas` trong khi HPA đang chạy. Nêu vòng lặp phá hoại
      sinh ra từ đó, cách xử lý mà bài 72 khuyến nghị, và **vì sao gỡ thẳng dòng đó rồi apply lại là
      chưa đủ**. Hai bước bạn đã chạy ở B8.4 là gì?
- [ ] Bạn bật VPA quản `cpu` cho đúng workload mà HPA đang theo `Utilization` của `cpu`. Vòng lặp
      tự kích sinh ra thế nào? Nêu hai cách tránh mà tài liệu để lại. Kể tên ba thành phần của VPA
      và nói **thành phần nào** thật sự ghi giá trị mới lên Pod ở `updateMode: Recreate`.
- [ ] **Nợ #1 của [sổ nợ lab](README.md#5-sổ-nợ-lab) giờ ở tình trạng nào?** Nói rõ phần nào đã trả
      xong và trả ở những bước nào; phần nào **chưa** trả, vì đúng ba lý do gì; và ba điều kiện phải
      có cùng lúc để trả nốt phần còn lại.
- [ ] Lab này không chụp snapshot mới. Vậy thứ gì bảo đảm lab sau mở được? Kể ba nhóm gate ở B10 và
      nói mỗi nhóm bắt được loại sai lệch nào.

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng sau, không nhìn tài liệu:

1. **Luồng một chu kỳ HPA, từ lúc CPU tăng tới lúc Pod mới chạy và ngược lại.** Bắt đầu từ một
   container đang bận trên `lab-k8s-worker2`. Kể đủ: ai đo mức tiêu thụ đó và nó xuất hiện ở API
   nào; controller nào đọc nó, theo chu kỳ hay liên tục; nó tìm Pod bằng cách nào; **phép chia** nó
   làm là gì và mẫu số ở đâu ra; kết quả được ghi vào **trường nào của object nào, qua cửa nào**; và
   ai biến trường đó thành Pod thật. Rồi kể đường về: vì sao đường về chậm hơn đường đi, cơ chế đó
   tên gì, nó nằm trong object hay trong controller, và bạn đổi nó ở đâu. Ở **mỗi mắt xích**, nêu
   một cách nó có thể hỏng cùng triệu chứng nhìn thấy được ở `kubectl get hpa`.
2. **Luồng trục dọc, và biên giới của lab này.** Bắt đầu từ câu hỏi "Pod này xin quá nhiều CPU, ai
   sửa `requests` cho tôi". Kể đủ: cách thủ công là gì và bạn đã làm nó ở lab nào; bản tự động tên
   gì, gồm ba thành phần nào, mỗi thành phần làm gì và cái nào là mutating webhook; vì sao nó
   **không có** trên cluster lab trong khi HPA thì có; và điều gì xảy ra nếu bạn bật nó lên cạnh HPA
   trên cùng một workload theo cùng một tài nguyên. Kết lại bằng ba điều kiện để trả nốt nợ #1 và
   nói rõ vì sao thiếu một điều kiện là phải để nợ chứ không được "cài thử cho xong".

Khi mọi checkbox được đánh dấu và bạn không còn nhầm **`<unknown>` do thiếu `requests`** với
**`<unknown>` do hỏng metrics-server**, **`AbleToScale`** với **`ScalingActive`**, **cửa sổ ổn định**
với **chu kỳ đồng bộ**, hay **co giãn dọc thủ công** với **VPA** — Lab 11b hoàn tất.

Lab này **trả một phần của [nợ #1](README.md#5-sổ-nợ-lab)**: toàn bộ phần HPA đã trả, làm ở B3 đến
B8 trên cluster thật với tải thật. **Phần VPA vẫn còn treo** vì cluster baseline không có add-on đó
và lab không được phép cài thêm hạ tầng; ba điều kiện để trả nốt nằm ở B9.3. Lab **không phát sinh
nợ mới**. Những phần cố ý không làm đều có lý do ở [bảng mục
1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành) và thuộc đúng thứ tự lộ trình đã định —
[giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng),
[20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) và
[23](../00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) — chứ không phải nợ.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Sự cố của
metrics-server nằm ở [mục 4 của Lab 11a](LAB-11A-OBSERVABILITY.md#4-troubleshooting-của-lab-này).
Bảng dưới chỉ liệt kê sự cố phát sinh từ nội dung bài học 11b.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| **B4 hoặc B6: `kubectl get hpa` hiện `<unknown>` ở cột mục tiêu** | `kubectl -n lab-11b describe hpa web \| grep -A 4 Conditions`; `kubectl top pod -n lab-11b` | Hai nguyên nhân cho cùng một triệu chứng, phân biệt bằng đúng lệnh `top pod`. **Nếu `top pod` có số:** tử số có, mẫu số thiếu — container không khai `requests.cpu`, đọc lại B5.3 và kiểm `resources.requests` của Deployment. **Nếu `top pod` cũng rỗng hoặc báo lỗi:** nguồn metric mới là gốc — quay lại [B1](#b1-gate-điều-kiện-đầu-vào--nguồn-metric-phải-chạy-thật), và nếu B1 fail thì về [Lab 11a B5](LAB-11A-OBSERVABILITY.md#b5-cài-metrics-server-và-chữa-lỗi-certificate). **Không** sửa target của HPA để "cho nó chạy" |
| **B6.2: tải chạy nhưng HPA không mở rộng, `utilization` mãi thấp hơn mục tiêu** | `kubectl -n lab-11b logs -l batch.kubernetes.io/job-name=load --tail=5`; `kubectl top pod -n lab-11b -l app=web`; so `utilization` với `TARGET_PCT` | Đi theo thứ tự này, đừng nhảy cóc. **Một:** Pod sinh tải có gọi được Service không — chạy lại B3.2; nếu `wget` hỏng thì DNS hoặc Service mới là gốc, không phải HPA. **Hai:** tải có thật sự tới không — `kubectl top pod -l app=web` phải cho số lớn hơn ngưỡng ghi ở `b0-tham-so.txt`. **Ba:** nếu tải tới mà vẫn thiếu, tăng `LOAD_N` thêm **một** Pod rồi apply lại Job, và **chạy lại gate trần ở B0.3** trước khi tăng — trần đó là thứ giữ cluster không treo. **Không** hạ `requests.cpu` của Deployment để "cho dễ vượt": làm vậy là đổi mẫu số giữa chừng và mọi số đo trước đó thành vô nghĩa |
| B6.2: HPA nhảy thẳng lên `maxReplicas` ngay vòng đầu | `kubectl -n lab-11b describe hpa web \| grep -A 6 Events`; đọc `utilization` ở đỉnh | Đây **không phải sự cố**. `requests.cpu` được đặt thấp có chủ đích nên tải vượt mục tiêu nhiều lần, và công thức `desiredReplicas` cho ra số lớn hơn trần. `ScalingLimited` sẽ là `True` với lý do chạm trần — ghi lại vào evidence và đi tiếp; trần đã được gate ở B0.3 nên cluster vẫn an toàn |
| B6.4 hoặc B7.3: chờ hết 60 vòng mà số replica không về mức ban đầu | `kubectl -n lab-11b get pods -l batch.kubernetes.io/job-name=load`; `kubectl top pod -n lab-11b -l app=web` | Còn Pod sinh tải sót lại, hoặc `activeDeadlineSeconds` chưa tới. Chạy lệnh 1 trong [mục 2](#2-quy-ước-và-an-toàn) để cắt nguồn tải rồi đếm lại từ đầu. Nếu tải đã tắt hẳn mà vẫn không xuống, đọc `describe hpa` xem `minReplicas` có bị để lại ở giá trị cao từ B8.2 không |
| B6.4: `DOWN_ROUND` không lớn hơn `UP_ROUND` | `cat ~/lab-evidence/11b/b6-chu-ky-1.txt`; `cat ~/lab-evidence/11b/b6-kcm-co.txt` | Nếu `b6-kcm-co.txt` cho thấy cluster đặt `--horizontal-pod-autoscaler-downscale-stabilization` rất nhỏ thì kết quả này **đúng với cấu hình thật của bạn** — ghi vào evidence và chuyển sang so cặp `DOWN_ROUND` / `DOWN_ROUND2` ở B7.3, đó mới là phép so có kiểm soát. **Không** sửa manifest kube-controller-manager để "đưa về mặc định"; B10.2 sẽ bắt được |
| B7.2: `patch --patch-file` báo lỗi | `cat ~/lab-work/11b/b7-behavior.yaml`; `kubectl -n lab-11b get hpa web -o yaml \| head -20` | File patch phải bắt đầu bằng `spec:` và thụt đúng hai mức. Nếu object của bạn đang ở `autoscaling/v1` thì trường `behavior` không tồn tại — kiểm lại B4.4 và tạo lại HPA từ manifest `autoscaling/v2` |
| B8.4: `kubectl apply set-last-applied` báo không tìm thấy annotation | `kubectl -n lab-11b get deploy web -o jsonpath='{.metadata.annotations}'` | Deployment được tạo bằng `create` chứ không phải `apply` nên chưa có annotation. Thêm `--create-annotation=true` vào chính lệnh đó. Đừng `kubectl edit` để chèn tay |
| B8.4: apply xong replicas vẫn tụt về 1 | `cat ~/lab-evidence/11b/b8-last-applied.txt \| grep replicas`; `grep -c replicas ~/lab-work/11b/b8-web-khong-replicas.yaml` | Bạn chạy `apply` trước `set-last-applied`, hoặc dòng `replicas` chưa bị lọc khỏi file. Kiểm file trước, chạy lại đúng thứ tự hai bước của B8.4 |
| B5.2: `ScalingActive` mãi rỗng, không `True` cũng không `False` | `kubectl -n lab-11b get hpa web-khong-request -o yaml \| tail -20` | Controller chưa chạy vòng đầu tiên cho object mới. Chờ hết vòng lặp trong bước đó. Nếu sau đó vẫn rỗng, kiểm `kubectl -n kube-system get pods -l component=kube-controller-manager` |
| B3.2: `wget` từ Pod client không lấy được trang | `kubectl -n lab-11b get endpointslice -l kubernetes.io/service-name=web`; `kubectl -n lab-11b logs deploy/web --tail=10` | EndpointSlice rỗng nghĩa là selector của Service không khớp label Pod — đọc lại manifest. Nếu EndpointSlice có mà vẫn không gọi được, đây là sự cố mạng chứ không phải sự cố của lab: chạy [tầng 5 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) |
| B6.1: Pod sinh tải `Pending` | `kubectl -n lab-11b describe pod -l batch.kubernetes.io/job-name=load \| tail -20` | Hai worker hết chỗ, thường vì một lab trước để lại workload. Chạy lệnh 4 trong [mục 2](#2-quy-ước-và-an-toàn), kiểm `kubectl get pods -A` rồi bắt đầu lại từ B0. **Không** gỡ taint của master để có thêm chỗ |
| B10.1: namespace `lab-11b` kẹt `Terminating` | `kubectl get all -n lab-11b`; `kubectl -n lab-11b get pods` | Thường là Pod sinh tải còn trong grace period. Chờ; **không** cưỡng chế finalizer của Namespace |
| **B10.2: `diff` báo checksum hoặc args khác** | `diff -u ~/lab-evidence/11b/b0-config-sha.txt ~/lab-evidence/11b/b10-config-sha.txt`; `diff -u ~/lab-evidence/11b/b0-metrics-server.txt ~/lab-evidence/11b/b10-metrics-server.txt` | Một thứ lab hứa không đụng vào đã bị sửa. Vì lab **không chụp snapshot**, sai lệch này sẽ đi theo cluster mà không mốc nào ghi nhận. Tắt cả ba VM, restore cả ba về `04-metrics-ready` — không bao giờ restore riêng một VM — rồi làm lại từ B0 |
| B10.3: `kubectl top` fail ở gate cuối dù B1 đã PASS | `kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide` | Pod metrics-server bị lập lịch lại sau khi namespace `lab-11b` bị xóa và chưa thu thập xong. Chờ rồi chạy lại B10.3. **Không sang lab sau khi gate này chưa PASS** — mốc `04-metrics-ready` được định nghĩa bằng chính `kubectl top` chạy được |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Autoscaling Workloads](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/autoscaling/)
  — nguồn của bài [71](../71-autoscaling-vi.md), dùng ở B2 và B9
- [Kubernetes v1.35 — Horizontal Pod Autoscaling](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
  — nguồn của bài [72](../72-horizontal-pod-autoscale-vi.md); công thức, dung sai, cửa sổ ổn định và
  `behavior` dùng ở B4, B6, B7 và B8
- [Kubernetes v1.35 — Vertical Pod Autoscaling](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/)
  — nguồn của bài [73](../73-vertical-pod-autoscale-vi.md); ba thành phần và các `updateMode` dùng ở
  B9
- [Kubernetes v1.35 — HorizontalPodAutoscaler Walkthrough](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
  — nguồn của bài [342](../342-hpa-walkthrough-vi.md); hình dạng workload và phụ lục status
  condition dùng ở B3, B4, B6
- [Kubernetes v1.35 — HorizontalPodAutoscaler API `autoscaling/v2`](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/)
  — nguồn của tên trường trong manifest ở B4.4 và của patch `behavior` ở B7.2
- [Kubernetes v1.35 — `kubectl autoscale`](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_autoscale/)
  — nguồn của cờ dùng ở B4.1 và B5.1
- [Kubernetes v1.35 — Resource metrics pipeline](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
  — nguồn của mô tả Metrics API kiểm ở B1; bài dịch tương ứng là
  [311](../311-resource-metrics-pipeline-vi.md)
- [Kubernetes v1.35 — Resize CPU and Memory Resources assigned to Containers](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)
  — nguồn của co giãn dọc **thủ công** nhắc ở B9.2; đã thực hành ở
  [Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md#b6-resize-tài-nguyên-của-container-đang-chạy)
- [metrics-server — README và bảng tương thích](https://github.com/kubernetes-sigs/metrics-server#compatibility-matrix)
  — nguồn để chốt dòng metrics-server ở
  [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00), điều kiện đầu vào của lab
  này
- [Vertical Pod Autoscaler — kho GitHub](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
  — nơi lấy bảng tương thích và manifest nếu về sau trả nốt phần VPA của nợ #1, theo ba điều kiện ở
  B9.3

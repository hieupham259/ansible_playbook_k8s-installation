# Thay đổi Access Mode của một PersistentVolume sang ReadWriteOncePod (Change the Access Mode of a PersistentVolume to ReadWriteOncePod)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/change-pv-access-mode-readwriteoncepod/>
>
> Trang này hướng dẫn cách thay đổi access mode trên một PersistentVolume hiện có để dùng
> `ReadWriteOncePod`.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 26 — Vận hành lưu trữ](00-ALO-TRINH-ADMIN.md#giai-đoạn-26--vận-hành-lưu-trữ),
bài 3/4 · Phần II không có lab riêng: kiểm chứng bằng **Checkpoint của chính giai đoạn 26** trên
cluster lab dựng ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) — đổi StorageClass mặc định, rồi
đổi `reclaimPolicy` của một PV đang tồn tại từ `Delete` sang `Retain`, xóa PVC và chứng minh dữ
liệu còn nguyên.

Bài đứng ngay sau bài [194 — Thay đổi Reclaim Policy của một PersistentVolume](194-change-pv-reclaim-policy-vi.md)
(bài 2/4 của giai đoạn) và phải làm **sau** bài đó, vì bước đầu tiên của quy trình ở đây dùng
chính thao tác đổi reclaim policy.

Khái niệm access mode (trong đó có `ReadWriteOncePod`) bạn đã đọc ở
[bài 92](92-persistent-volumes-vi.md). Lưu ý: phần thực hành yêu cầu CSI driver hỗ trợ
`ReadWriteOncePod`; nếu provisioner của [Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md) không hỗ trợ, giữ ở mức đọc hiểu quy trình.

**Phải hiểu ở lần đọc này:**

- Hạn chế của `ReadWriteOnce`: nó giới hạn truy cập theo từng **node**, nên **nhiều Pod trên
  cùng một node vẫn đọc/ghi đồng thời cùng một volume** — rủi ro cho ứng dụng cần quyền ghi đơn
  (single-writer) nghiêm ngặt. `ReadWriteOncePod` siết xuống đúng một Pod.
- Phạm vi hỗ trợ: `ReadWriteOncePod` **chỉ dành cho CSI volume**, và chỉ hỗ trợ di chuyển từ
  `ReadWriteOnce` sang `ReadWriteOncePod`.
- Trình tự di chuyển an toàn: đổi reclaim policy sang `Retain` **trước** (để PV không bị xóa
  theo PVC) → dừng workload rồi xóa PVC → xóa trắng `spec.claimRef.uid` trên PV → patch
  `accessModes` → tạo lại PVC với `ReadWriteOncePod` và `spec.volumeName` → khôi phục reclaim
  policy nếu cần.
- `ReadWriteOncePod` **không thể kết hợp với access mode khác**: khi cập nhật, nó phải là access
  mode duy nhất trên PV, nếu không request thất bại.
- Vì sao phải xóa `spec.claimRef.uid`: để PVC được tạo lại có thể bind vào đúng PV cũ.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ghi chú về phiên bản tối thiểu của các CSI sidecar (csi-provisioner, csi-attacher, csi-resizer) | chỉ cần kiểm tra khi tự vận hành một CSI driver đời cũ | Lab 6a, khi chọn provisioner |
| Ghi chú về feature gate cho Kubernetes cũ hơn v1.29 | cluster lab chạy phiên bản đã có tính năng ở mức stable | không cần |

---

Trang này hướng dẫn cách thay đổi access mode trên một PersistentVolume hiện có để dùng
`ReadWriteOncePod`.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn v1.22. Để kiểm tra phiên bản, nhập
`kubectl version`.

> **Ghi chú:** Access mode `ReadWriteOncePod` đã lên mức ổn định (stable) trong bản phát hành
> Kubernetes v1.29. Nếu bạn đang chạy một phiên bản Kubernetes cũ hơn v1.29, bạn có thể cần bật
> một feature gate. Hãy kiểm tra tài liệu cho phiên bản Kubernetes của bạn.

> **Ghi chú:** Access mode `ReadWriteOncePod` chỉ được hỗ trợ cho các CSI volume. Để dùng access
> mode này, bạn sẽ cần cập nhật các
> [CSI sidecar](https://kubernetes-csi.github.io/docs/sidecar-containers.html) sau lên các phiên
> bản này hoặc mới hơn:
>
> * [csi-provisioner:v3.0.0+](https://github.com/kubernetes-csi/external-provisioner/releases/tag/v3.0.0)
> * [csi-attacher:v3.3.0+](https://github.com/kubernetes-csi/external-attacher/releases/tag/v3.3.0)
> * [csi-resizer:v1.3.0+](https://github.com/kubernetes-csi/external-resizer/releases/tag/v1.3.0)

## Vì sao tôi nên dùng `ReadWriteOncePod`? (Why should I use `ReadWriteOncePod`?)

Trước Kubernetes v1.22, access mode `ReadWriteOnce` thường được dùng để giới hạn quyền truy cập
PersistentVolume cho những workload cần quyền ghi đơn (single-writer) vào lưu trữ. Tuy nhiên,
access mode này có một hạn chế: nó chỉ giới hạn truy cập volume theo từng *node*, cho phép
nhiều Pod trên cùng một node đọc và ghi đồng thời vào cùng một volume. Điều này có thể gây rủi
ro cho những ứng dụng đòi hỏi nghiêm ngặt quyền ghi đơn để bảo đảm an toàn dữ liệu.

Nếu việc bảo đảm quyền ghi đơn là thiết yếu với workload của bạn, hãy cân nhắc di chuyển
(migrate) các volume của bạn sang `ReadWriteOncePod`.

## Di chuyển các PersistentVolume hiện có (Migrating existing PersistentVolumes)

Nếu bạn có các PersistentVolume sẵn có, chúng có thể được di chuyển sang dùng
`ReadWriteOncePod`. Chỉ hỗ trợ di chuyển từ `ReadWriteOnce` sang `ReadWriteOncePod`.

Trong ví dụ này, đã có sẵn một PersistentVolumeClaim "cat-pictures-pvc" dạng `ReadWriteOnce`
được bind với một PersistentVolume "cat-pictures-pv", và một Deployment "cat-pictures-writer"
đang dùng PersistentVolumeClaim này.

> **Ghi chú:** Nếu storage plugin của bạn hỗ trợ
> [cấp phát động (Dynamic provisioning)](98-dynamic-provisioning-vi.md),
> "cat-pictures-pv" sẽ được tạo tự động cho bạn, nhưng tên của nó có thể khác. Để lấy tên
> PersistentVolume của bạn, chạy:
>
> ```shell
> kubectl get pvc cat-pictures-pvc -o jsonpath='{.spec.volumeName}'
> ```

Và bạn có thể xem PVC trước khi thực hiện thay đổi. Hoặc xem manifest ở máy của bạn, hoặc chạy
`kubectl get pvc <tên-pvc> -o yaml`. Kết quả tương tự như:

```yaml
# cat-pictures-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: cat-pictures-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Đây là một Deployment ví dụ dựa trên PersistentVolumeClaim đó:

```yaml
# cat-pictures-writer-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cat-pictures-writer
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cat-pictures-writer
  template:
    metadata:
      labels:
        app: cat-pictures-writer
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        volumeMounts:
        - name: cat-pictures
          mountPath: /mnt
      volumes:
      - name: cat-pictures
        persistentVolumeClaim:
          claimName: cat-pictures-pvc
          readOnly: false
```

Bước đầu tiên, bạn cần sửa `spec.persistentVolumeReclaimPolicy` của PersistentVolume và đặt nó
thành `Retain`. Điều này bảo đảm PersistentVolume của bạn sẽ không bị xóa khi bạn xóa
PersistentVolumeClaim tương ứng:

```shell
kubectl patch pv cat-pictures-pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

Tiếp theo, bạn cần dừng mọi workload đang dùng PersistentVolumeClaim được bind với
PersistentVolume mà bạn muốn di chuyển, rồi xóa PersistentVolumeClaim đó. Tránh thực hiện bất
kỳ thay đổi nào khác trên PersistentVolumeClaim, chẳng hạn thay đổi kích thước (resize) volume,
cho đến khi việc di chuyển hoàn tất.

Khi đã xong bước đó, bạn cần xóa trắng `spec.claimRef.uid` của PersistentVolume để bảo đảm các
PersistentVolumeClaim có thể bind vào nó khi được tạo lại:

```shell
kubectl scale --replicas=0 deployment cat-pictures-writer
kubectl delete pvc cat-pictures-pvc
kubectl patch pv cat-pictures-pv -p '{"spec":{"claimRef":{"uid":""}}}'
```

Sau đó, thay danh sách access mode hợp lệ của PersistentVolume thành (chỉ còn)
`ReadWriteOncePod`:

```shell
kubectl patch pv cat-pictures-pv -p '{"spec":{"accessModes":["ReadWriteOncePod"]}}'
```

> **Ghi chú:** Access mode `ReadWriteOncePod` không thể kết hợp với các access mode khác. Hãy
> bảo đảm `ReadWriteOncePod` là access mode duy nhất trên PersistentVolume khi cập nhật, nếu
> không request sẽ thất bại.

Tiếp theo, bạn cần sửa PersistentVolumeClaim của mình để đặt `ReadWriteOncePod` làm access mode
duy nhất. Bạn cũng nên đặt `spec.volumeName` của PersistentVolumeClaim thành tên
PersistentVolume của bạn để bảo đảm nó bind vào đúng PersistentVolume cụ thể này.

Khi đã xong, bạn có thể tạo lại PersistentVolumeClaim và khởi động workload của mình:

```shell
# QUAN TRỌNG: Nhớ sửa PVC trong cat-pictures-pvc.yaml trước khi apply. Bạn cần:
# - Đặt ReadWriteOncePod làm access mode duy nhất
# - Đặt spec.volumeName thành "cat-pictures-pv"

kubectl apply -f cat-pictures-pvc.yaml
kubectl apply -f cat-pictures-writer-deployment.yaml
```

Cuối cùng, bạn có thể sửa `spec.persistentVolumeReclaimPolicy` của PersistentVolume và đặt lại
về `Delete` nếu trước đó bạn đã đổi nó.

```shell
kubectl patch pv cat-pictures-pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
```

## Tiếp theo (What's next)

* Tìm hiểu thêm về [PersistentVolume](92-persistent-volumes-vi.md).
* Tìm hiểu thêm về [PersistentVolumeClaim](92-persistent-volumes-vi.md#persistentvolumeclaims).
* Tìm hiểu thêm về [Cấu hình một Pod dùng PersistentVolume để lưu trữ](https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở checkpoint giai đoạn 26:

1. Ba replica của Deployment "cat-pictures-writer" cùng mount một PVC `ReadWriteOnce` và cả ba
   được lập lịch lên `lab-k8s-worker1` của bạn. Có gì vi phạm access mode không? Đây chính là rủi
   ro nào mà `ReadWriteOncePod` được sinh ra để bịt?
2. Vì sao bước **đầu tiên** của quy trình di chuyển lại là đổi reclaim policy sang `Retain`,
   trước cả khi đụng tới access mode?
3. Sau khi xóa PVC, vì sao còn phải patch `spec.claimRef.uid` của PV thành chuỗi rỗng?
4. Bạn patch PV thành `{"accessModes":["ReadWriteOncePod","ReadWriteOnce"]}` để "giữ đường
   lui". Kết quả là gì?
5. Khi tạo lại PVC, vì sao trang này khuyên đặt thêm `spec.volumeName` thay vì chỉ đổi access
   mode?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không vi phạm.** `ReadWriteOnce` giới hạn theo **node**, không theo Pod: cả ba Pod nằm
   trên cùng một node nên đều được đọc/ghi đồng thời vào volume. Đây chính là hạn chế bài nêu —
   ứng dụng cần quyền ghi đơn nghiêm ngặt vẫn có thể bị nhiều writer ghi đè lẫn nhau, và
   `ReadWriteOncePod` bịt lỗ hổng này bằng cách siết truy cập xuống đúng một Pod. Trực giác
   "ReadWriteOnce = một Pod" là sai.
2. Vì quy trình **bắt buộc phải xóa PVC** ở bước sau. Nếu reclaim policy của PV đang là
   `Delete` (mặc định của volume cấp phát động), xóa PVC sẽ kéo theo **PV và dữ liệu bị xóa**.
   Đặt `Retain` trước bảo đảm PV sống sót qua việc xóa PVC.
3. Vì PV vẫn còn giữ tham chiếu (`claimRef`) tới PVC cũ, trong đó có `uid` của đối tượng đã bị
   xóa. **Xóa trắng `uid` để PVC được tạo lại — vốn có uid mới — có thể bind vào PV này**; nếu
   không, PV vẫn coi như thuộc về claim cũ.
4. **Request thất bại.** Bài nói rõ `ReadWriteOncePod` không thể kết hợp với bất kỳ access mode
   nào khác; khi cập nhật, nó phải là access mode **duy nhất** trên PV.
5. Để bảo đảm PVC **bind vào đúng PersistentVolume đang giữ dữ liệu cũ**. Không có
   `spec.volumeName`, một cluster có dynamic provisioning hoàn toàn có thể cấp phát một volume
   mới trống rỗng cho PVC đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

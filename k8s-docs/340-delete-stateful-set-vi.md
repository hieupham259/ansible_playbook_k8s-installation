# Xóa một StatefulSet (Delete a StatefulSet)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/delete-stateful-set/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), dòng **Thực
hành**, bài 4/10 · Kiểm chứng ở [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) phần B3.3 —
xóa theo tầng so với `--cascade=orphan`, rồi dọn Pod bằng label; và
[Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) phần B13 nhắc lại đúng điểm mà bài này thêm
vào: Service quản trị **không** bị xóa theo StatefulSet.

Bài về StatefulSet nhưng được xếp ở giai đoạn 5 vì nó chạm tới **headless Service**: mục *Xóa một
StatefulSet* nói bạn có thể phải xóa riêng Service đó sau khi StatefulSet đã biến mất, và chỉ ở
giai đoạn này bạn mới có Service quản trị thật để mà xóa.

**Phải hiểu ở lần đọc này:**

- Mục *Xóa một StatefulSet*: `kubectl delete` theo file hoặc theo tên sẽ **scale StatefulSet xuống
  0** và xóa luôn mọi Pod thuộc workload. `--cascade=orphan` giữ Pod lại sau khi object StatefulSet
  đã mất, và khi đó bạn tự dọn bằng label: `kubectl delete pods -l app.kubernetes.io/name=MyApp`.
- **Headless Service không đi theo.** Bài nói thẳng: "Bạn có thể cần xóa riêng headless service liên
  quan sau khi bản thân StatefulSet đã bị xóa", bằng `kubectl delete service <service-name>`.
- Mục *Persistent Volume*, ý ở mức bạn cần lúc này: **xóa Pod của StatefulSet không xóa volume kèm
  theo**, và bài nêu rõ lý do — để bạn có cơ hội sao chép dữ liệu ra trước khi xóa.
- Mục *Xóa hoàn toàn một StatefulSet* và trật tự của nó: đọc `terminationGracePeriodSeconds` của Pod
  trước, xóa StatefulSet theo label, `sleep $grace` cho Pod kết thúc, **rồi mới** xóa claim. Thứ tự
  này tồn tại để không đụng vào claim khi Pod còn đang dùng nó.
- Mục *Buộc xóa các Pod của StatefulSet*: chỉ đặt ra khi Pod kẹt `Terminating` hoặc `Unknown` lâu;
  bài gọi thẳng đó là "thao tác tiềm ẩn nguy hiểm" và đẩy chi tiết sang bài
  [341](341-force-delete-stateful-set-pod-vi.md).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Phần còn lại của mục *Persistent Volume* — vì sao xóa PVC "có thể kích hoạt việc xóa các Persistent Volume phía sau, tùy thuộc vào storage class và reclaim policy" | chưa học PersistentVolume, PersistentVolumeClaim, StorageClass và reclaim policy | [giai đoạn 6 — Lưu trữ](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài [92](92-persistent-volumes-vi.md) và [96](96-storage-classes-vi.md) |
| Bước `kubectl delete pvc -l app.kubernetes.io/name=MyApp` trong mục *Xóa hoàn toàn một StatefulSet* | StatefulSet bạn đã dựng ở [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) cố ý không sinh PVC nào, nên bước này chưa có gì để xóa — đó là **nợ #2** của lộ trình | [Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md); xem [sổ nợ lộ trình](00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình) |

---

Trang này hướng dẫn bạn cách xóa một StatefulSet.

## Trước khi bạn bắt đầu (Before you begin)

- Trang này giả định rằng bạn có một ứng dụng đang chạy trên cluster, được biểu diễn bởi một StatefulSet.

## Xóa một StatefulSet (Deleting a StatefulSet)

Bạn có thể xóa một StatefulSet theo cùng cách bạn xóa các resource khác trong Kubernetes: dùng lệnh `kubectl delete`, và chỉ định StatefulSet theo file hoặc theo tên.

```shell
kubectl delete -f <file.yaml>
```

```shell
kubectl delete statefulsets <statefulset-name>
```

Bạn có thể cần xóa riêng headless service liên quan sau khi bản thân StatefulSet đã bị xóa.

```shell
kubectl delete service <service-name>
```

Khi xóa một StatefulSet thông qua `kubectl`, StatefulSet được scale xuống 0. Tất cả các Pod thuộc workload này cũng bị xóa. Nếu bạn chỉ muốn xóa StatefulSet mà không xóa các Pod, hãy dùng `--cascade=orphan`. Ví dụ:

```shell
kubectl delete -f <file.yaml> --cascade=orphan
```

Bằng cách truyền `--cascade=orphan` vào `kubectl delete`, các Pod do StatefulSet quản lý được giữ lại ngay cả sau khi bản thân đối tượng StatefulSet đã bị xóa. Nếu các Pod có label `app.kubernetes.io/name=MyApp`, sau đó bạn có thể xóa chúng như sau:

```shell
kubectl delete pods -l app.kubernetes.io/name=MyApp
```

### Persistent Volume (Persistent Volumes)

Việc xóa các Pod trong một StatefulSet sẽ không xóa các volume liên quan. Điều này nhằm đảm bảo rằng bạn có cơ hội sao chép dữ liệu ra khỏi volume trước khi xóa nó. Việc xóa PVC sau khi các Pod đã kết thúc (terminate) có thể kích hoạt việc xóa các Persistent Volume phía sau, tùy thuộc vào storage class và reclaim policy. Bạn không bao giờ nên giả định rằng mình vẫn có khả năng truy cập một volume sau khi claim đã bị xóa.

> **Ghi chú:**
> Hãy thận trọng khi xóa một PVC, vì việc này có thể dẫn tới mất dữ liệu.

### Xóa hoàn toàn một StatefulSet (Complete deletion of a StatefulSet)

Để xóa mọi thứ trong một StatefulSet, bao gồm cả các Pod liên quan, bạn có thể chạy một chuỗi lệnh tương tự như sau:

```shell
grace=$(kubectl get pods <stateful-set-pod> --template '{{.spec.terminationGracePeriodSeconds}}')
kubectl delete statefulset -l app.kubernetes.io/name=MyApp
sleep $grace
kubectl delete pvc -l app.kubernetes.io/name=MyApp

```

Trong ví dụ trên, các Pod có label `app.kubernetes.io/name=MyApp`; hãy thay bằng label của riêng bạn cho phù hợp.

### Buộc xóa các Pod của StatefulSet (Force deletion of StatefulSet pods)

Nếu bạn thấy một số Pod trong StatefulSet của mình bị kẹt ở trạng thái 'Terminating' hoặc 'Unknown' trong một khoảng thời gian dài, bạn có thể cần can thiệp thủ công để buộc xóa (force delete) các Pod khỏi apiserver. Đây là một thao tác tiềm ẩn nguy hiểm. Tham khảo [Buộc xóa các Pod của StatefulSet (Force Delete StatefulSet Pods)](341-force-delete-stateful-set-pod-vi.md) để biết chi tiết.

## Tiếp theo (What's next)

Tìm hiểu thêm về [buộc xóa các Pod của StatefulSet](341-force-delete-stateful-set-pod-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Trên cluster lab, namespace `lab-5a` có StatefulSet `web` với ba Pod. So sánh
   `kubectl delete statefulset web -n lab-5a` với cùng lệnh đó thêm `--cascade=orphan`: Pod đi đâu
   trong mỗi trường hợp, và sau lệnh thứ hai bạn dọn Pod còn lại bằng cách nào?
2. **Câu bẫy.** Cũng trong `lab-5a` còn có headless Service `web-headless` mà StatefulSet dùng làm
   Service quản trị. Bạn xóa StatefulSet `web`. `web-headless` còn không? Vì sao trực giác "xóa cái
   cha thì cái con đi theo" lại sai ở đây?
3. Vì sao chuỗi lệnh ở mục *Xóa hoàn toàn một StatefulSet* phải chèn `sleep $grace` giữa lệnh xóa
   StatefulSet và lệnh xóa claim, thay vì chạy hai lệnh liền nhau?
4. Một Pod của StatefulSet kẹt ở `Terminating` rất lâu trên `lab-k8s-worker2`. Bài cho phép bạn làm
   gì, và kèm theo cảnh báo nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Lệnh thứ nhất: StatefulSet được **scale xuống 0** và **mọi Pod thuộc workload bị xóa theo**. Lệnh
   thứ hai với `--cascade=orphan`: object StatefulSet biến mất nhưng **Pod được giữ lại** — chúng
   thành Pod mồ côi, không còn ai quản lý. Dọn chúng bằng label, đúng như bài viết:
   `kubectl delete pods -l app.kubernetes.io/name=MyApp` (thay bằng label thật của bạn).
2. **Vẫn còn.** Bài nói rõ bạn "có thể cần xóa riêng headless service liên quan **sau khi** bản thân
   StatefulSet đã bị xóa", bằng một lệnh `kubectl delete service <service-name>` riêng. Trực giác sai
   vì Service quản trị **không phải con của StatefulSet**: StatefulSet chỉ tham chiếu tên nó qua
   `serviceName`, còn object Service là thứ bạn tự tạo và tự sở hữu. Cái đi theo StatefulSet là Pod,
   không phải Service.
3. Vì `grace` chính là `terminationGracePeriodSeconds` đọc từ Pod ở lệnh đầu, tức khoảng thời gian
   Pod cần để kết thúc. `sleep $grace` là để **chờ Pod thực sự kết thúc rồi mới xóa claim**. Ăn khớp
   với cảnh báo của mục *Persistent Volume*: xóa claim là thao tác có thể dẫn tới mất dữ liệu, và
   "bạn không bao giờ nên giả định rằng mình vẫn có khả năng truy cập một volume sau khi claim đã bị
   xóa" — nên không được đụng vào claim khi Pod còn đang dùng volume.
4. Bài cho phép **can thiệp thủ công để buộc xóa (force delete) Pod khỏi apiserver**, nhưng chỉ đặt
   ra tình huống này khi Pod kẹt ở `Terminating` hoặc `Unknown` trong một khoảng thời gian dài. Cảnh
   báo kèm theo: **"Đây là một thao tác tiềm ẩn nguy hiểm"**, và bài không hướng dẫn tại chỗ mà đẩy
   sang bài [341 — Buộc xóa các Pod của StatefulSet](341-force-delete-stateful-set-pod-vi.md).

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

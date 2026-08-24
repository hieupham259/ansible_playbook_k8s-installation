# Các hook vòng đời của Container (Container Lifecycle Hooks)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 2](00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime), bài 4/8 ·
Kiểm chứng ở Lab 2 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Lộ trình ghi rõ: `preStop` liên quan trực tiếp đến **shutdown êm** ở giai đoạn 3. Đọc bài này
với ý thức rằng nó là nửa đầu của câu chuyện chấm dứt Pod.

**Phải hiểu ở lần đọc này:**

- Hai hook chính: **`PostStart`** chạy **đồng thời** với ENTRYPOINT — không có gì bảo đảm tiến
  trình chính đã sẵn sàng khi hook chạy; **`PreStop`** chạy **trước** khi tín hiệu TERM được
  gửi.
- Cơ chế thời gian của `PreStop`, và đây là chỗ dễ cấu hình sai nhất:
  `terminationGracePeriodSeconds` **bắt đầu đếm trước khi hook chạy** và bao trùm **cả** thời
  gian hook lẫn thời gian container tự dừng. Hook 55 giây + dừng 10 giây > grace period 60
  giây thì container bị kill giữa chừng.
- Ba loại handler: `exec` (chạy trong container, tài nguyên tính vào container), `httpGet` và
  `sleep` (do kubelet thực thi).
- **Hook thất bại thì container bị kill.**
- Cơ chế gửi là **ít nhất một lần** — hook của bạn phải chịu được việc bị gọi hai lần.
- Debug: log của handler **không** hiện trong event Pod; chỉ có event `FailedPostStartHook`
  hoặc `FailedPreStopHook`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `terminationGracePeriodSeconds` và toàn bộ quy trình chấm dứt Pod | là bài xương sống riêng | giai đoạn 3, bài [47](47-pod-lifecycle-vi.md) |
| Hook `StopSignal` | chi tiết của cùng quy trình chấm dứt | giai đoạn 3 |
| Liveness/startup probe được nhắc trong định nghĩa `PreStop` | chưa học probe | giai đoạn 3, bài [49](49-probes-vi.md) |
| Preemption và tranh chấp tài nguyên | thuộc chủ đề lập lịch | giai đoạn 7 |

---

Trang này mô tả cách các Container do kubelet quản lý có thể sử dụng framework hook
vòng đời của Container để chạy code được kích hoạt bởi các sự kiện trong vòng đời
quản lý của chúng.

## Tổng quan (Overview)

Tương tự như nhiều framework ngôn ngữ lập trình có các hook vòng đời cho thành phần
(component lifecycle hooks), chẳng hạn như Angular, Kubernetes cung cấp cho các
Container các hook vòng đời. Các hook này cho phép Container nhận biết được các sự
kiện trong vòng đời quản lý của chúng và chạy code được hiện thực trong một trình xử
lý (handler) khi hook vòng đời tương ứng được thực thi.

## Các hook của Container (Container hooks) {#container-hooks}

Có hai hook được cung cấp cho các Container:

`PostStart`

Hook này được thực thi ngay sau khi một container được tạo. Nó chạy **đồng thời**
với `ENTRYPOINT` (tiến trình chính) của container, nghĩa là hook có thể chạy trước,
trong hoặc sau khi tiến trình chính khởi động.

Không có tham số nào được truyền cho handler.

> **Ghi chú:** Mặc dù hook chạy đồng thời với tiến trình của container, nó có thể
> làm chậm việc cập nhật trạng thái của container; container có thể không chuyển
> sang trạng thái `Running` cho đến khi hook hoàn tất.

`PreStop`

Hook này được gọi ngay trước khi một container bị chấm dứt (terminate) do một yêu
cầu API hoặc một sự kiện quản lý như liveness/startup probe thất bại, bị chiếm chỗ
(preemption), tranh chấp tài nguyên (resource contention) và các sự kiện khác. Lời
gọi đến hook `PreStop` sẽ thất bại nếu container đã ở trạng thái terminated hoặc
completed, và hook phải hoàn tất trước khi tín hiệu TERM để dừng container có thể
được gửi đi. Quá trình đếm ngược thời gian gia hạn chấm dứt (termination grace
period) của Pod bắt đầu trước khi hook `PreStop` được thực thi, vì vậy bất kể kết
quả của handler ra sao, container cuối cùng vẫn sẽ chấm dứt trong khoảng thời gian
gia hạn chấm dứt của Pod. Không có tham số nào được truyền cho handler.

Bạn có thể xem mô tả chi tiết hơn về hành vi chấm dứt tại
[Chấm dứt Pod (Termination of Pods)](47-pod-lifecycle-vi.md#pod-termination).

`StopSignal`

Hook vòng đời StopSignal có thể được dùng để định nghĩa một tín hiệu dừng (stop
signal) sẽ được gửi tới container khi nó bị dừng. Nếu bạn thiết lập giá trị này, nó
sẽ ghi đè mọi chỉ thị `STOPSIGNAL` được định nghĩa bên trong container image.

Bạn có thể xem mô tả chi tiết hơn về hành vi chấm dứt với tín hiệu dừng tùy chỉnh tại
[Tín hiệu dừng (Stop Signals)](47-pod-lifecycle-vi.md#pod-termination-stop-signals).

### Các cách hiện thực hook handler (Hook handler implementations)

Container có thể truy cập một hook bằng cách hiện thực và đăng ký một handler cho
hook đó. Có ba loại hook handler có thể được hiện thực cho các Container:

* Exec - Thực thi một lệnh cụ thể, chẳng hạn `pre-stop.sh`, bên trong cgroups và
  namespaces của Container. Tài nguyên mà lệnh này tiêu thụ được tính vào Container.
* HTTP - Thực thi một HTTP request tới một endpoint cụ thể trên Container.
* Sleep - Tạm dừng container trong một khoảng thời gian được chỉ định.

### Thực thi hook handler (Hook handler execution)

Khi một hook quản lý vòng đời của Container được gọi, hệ thống quản lý của
Kubernetes sẽ thực thi handler theo hành động của hook: `httpGet`, `tcpSocket`
([đã ngưng sử dụng - deprecated](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.35/#lifecyclehandler-v1-core))
và `sleep` được thực thi bởi tiến trình kubelet, còn `exec` được thực thi trong
container.

Lời gọi handler của hook `PostStart` được khởi tạo khi một container được tạo,
nghĩa là ENTRYPOINT của container và hook `PostStart` được kích hoạt đồng thời.
(Điều này có nghĩa là thường không hợp lý khi dùng một HTTP hook cho `PostStart`,
vì không có gì đảm bảo rằng tiến trình của container đã khởi động hoàn toàn vào
lúc hook chạy.) Nếu hook `PostStart` mất quá nhiều thời gian để thực thi hoặc bị
treo, nó có thể ngăn container chuyển sang trạng thái `running`.

Các hook `PreStop` không được thực thi bất đồng bộ với tín hiệu dừng Container;
hook phải hoàn tất việc thực thi trước khi tín hiệu TERM có thể được gửi. Nếu một
hook `PreStop` bị treo trong khi thực thi, phase của Pod sẽ là `Terminating` và
giữ nguyên như vậy cho đến khi Pod bị kill sau khi `terminationGracePeriodSeconds`
hết hạn. Khoảng thời gian gia hạn này áp dụng cho tổng thời gian cần thiết để cả
hook `PreStop` thực thi xong lẫn Container dừng lại một cách bình thường. Ví dụ,
nếu `terminationGracePeriodSeconds` là 60, hook mất 55 giây để hoàn tất, và
Container mất 10 giây để dừng bình thường sau khi nhận tín hiệu, thì Container sẽ
bị kill trước khi nó kịp dừng bình thường, vì `terminationGracePeriodSeconds` nhỏ
hơn tổng thời gian (55+10) mà hai việc này cần để diễn ra.

Nếu một hook `PostStart` hoặc `PreStop` thất bại, nó sẽ kill Container.

Người dùng nên làm cho các hook handler của mình nhẹ nhất có thể. Tuy nhiên, có
những trường hợp mà các lệnh chạy lâu là hợp lý, chẳng hạn như khi lưu trạng thái
trước khi dừng một Container.

### Đảm bảo việc gửi hook (Hook delivery guarantees)

Việc gửi hook được thiết kế theo cơ chế *ít nhất một lần (at least once)*, nghĩa
là một hook có thể được gọi nhiều lần cho bất kỳ sự kiện nào, chẳng hạn cho
`PostStart` hoặc `PreStop`. Việc xử lý điều này một cách đúng đắn là trách nhiệm
của phần hiện thực hook.

Nói chung, hook chỉ được gửi một lần duy nhất. Ví dụ, nếu bên nhận của một HTTP
hook bị ngừng hoạt động và không thể nhận lưu lượng (traffic), sẽ không có nỗ lực
gửi lại nào. Tuy nhiên, trong một số trường hợp hiếm gặp, việc gửi hai lần có thể
xảy ra. Chẳng hạn, nếu kubelet khởi động lại giữa lúc đang gửi một hook, hook đó
có thể được gửi lại sau khi kubelet hoạt động trở lại.

### Gỡ lỗi các Hook handler (Debugging Hook handlers)

Log của một Hook handler không được hiển thị trong các sự kiện (event) của Pod.
Nếu một handler thất bại vì lý do nào đó, nó sẽ phát ra một event. Với `PostStart`,
đó là event `FailedPostStartHook`, và với `PreStop`, đó là event
`FailedPreStopHook`. Để tự tạo ra một event `FailedPostStartHook`, hãy sửa file
[lifecycle-events.yaml](https://k8s.io/examples/pods/lifecycle-events.yaml)
để đổi lệnh postStart thành "badcommand" và apply nó. Dưới đây là ví dụ output
của các event mà bạn thấy khi chạy `kubectl describe pod lifecycle-demo`:

```
Events:
  Type     Reason               Age              From               Message
  ----     ------               ----             ----               -------
  Normal   Scheduled            7s               default-scheduler  Successfully assigned default/lifecycle-demo to ip-XXX-XXX-XX-XX.us-east-2...
  Normal   Pulled               6s               kubelet            Successfully pulled image "nginx" in 229.604315ms
  Normal   Pulling              4s (x2 over 6s)  kubelet            Pulling image "nginx"
  Normal   Created              4s (x2 over 5s)  kubelet            Created container lifecycle-demo-container
  Normal   Started              4s (x2 over 5s)  kubelet            Started container lifecycle-demo-container
  Warning  FailedPostStartHook  4s (x2 over 5s)  kubelet            Exec lifecycle hook ([badcommand]) for Container "lifecycle-demo-container" in Pod "lifecycle-demo_default(30229739-9651-4e5a-9a32-a8f1688862db)" failed - error: command 'badcommand' exited with 126: , message: "OCI runtime exec failed: exec failed: container_linux.go:380: starting container process caused: exec: \"badcommand\": executable file not found in $PATH: unknown\r\n"
  Normal   Killing              4s (x2 over 5s)  kubelet            FailedPostStartHook
  Normal   Pulled               4s               kubelet            Successfully pulled image "nginx" in 215.66395ms
  Warning  BackOff              2s (x2 over 3s)  kubelet            Back-off restarting failed container
```

## Tiếp theo (What's next)

* Tìm hiểu thêm về [Môi trường của Container (Container environment)](41-container-environment-vi.md).
* Thực hành thực tế với việc
  [gắn handler vào các sự kiện vòng đời của Container](272-attach-handler-lifecycle-event-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. `PostStart` chạy trước hay sau ENTRYPOINT của container? Vì sao bài khuyên không nên dùng
   HTTP handler cho `PostStart`?
2. `terminationGracePeriodSeconds` là 30. Hook `PreStop` của bạn mất 25 giây, và ứng dụng cần
   thêm 10 giây nữa để đóng kết nối cho sạch. Chuyện gì xảy ra?
3. Hook `PreStop` của bạn thất bại. Container ra sao?
4. Hook handler của bạn ghi log ra stdout. Bạn tìm log đó ở đâu, và `kubectl describe pod` cho
   bạn thấy gì khi hook lỗi?
5. Hook được gửi mấy lần? Điều đó ràng buộc gì lên cách bạn viết handler?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Đồng thời.** Hook được kích hoạt khi container được tạo, cùng lúc với ENTRYPOINT, nên nó
   có thể chạy trước, trong hoặc sau khi tiến trình chính khởi động. HTTP handler không hợp lý
   vì **không có gì đảm bảo tiến trình đã lắng nghe cổng** vào lúc hook gọi tới.
2. Container **bị kill trước khi kịp đóng sạch**. Grace period bắt đầu đếm **trước khi** hook
   chạy và bao trùm cả hai giai đoạn: 25 + 10 = 35 giây > 30 giây. Muốn đúng thì phải nâng
   `terminationGracePeriodSeconds` lên trên tổng thời gian đó.
3. **Container bị kill.** Bài nói rõ: nếu hook `PostStart` hoặc `PreStop` thất bại, nó sẽ kill
   container.
4. **Log của hook handler không hiển thị trong event của Pod.** `kubectl describe pod` chỉ cho
   bạn một event `Warning`: `FailedPostStartHook` hoặc `FailedPreStopHook`, kèm thông báo lỗi
   của lệnh.
5. **Ít nhất một lần.** Thông thường đúng một lần, nhưng nếu kubelet khởi động lại giữa lúc gửi
   thì hook có thể chạy hai lần. Ràng buộc: handler phải **idempotent** — chạy lại không được
   gây hại.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

# Lập lịch workload nhận biết topology (Topology-Aware Workload Scheduling)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/topology-aware-scheduling/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 14/15 · Kiểm chứng ở [Lab 13](labs/LAB-13-DRA.md).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Bài này ở trạng thái `alpha` từ v1.36 — **tên plugin và cấu hình có thể
đổi giữa các phiên bản**. Cluster lab 3 VM trên VMware **chỉ có một miền topology** (không
rack, không zone), nên việc chấm điểm giữa nhiều placement ứng viên không có gì để so. Đọc để
hiểu khái niệm.

Đây là mặt **thuật toán** của cặp bài cùng tên; mặt **API** — bạn khai `schedulingConstraints`
thế nào — đã đọc ở [bài 80](80-workload-topology-scheduling-vi.md). Bài này lấp ba pha còn
trống của thuật toán lập lịch theo placement ở
[bài 151](151-podgroup-scheduling-vi.md) bằng những plugin cụ thể.

**Phải hiểu ở lần đọc này:**

- TAS ở đây là **một thuật toán lập lịch theo placement**, tìm placement tối ưu cho PodGroup
  sao cho **tất cả Pod nằm cùng một miền topology**.
- Ba plugin và vai trò của từng cái: `TopologyPlacement` sinh placement ứng viên bằng cách
  **gom node theo các giá trị khác nhau của topology `key`** được yêu cầu; `NodeResourcesFit`
  chấm điểm placement theo **tỉ lệ cấp phát trên tất cả node trong placement**;
  `PodGroupPodsCount` chấm điểm theo **tổng số Pod của nhóm lập lịch thành công được**.
- Hai interface mà chúng cài đặt: `TopologyPlacement` là `PlacementGeneratePlugin`, còn hai
  plugin kia là `PlacementScorePlugin`.
- Mặc định `NodeResourcesFit` và `PodGroupPodsCount` **cùng trọng số 1**, để cân bằng giữa
  bin-packing và việc lập lịch được càng nhiều Pod càng tốt.
- Ranh giới trong cấu hình: chấm điểm placement **luôn dùng chiến lược `MostAllocated`**;
  trường `type` của `scoringStrategy` **chỉ được xem xét trong lập lịch theo từng Pod**, trong
  khi **trọng số tài nguyên áp dụng cho cả hai** thuật toán.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Đoạn `KubeSchedulerConfiguration` chỉnh trọng số | cần cluster nhiều miền topology mới có ý nghĩa | khi công việc thực sự cần |
| Bin-packing và ý nghĩa của `MostAllocated` / `LeastAllocated` | nền đã học | giai đoạn 7 — [bài 148](148-resource-bin-packing-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

*Lập lịch nhận biết topology* (Topology-Aware Scheduling — TAS) là một [thuật toán lập lịch theo placement](151-podgroup-scheduling-vi.md#placement-scheduling-algorithm)
cho phép tìm vị trí sắp đặt (placement) tối ưu cho PodGroup đang được xem xét, đảm bảo rằng tất cả các pod
sẽ được đặt cùng nhau (collocated) trong cùng một miền topology (topology domain). Người dùng có thể điều chỉnh TAS
theo nhu cầu cụ thể của mình bằng cách thay đổi cấu hình các plugin TAS.

## Scheduling framework: cấu hình các plugin TAS (Scheduling framework: TAS plugins configuration)

Scheduler bao gồm các plugin in-tree mới và được mở rộng, triển khai các điểm mở rộng TAS:

* `TopologyPlacement`: Triển khai interface `PlacementGeneratePlugin`. Nó sinh ra các placement
  ứng viên bằng cách gom nhóm các node dựa trên các giá trị khác nhau của topology `key` được yêu cầu (định nghĩa
  trong PodGroup).

* `NodeResourcesFit`: Được mở rộng để triển khai interface `PlacementScorePlugin`. Theo
  logic tương tự như bin-packing tiêu chuẩn cho pod, nó chấm điểm các placement dựa trên tỉ lệ cấp phát (allocation ratio)
  trên tất cả các node trong placement. Nó dùng chiến lược `MostAllocated` để tối đa hóa mức sử dụng
  tài nguyên trong một placement, và kế thừa các trọng số tài nguyên từ cấu hình plugin tiêu chuẩn
  theo từng pod.

* `PodGroupPodsCount`: Triển khai interface `PlacementScorePlugin`. Nó chấm điểm các placement
  ứng viên dựa trên tổng số pod trong PodGroup mà bạn có thể lập lịch thành công.

### Tùy chỉnh trọng số plugin và trọng số tài nguyên bin-packing (Customizing plugin weights and bin-packing resource weights)

Theo mặc định, các plugin `NodeResourcesFit` và `PodGroupPodsCount` được cấu hình với trọng số
bằng nhau (cả hai mặc định là 1) để duy trì sự cân bằng tốt giữa logic bin-packing và việc lập lịch
càng nhiều pod càng tốt.

Bạn có thể điều chỉnh các trọng số này, hoặc các trọng số tài nguyên trong chiến lược bin-packing, trong
KubeSchedulerConfiguration của bạn. Dưới đây là một đoạn ví dụ cho thấy cách thay đổi trọng số cho cả hai
plugin, và cách ghi đè các trọng số tài nguyên của `NodeResourcesFit`. Thay đổi thứ hai sẽ áp dụng
cho cả thuật toán chấm điểm theo từng pod lẫn thuật toán chấm điểm placement:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
    plugins:
      placementScore:
        enabled:
          # 1) Thay đổi trọng số mặc định của các plugin chấm điểm placement
          - name: NodeResourcesFit
            weight: 2
          - name: PodGroupPodsCount
            weight: 5
    pluginConfig:
      - name: NodeResourcesFit
        args:
          # 2) Thay đổi trọng số tài nguyên chấm điểm cho cả thuật toán theo từng pod
          # lẫn thuật toán chấm điểm placement
          scoringStrategy:
            # Trường type chỉ được xem xét trong lập lịch theo từng pod. Chấm điểm placement
            # luôn dùng chiến lược MostAllocated
            type: LeastAllocated
            # Trọng số tài nguyên sẽ được dùng trong cả thuật toán theo từng pod lẫn thuật toán chấm điểm placement
            resources:
              - name: cpu
                weight: 2
              - name: memory
                weight: 3
```

## Tiếp theo (What's next)

* Tìm hiểu thêm về [API lập lịch nhận biết topology](80-workload-topology-scheduling-vi.md).
* Đọc về [lập lịch pod group](151-podgroup-scheduling-vi.md).
* Đọc về [các chính sách pod group](79-workload-policies-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Ba plugin TAS cài đặt hai interface nào, và mỗi plugin trả lời câu hỏi gì trong quá trình
   chọn placement?
2. Bạn đặt `scoringStrategy.type: LeastAllocated` trong cấu hình `NodeResourcesFit`. Việc chấm
   điểm placement có chuyển sang `LeastAllocated` không? Phần nào của khối `scoringStrategy`
   thực sự ảnh hưởng tới cả hai thuật toán?
3. Vì sao mặc định `NodeResourcesFit` và `PodGroupPodsCount` có trọng số bằng nhau — đánh đổi
   ở đây là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **`TopologyPlacement` cài `PlacementGeneratePlugin`** — nó trả lời "có những miền nào đáng
   xét", bằng cách gom nhóm các node theo các giá trị khác nhau của topology `key` được yêu cầu
   trong PodGroup. **`NodeResourcesFit` và `PodGroupPodsCount` cài `PlacementScorePlugin`** —
   chúng trả lời "miền nào tốt hơn": cái trước chấm theo **tỉ lệ cấp phát trên tất cả node
   trong placement** (bin-packing), cái sau chấm theo **số Pod của nhóm lập lịch thành công
   được**.
2. **Không.** Bài ghi rõ ngay trong chú thích của ví dụ: **chấm điểm placement luôn dùng chiến
   lược `MostAllocated`**, còn trường `type` **chỉ được xem xét trong lập lịch theo từng Pod**.
   Thứ thực sự dùng chung là **danh sách `resources` với trọng số tài nguyên** — đổi trọng số
   `cpu`/`memory` ở đây thì cả thuật toán chấm điểm theo từng Pod lẫn thuật toán chấm điểm
   placement đều chịu ảnh hưởng. Đây là bẫy dễ dính vì hai thiết lập nằm trong cùng một khối
   `scoringStrategy`.
3. Vì hai plugin đó kéo về hai hướng khác nhau: `NodeResourcesFit` đẩy về phía **dồn chặt tài
   nguyên** (bin-packing, `MostAllocated`), còn `PodGroupPodsCount` đẩy về phía **lập lịch được
   càng nhiều Pod càng tốt**. Trọng số mặc định bằng nhau (cả hai là 1) là để **duy trì sự cân
   bằng tốt giữa hai mục tiêu đó**. Nếu bạn ưu tiên chạy được nhiều Pod hơn là gọn tài nguyên,
   nâng trọng số `PodGroupPodsCount` lên, và ngược lại.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

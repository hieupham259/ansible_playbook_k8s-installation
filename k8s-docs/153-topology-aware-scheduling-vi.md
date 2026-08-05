# Lập lịch workload nhận biết topology (Topology-Aware Workload Scheduling)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/topology-aware-scheduling/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

*Lập lịch nhận biết topology* (Topology-Aware Scheduling — TAS) là một [thuật toán lập lịch theo placement](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/#placement-scheduling-algorithm)
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

* Tìm hiểu thêm về [API lập lịch nhận biết topology](https://kubernetes.io/docs/concepts/workloads/workload-api/topology-aware-scheduling/).
* Đọc về [lập lịch pod group](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/).
* Đọc về [các chính sách pod group](https://kubernetes.io/docs/concepts/workloads/workload-api/policies/).

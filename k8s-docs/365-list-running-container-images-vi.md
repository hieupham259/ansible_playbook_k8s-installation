# Liệt kê tất cả Container image đang chạy trong Cluster (List All Container Images Running in a Cluster)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/access-application-cluster/list-all-running-container-images/

Trang này hướng dẫn cách dùng kubectl để liệt kê tất cả các Container image
của các Pod đang chạy trong một cluster.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

Trong bài thực hành này, bạn sẽ dùng kubectl để lấy tất cả các Pod
đang chạy trong một cluster, rồi định dạng đầu ra để rút ra danh sách
các Container của từng Pod.

## Liệt kê tất cả Container image trong mọi namespace (List all Container images in all namespaces)

- Lấy tất cả các Pod trong mọi namespace bằng `kubectl get pods --all-namespaces`
- Định dạng đầu ra để chỉ bao gồm danh sách tên Container image
  bằng `-o jsonpath={.items[*].spec['initContainers', 'containers'][*].image}`. Cách này sẽ
  phân giải đệ quy trường `image` từ JSON trả về.
  - Xem [tài liệu tham khảo jsonpath](https://kubernetes.io/docs/reference/kubectl/jsonpath/)
    để biết thêm thông tin về cách dùng jsonpath.
- Định dạng đầu ra bằng các công cụ tiêu chuẩn: `tr`, `sort`, `uniq`
  - Dùng `tr` để thay khoảng trắng bằng ký tự xuống dòng
  - Dùng `sort` để sắp xếp kết quả
  - Dùng `uniq` để gom và đếm số lượng image

```shell
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec['initContainers', 'containers'][*].image}" |\
tr -s '[[:space:]]' '\n' |\
sort |\
uniq -c
```
Biểu thức jsonpath được diễn giải như sau:

- `.items[*]`: với mỗi giá trị trả về
- `.spec`: lấy phần spec
- `['initContainers', 'containers'][*]`: với mỗi container
- `.image`: lấy image

> **Ghi chú:**
> Khi lấy một Pod đơn lẻ theo tên, ví dụ `kubectl get pod nginx`,
> phần `.items[*]` của biểu thức phải được bỏ đi vì kết quả trả về là một
> Pod đơn lẻ chứ không phải một danh sách các phần tử.

## Liệt kê Container image theo từng Pod (List Container images by Pod)

Bạn có thể kiểm soát định dạng chi tiết hơn bằng cách dùng phép toán `range` để
lặp qua từng phần tử một.

```shell
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}' |\
sort
```

## Liệt kê Container image có lọc theo label của Pod (List Container images filtering by Pod label)

Để chỉ nhắm tới các Pod khớp với một label cụ thể, hãy dùng cờ -l. Lệnh
sau chỉ khớp các Pod có label khớp với `app=nginx`.

```shell
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" -l app=nginx
```

## Liệt kê Container image có lọc theo namespace của Pod (List Container images filtering by Pod namespace)

Để chỉ nhắm tới các Pod trong một namespace cụ thể, hãy dùng cờ namespace. Lệnh
sau chỉ khớp các Pod trong namespace `kube-system`.

```shell
kubectl get pods --namespace kube-system -o jsonpath="{.items[*].spec.containers[*].image}"
```

## Liệt kê Container image dùng go-template thay cho jsonpath (List Container images using a go-template instead of jsonpath)

Thay cho jsonpath, Kubectl hỗ trợ dùng [go-templates](https://pkg.go.dev/text/template)
để định dạng đầu ra:

```shell
kubectl get pods --all-namespaces -o go-template --template="{{range .items}}{{range .spec.containers}}{{.image}} {{end}}{{end}}"
```

## Tiếp theo (What's next)

### Tài liệu tham khảo (Reference)

* Tài liệu tham khảo [Jsonpath](https://kubernetes.io/docs/reference/kubectl/jsonpath/)
* Tài liệu tham khảo [Go template](https://pkg.go.dev/text/template)

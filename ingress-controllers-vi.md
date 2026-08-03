# Ingress Controllers

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/>
>
> Để một [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) hoạt động được trong cluster của bạn,
> phải có một _ingress controller_ đang chạy.
> Bạn cần chọn ít nhất một ingress controller và đảm bảo nó đã được thiết lập trong cluster của bạn.
> Trang này liệt kê các ingress controller phổ biến mà bạn có thể triển khai.

> **Ghi chú:**
>
> Dự án Kubernetes khuyến nghị sử dụng [Gateway](https://gateway-api.sigs.k8s.io/) thay cho
> [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/).
> Ingress API đã bị đóng băng (frozen).
>
> Điều này có nghĩa là:
> * Ingress API đã ở trạng thái generally available (GA), và tuân theo các [đảm bảo về tính ổn định](https://kubernetes.io/docs/reference/using-api/deprecation-policy/#deprecating-parts-of-the-api) dành cho các API đã GA.
>   Dự án Kubernetes không có kế hoạch loại bỏ Ingress khỏi Kubernetes.
> * Ingress API sẽ không được phát triển thêm nữa, và sẽ không có bất kỳ thay đổi
>   hay cập nhật nào tiếp theo.

## Các ingress controller (Ingress controllers)

Kubernetes, với tư cách là một dự án, hỗ trợ và bảo trì các ingress controller [AWS](https://github.com/kubernetes-sigs/aws-load-balancer-controller#readme) và [GCE](https://git.k8s.io/ingress-gce/README.md#readme).

## Các ingress controller của bên thứ ba (Third party ingress controllers)

> **Ghi chú:**
>
> Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án đó, vốn được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi. [Thông tin thêm.](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/#third-party-content-disclaimer)

* [AKS Application Gateway Ingress Controller](https://docs.microsoft.com/azure/application-gateway/tutorial-ingress-controller-add-on-existing?toc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Faks%2Ftoc.json&bc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fbread%2Ftoc.json) là một ingress controller cấu hình [Azure Application Gateway](https://docs.microsoft.com/azure/application-gateway/overview).
* [Alibaba Cloud API Gateway Ingress](https://www.alibabacloud.com/help/en/api-gateway/cloud-native-api-gateway/user-guide/ingress-managementapig-ngress-management) là một ingress controller cấu hình [Alibaba Cloud Native API Gateway](https://www.alibabacloud.com/help/en/api-gateway/cloud-native-api-gateway/product-overview/what-is-cloud-native-api-gateway), đồng thời cũng là phiên bản thương mại của [Higress](https://github.com/alibaba/higress).
* [Apache APISIX ingress controller](https://github.com/apache/apisix-ingress-controller) là một ingress controller dựa trên [Apache APISIX](https://github.com/apache/apisix).
* [Avi Kubernetes Operator](https://github.com/vmware/load-balancer-and-ingress-services-for-kubernetes) cung cấp cân bằng tải L4-L7 sử dụng [VMware NSX Advanced Load Balancer](https://avinetworks.com/).
* [BFE Ingress Controller](https://github.com/bfenetworks/ingress-bfe) là một ingress controller dựa trên [BFE](https://www.bfe-networks.net).
* [BunkerWeb Ingress Controller](https://docs.bunkerweb.io/latest/integrations/#kubernetes) là một ingress controller cho [BunkerWeb](https://www.bunkerweb.io/), một WAF (Web Application Firewall - tường lửa ứng dụng web) dựa trên nginx.
* [Cilium Ingress Controller](https://docs.cilium.io/en/stable/network/servicemesh/ingress/) là một ingress controller được vận hành bởi [Cilium](https://cilium.io/).
* [Citrix ingress controller](https://github.com/citrix/citrix-k8s-ingress-controller#readme) hoạt động với
  Citrix Application Delivery Controller.
* [Contour](https://projectcontour.io/) là một ingress controller dựa trên [Envoy](https://www.envoyproxy.io/).
* [Emissary-Ingress](https://www.getambassador.io/products/api-gateway) API Gateway là một ingress
  controller dựa trên [Envoy](https://www.envoyproxy.io).
* [EnRoute](https://getenroute.io/) là một API gateway dựa trên [Envoy](https://www.envoyproxy.io) có thể chạy như một ingress controller.
* F5 BIG-IP [Container Ingress Services for Kubernetes](https://clouddocs.f5.com/containers/latest/userguide/kubernetes/)
  cho phép bạn dùng một Ingress để cấu hình các virtual server F5 BIG-IP.
* [FortiADC Ingress Controller](https://docs.fortinet.com/document/fortiadc/7.0.0/fortiadc-ingress-controller/742835/fortiadc-ingress-controller-overview) hỗ trợ tài nguyên Ingress của Kubernetes và cho phép bạn quản lý các đối tượng FortiADC từ Kubernetes.
* [Gloo](https://gloo.solo.io) là một ingress controller mã nguồn mở dựa trên [Envoy](https://www.envoyproxy.io),
  cung cấp chức năng API gateway.
* [HAProxy Ingress](https://haproxy-ingress.github.io/) là một ingress controller cho
  [HAProxy](https://www.haproxy.org/#desc).
* [Higress](https://github.com/alibaba/higress) là một API gateway dựa trên [Envoy](https://www.envoyproxy.io) có thể chạy như một ingress controller.
* [HAProxy Ingress Controller for Kubernetes](https://github.com/haproxytech/kubernetes-ingress#readme)
  cũng là một ingress controller cho [HAProxy](https://www.haproxy.org/#desc).
* [Istio Ingress](https://istio.io/latest/docs/tasks/traffic-management/ingress/kubernetes-ingress/)
  là một ingress controller dựa trên [Istio](https://istio.io/).
* [Kong Ingress Controller for Kubernetes](https://github.com/Kong/kubernetes-ingress-controller#readme)
  là một ingress controller điều khiển [Kong Gateway](https://konghq.com/kong/).
* [Kusk Gateway](https://kusk.kubeshop.io/) là một ingress controller vận hành theo OpenAPI (OpenAPI-driven), dựa trên [Envoy](https://www.envoyproxy.io).
* [NGINX Ingress Controller for Kubernetes](https://www.nginx.com/products/nginx-ingress-controller/)
  hoạt động với webserver [NGINX](https://www.nginx.com/resources/glossary/nginx/) (đóng vai trò proxy).
* [ngrok-operator](https://github.com/ngrok/ngrok-operator) là một controller cho [ngrok](https://ngrok.com/), hỗ trợ cả Ingress và Gateway API để bổ sung truy cập công khai an toàn cho các Service K8s của bạn.
* [OCI Native Ingress Controller](https://github.com/oracle/oci-native-ingress-controller#readme) là một Ingress controller cho Oracle Cloud Infrastructure, cho phép bạn quản lý [OCI Load Balancer](https://docs.oracle.com/en-us/iaas/Content/Balance/home.htm).
* [OpenNJet Ingress Controller](https://gitee.com/njet-rd/open-njet-kic) là một ingress controller dựa trên [OpenNJet](https://njet.org.cn/).
* [Pomerium Ingress Controller](https://www.pomerium.com/docs/k8s/ingress.html) dựa trên [Pomerium](https://pomerium.com/), cung cấp chính sách truy cập nhận biết ngữ cảnh (context-aware access policy).
* [Skipper](https://opensource.zalando.com/skipper/kubernetes/ingress-controller/) là HTTP router và reverse proxy dùng để tổ hợp dịch vụ (service composition), bao gồm các trường hợp sử dụng như Kubernetes Ingress, được thiết kế như một thư viện để bạn xây dựng proxy tùy chỉnh của riêng mình.
* [Traefik Kubernetes Ingress provider](https://doc.traefik.io/traefik/providers/kubernetes-ingress/) là một
  ingress controller cho proxy [Traefik](https://traefik.io/traefik/).
* [Tyk Operator](https://github.com/TykTechnologies/tyk-operator) mở rộng Ingress bằng các Custom Resource để mang khả năng Quản lý API (API Management) đến Ingress. Tyk Operator hoạt động với Tyk Gateway mã nguồn mở và control plane Tyk Cloud.
* [Voyager](https://voyagermesh.com) là một ingress controller cho
  [HAProxy](https://www.haproxy.org/#desc).
* [Wallarm Ingress Controller](https://www.wallarm.com/solutions/waf-for-kubernetes) là một Ingress Controller cung cấp các khả năng WAAP (WAF) và Bảo mật API (API Security).

## Sử dụng nhiều Ingress controller (Using multiple Ingress controllers)

Bạn có thể triển khai bất kỳ số lượng ingress controller nào bằng cách dùng [ingress class](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class)
trong một cluster. Hãy ghi nhớ `.metadata.name` của tài nguyên ingress class của bạn. Khi tạo một ingress, bạn sẽ cần tên đó để chỉ định trường `ingressClassName` trên đối tượng Ingress của bạn (tham khảo [tài liệu IngressSpec v1](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/ingress-v1/#IngressSpec)). `ingressClassName` là sự thay thế cho [phương pháp dùng annotation](https://kubernetes.io/docs/concepts/services-networking/ingress/#deprecated-annotation) cũ hơn.

Nếu bạn không chỉ định IngressClass cho một Ingress, và cluster của bạn có đúng một IngressClass được đánh dấu là mặc định, thì Kubernetes sẽ [áp dụng](https://kubernetes.io/docs/concepts/services-networking/ingress/#default-ingress-class) IngressClass mặc định của cluster cho Ingress đó.
Bạn đánh dấu một IngressClass là mặc định bằng cách thiết lập [annotation `ingressclass.kubernetes.io/is-default-class`](https://kubernetes.io/docs/reference/labels-annotations-taints/#ingressclass-kubernetes-io-is-default-class) trên IngressClass đó, với giá trị chuỗi là `"true"`.

Lý tưởng nhất, mọi ingress controller đều nên đáp ứng đặc tả này, nhưng các ingress
controller khác nhau hoạt động hơi khác nhau một chút.

> **Ghi chú:**
>
> Hãy chắc chắn rằng bạn đã đọc kỹ tài liệu của ingress controller mà bạn chọn để hiểu những lưu ý (caveat) khi sử dụng nó.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/).

# Xác minh các artifact Kubernetes đã ký (Verify Signed Kubernetes Artifacts)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [beta]`

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần cài đặt sẵn các công cụ sau:

- `cosign` ([hướng dẫn cài đặt](https://docs.sigstore.dev/cosign/system_config/installation/))
- `curl` (thường được cung cấp sẵn bởi hệ điều hành của bạn)
- `jq` ([tải jq](https://jqlang.github.io/jq/download/))

## Xác minh chữ ký của các file nhị phân (Verifying binary signatures)

Quy trình phát hành (release) của Kubernetes ký tất cả các artifact nhị phân (các file tarball,
file SPDX, các file nhị phân độc lập) bằng cách sử dụng cơ chế ký không cần khóa (keyless
signing) của cosign. Để xác minh một file nhị phân cụ thể, hãy tải nó về cùng với chữ ký
(signature) và certificate của nó:

```bash
URL=https://dl.k8s.io/release/v1.36.0/bin/linux/amd64
BINARY=kubectl

FILES=(
    "$BINARY"
    "$BINARY.sig"
    "$BINARY.cert"
)

for FILE in "${FILES[@]}"; do
    curl -sSfL --retry 3 --retry-delay 3 "$URL/$FILE" -o "$FILE"
done
```

Sau đó xác minh blob bằng lệnh `cosign verify-blob`:

```shell
cosign verify-blob "$BINARY" \
  --signature "$BINARY".sig \
  --certificate "$BINARY".cert \
  --certificate-identity krel-staging@k8s-releng-prod.iam.gserviceaccount.com \
  --certificate-oidc-issuer https://accounts.google.com
```

> **Ghi chú:**
> Cosign 2.0 yêu cầu các tùy chọn `--certificate-identity` và `--certificate-oidc-issuer`.
>
> Để tìm hiểu thêm về ký không cần khóa, hãy tham khảo [Keyless Signatures](https://docs.sigstore.dev/cosign/signing/overview/).
>
> Các phiên bản Cosign trước đây yêu cầu bạn đặt biến `COSIGN_EXPERIMENTAL=1`.
>
> Để biết thêm thông tin, hãy tham khảo [blog sigstore](https://blog.sigstore.dev/cosign-2-0-released/)

## Xác minh chữ ký của các image (Verifying image signatures)

Để xem danh sách đầy đủ các image được ký, hãy tham khảo
trang [Releases](https://kubernetes.io/releases/download/).

Chọn một image từ danh sách này và xác minh chữ ký của nó bằng
lệnh `cosign verify`:

```shell
cosign verify registry.k8s.io/kube-apiserver-amd64:v1.36.0 \
  --certificate-identity krel-trust@k8s-releng-prod.iam.gserviceaccount.com \
  --certificate-oidc-issuer https://accounts.google.com \
  | jq .
```

### Xác minh image của tất cả các thành phần control plane (Verifying images for all control plane components)

Để xác minh tất cả các image control plane đã ký của phiên bản stable mới nhất
(v1.36.0), hãy chạy các lệnh sau:

```shell
curl -Ls "https://sbom.k8s.io/$(curl -Ls https://dl.k8s.io/release/stable.txt)/release" \
  | grep "SPDXID: SPDXRef-Package-registry.k8s.io" \
  | grep -v sha256 | cut -d- -f3- | sed 's/-/\//' | sed 's/-v1/:v1/' \
  | sort > images.txt
input=images.txt
while IFS= read -r image
do
  cosign verify "$image" \
    --certificate-identity krel-trust@k8s-releng-prod.iam.gserviceaccount.com \
    --certificate-oidc-issuer https://accounts.google.com \
    | jq .
done < "$input"
```

Sau khi đã xác minh một image, bạn có thể chỉ định image đó bằng digest của nó trong các
manifest Pod của bạn, như ví dụ sau:

```console
registry-url/image-name@sha256:45b23dee08af5e43a7fea6c4cf9c25ccf269ee113168c19722f87876677c5cb2
```

Để biết thêm thông tin, hãy tham khảo mục
[Chính sách pull image (Image Pull Policy)](40-images-vi.md#chính-sách-pull-image-image-pull-policy).

## Xác minh chữ ký image bằng Admission Controller (Verifying Image Signatures with Admission Controller)

Đối với các image không thuộc control plane (ví dụ
[image conformance](https://github.com/kubernetes/kubernetes/blob/master/test/conformance/image/README.md)),
chữ ký cũng có thể được xác minh tại thời điểm triển khai (deploy) bằng admission controller
[sigstore policy-controller](https://docs.sigstore.dev/policy-controller/overview).

Dưới đây là một số tài nguyên hữu ích để bắt đầu với `policy-controller`:

- [Cài đặt](https://github.com/sigstore/helm-charts/tree/main/charts/policy-controller)
- [Các tùy chọn cấu hình](https://github.com/sigstore/policy-controller/tree/main/config)

## Xác minh Software Bill Of Materials (Verify the Software Bill Of Materials)

Bạn có thể xác minh Software Bill of Materials (SBOM) của Kubernetes bằng
certificate và chữ ký sigstore, hoặc bằng các file SHA tương ứng:

```shell
# Lấy phiên bản phát hành Kubernetes mới nhất hiện có
VERSION=$(curl -Ls https://dl.k8s.io/release/stable.txt)

# Xác minh tổng SHA512
curl -Ls "https://sbom.k8s.io/$VERSION/release" -o "$VERSION.spdx"
echo "$(curl -Ls "https://sbom.k8s.io/$VERSION/release.sha512") $VERSION.spdx" | sha512sum --check

# Xác minh tổng SHA256
echo "$(curl -Ls "https://sbom.k8s.io/$VERSION/release.sha256") $VERSION.spdx" | sha256sum --check

# Lấy chữ ký và certificate sigstore
curl -Ls "https://sbom.k8s.io/$VERSION/release.sig" -o "$VERSION.spdx.sig"
curl -Ls "https://sbom.k8s.io/$VERSION/release.cert" -o "$VERSION.spdx.cert"

# Xác minh chữ ký sigstore
cosign verify-blob \
    --certificate "$VERSION.spdx.cert" \
    --signature "$VERSION.spdx.sig" \
    --certificate-identity krel-staging@k8s-releng-prod.iam.gserviceaccount.com \
    --certificate-oidc-issuer https://accounts.google.com \
    "$VERSION.spdx"
```

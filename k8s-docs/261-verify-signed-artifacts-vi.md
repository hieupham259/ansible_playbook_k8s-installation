# Xác minh các artifact Kubernetes đã ký (Verify Signed Kubernetes Artifacts)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 22 — Audit và mã hóa dữ liệu](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu),
bài 6/6 — **bài cuối giai đoạn** · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 22](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu).

**Nói thẳng về công cụ:** bài này cần `cosign` (và `jq`). Cả hai **không có trong baseline** —
không nằm ở [bảng A1.3](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) cũng như
[bảng A1.4 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) — và
**lộ trình không cài thêm phần mềm**. Vì vậy phần lớn bài này là **đọc để nắm quy trình**, không
phải để gõ theo. Phần duy nhất chạy được bằng công cụ sẵn có là **kiểm tra tổng SHA256/SHA512 của
SBOM** bằng `curl` và `sha256sum`/`sha512sum`.

**Phải hiểu ở lần đọc này:**

- Quy trình phát hành Kubernetes ký **toàn bộ artifact nhị phân** — tarball, file SPDX, các file
  nhị phân độc lập — bằng **cơ chế ký không cần khóa (keyless signing)** của cosign. Không có khóa
  công khai cố định để mang theo; thứ chứng minh danh tính là **certificate cộng OIDC issuer**.
- Hệ quả trực tiếp, nêu trong ô *Ghi chú*: `cosign` từ bản 2.0 **bắt buộc** hai tùy chọn
  `--certificate-identity` và `--certificate-oidc-issuer`. Bài dùng **hai danh tính khác nhau**:
  `krel-staging@…` cho blob và SBOM, `krel-trust@…` cho image.
- Bộ ba file cần tải khi xác minh một binary: **chính file đó, file `.sig` và file `.cert`**, rồi
  chạy `cosign verify-blob`. Với image thì `cosign verify <image>` làm việc thẳng trên registry,
  không phải tải trước.
- Điều phải làm **sau khi** xác minh xong, ở cuối mục *Xác minh image của tất cả các thành phần
  control plane*: chỉ định image trong manifest Pod **bằng digest** dạng
  `registry-url/image-name@sha256:…`, không bằng tag — nối tiếp mục
  [Chính sách pull image](40-images-vi.md#chính-sách-pull-image-image-pull-policy).
- Hai đường xác minh nữa: với image **không thuộc control plane**, chữ ký có thể được xác minh
  **ngay lúc triển khai** bằng admission controller `sigstore policy-controller`; còn **SBOM** thì
  xác minh được **hoặc** bằng certificate và chữ ký sigstore, **hoặc** bằng các file SHA tương ứng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mọi lệnh `cosign verify` và `cosign verify-blob`, cùng đoạn cài `cosign`/`jq` ở mục *Trước khi bạn bắt đầu* | hai công cụ này không có trong [A1.3](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) hay [A1.4 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00), và lộ trình không cài thêm phần mềm | không có bước nào trong lộ trình cài `cosign`; phần chạy được ngay trên `lab-k8s-master` chỉ là kiểm tra tổng SHA của SBOM ở mục *Xác minh Software Bill Of Materials* |
| Mục *Xác minh chữ ký image bằng Admission Controller* — `sigstore policy-controller` | là add-on bên thứ ba cài bằng Helm chart, ngoài stack khóa của [A1.4](labs/LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) | chặng admission đã học ở [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy); việc rà quyền của tích hợp bên thứ ba nằm ở bài [256](256-securing-a-cluster-vi.md) vừa đọc |
| Vòng lặp shell dựng `images.txt` từ SBOM (`grep`/`cut`/`sed`) | là thao tác gom danh sách bằng shell, không phải cơ chế xác minh | không cần cho Checkpoint giai đoạn 22; cơ chế cần nắm nằm ở hai lệnh `cosign` phía trên nó |
| Các số phiên bản và URL cụ thể trong ví dụ | số phiên bản của lộ trình chỉ giữ ở một chỗ | [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) |

---

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

## Xác minh chữ ký image bằng Admission Controller (Verifying Image Signatures with Admission Controller) {#verifying-image-signatures-with-admission-controller}

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 22:

1. "Ký không cần khóa (keyless signing)" nghĩa là gì ở đây, và vì sao chính vì thế mà lệnh
   `cosign verify-blob` bắt buộc phải kèm `--certificate-identity` và
   `--certificate-oidc-issuer`?
2. Muốn xác minh một file nhị phân của bản phát hành, bạn phải tải về mấy file và là những file
   nào? Bài dùng hai danh tính certificate khác nhau — mỗi danh tính dùng cho loại artifact nào?
3. **Câu bẫy.** Bạn vừa `cosign verify` thành công một image control plane theo tag. Ghi đúng cái
   tag đó vào manifest Pod đã đủ bảo đảm thứ được kéo về đúng là image bạn vừa xác minh chưa? Bài
   bảo làm thế nào?
4. Trên `lab-k8s-master` không có `cosign` và lộ trình không cài thêm phần mềm. Phần nào của bài
   bạn vẫn chạy được bằng công cụ sẵn có, và bài xếp phần đó là đường xác minh thứ mấy trong hai
   đường dành cho SBOM?
5. Với các image **không thuộc** control plane, bài đề xuất xác minh chữ ký ở thời điểm nào và
   bằng thành phần gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nghĩa là quy trình phát hành **không dùng một cặp khóa cố định** để ký; **certificate cộng với
   OIDC issuer** mới là thứ chứng minh ai đã ký. Vì không có khóa công khai để đối chiếu, người
   xác minh **phải tự khai báo danh tính mình kỳ vọng** — đó chính là
   `--certificate-identity` và `--certificate-oidc-issuer`, hai tùy chọn mà cosign 2.0 bắt buộc.
   Thiếu chúng thì lệnh không biết chữ ký hợp lệ đó là **của ai**.
2. **Ba file: chính file nhị phân, file `.sig` và file `.cert`.** Hai danh tính:
   **`krel-staging@k8s-releng-prod.iam.gserviceaccount.com`** dùng cho **blob** — file nhị phân và
   SBOM; **`krel-trust@k8s-releng-prod.iam.gserviceaccount.com`** dùng cho **image**. Cả hai đều
   đi với OIDC issuer `https://accounts.google.com`.
3. **Chưa đủ.** Tag có thể được trỏ sang một image khác sau khi bạn xác minh xong, nên cái được
   kéo về chưa chắc là cái đã kiểm. Bài nói rõ: sau khi xác minh, **chỉ định image bằng digest**
   trong manifest Pod — dạng `registry-url/image-name@sha256:…` — vì digest gắn chặt vào đúng nội
   dung đã được xác minh.
4. Chạy được **mục *Xác minh Software Bill Of Materials*, nhánh kiểm tra tổng SHA**: tải file
   `.spdx` bằng `curl` rồi đối chiếu bằng `sha512sum --check` / `sha256sum --check`. Đó là **đường
   thứ hai** trong hai đường bài nêu cho SBOM — đường thứ nhất là certificate và chữ ký sigstore,
   và đường đó **cần `cosign`**, thứ không có trong baseline.
5. Xác minh **tại thời điểm triển khai (deploy)**, bằng admission controller
   **`sigstore policy-controller`** — tức là chặn ngay khi Pod được tạo, thay vì kiểm thủ công
   trước đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là **bài cuối của giai đoạn 22**:
tiếp theo hãy làm **Checkpoint** ghi ở cuối
[mục giai đoạn 22](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) trên cluster lab —
bật audit log với một policy tối thiểu rồi tìm lại được chính request của mình trong log; bật mã
hóa Secret at rest, tạo một Secret mới, rồi chứng minh bằng `etcdctl get` rằng giá trị trong etcd
không còn đọc được dưới dạng thường. Đây cũng là chỗ **trả nợ #6**, nên **đọc lại bài
[109](109-secret-vi.md) trước khi làm**, và làm theo bài [208](208-encrypt-data-vi.md).

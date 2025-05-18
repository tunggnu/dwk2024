---
path: "/part-3/2-deployment-pipeline-vi"
title: "Quy trình Triển khai (Deployment Pipeline)"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn có thể

- Tạo quy trình triển khai của riêng bạn lên GKE với Github Actions và kích hoạt triển khai liên tục từ một lần đẩy mã (git push) lên môi trường sản xuất

</text-box>

Hãy thiết lập một quy trình triển khai sử dụng GitHub Actions. Chúng ta chỉ cần một thứ gì đó để triển khai nên hãy tạo một website mới.

Tạo một Dockerfile với nội dung sau:

```Dockerfile
FROM nginx:1.19-alpine

COPY index.html /usr/share/nginx/html
```

và thêm file index.html với nội dung sau

```html
<!DOCTYPE html>
<html>
  <body style="background-color: gray;">
    <p>
      Nội dung
    </p>
  </body>
</html>
```

Hãy đảm bảo mọi thứ hoạt động bằng cách thực hiện `docker build . -t colorcontent && docker run -p 3000:80 colorcontent` để build và chạy nó, sau đó truy cập [http://localhost:3000](http://localhost:3000). Tiếp theo là thêm các manifest cho website của chúng ta.

**manifests/service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: dwk-environments-svc
spec:
  type: LoadBalancer
  selector:
    app: dwk-environments
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
```

**manifests/deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dwk-environments
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dwk-environments
  template:
    metadata:
      labels:
        app: dwk-environments
    spec:
      containers:
        - name: dwk-environments
          image: jakousa/colorcontent
```

Tiếp theo, để kiểm tra manifest, hãy triển khai nó vào cụm của chúng ta. Ở trên tôi đã đẩy image đã build bằng `docker push`.

```console
$ kubectl apply -f manifests/service.yaml
$ kubectl apply -f manifests/deployment.yaml
```

### Kustomize

Áp dụng nhiều file như vậy khá phiền phức. Chúng ta có thể chỉ định một thư mục như sau: `kubectl apply -f manifests/`, nhưng đây là thời điểm tuyệt vời để chú ý đến Kustomize.

[Kustomize](https://github.com/kubernetes-sigs/kustomize) là một công cụ giúp tùy biến cấu hình và đã được tích hợp sẵn trong kubectl. Trong trường hợp này, chúng ta sẽ dùng nó để xác định những file nào là quan trọng với Kubernetes. Ngoài ra, chúng ta có thể dùng Helm và có thể là [Helmsman](https://github.com/Praqma/helmsman), nhưng sẽ không đề cập trong phạm vi khóa học này.

Với chúng ta, một file mới _kustomization.yaml_ ở thư mục gốc của dự án là đủ. File _kustomization.yaml_ nên bao gồm hướng dẫn sử dụng deployment.yaml và service.yaml.

**kustomization.yaml**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - manifests/deployment.yaml
  - manifests/service.yaml
```

Bây giờ chúng ta có thể triển khai bằng cờ `-k` để chỉ định sử dụng Kustomize.

```console
$ kubectl apply -k .
```

Chúng ta có thể xem trước file với `kubectl kustomize .`. Kustomize sẽ là công cụ thiết yếu cho quy trình triển khai của chúng ta. Nó cho phép chúng ta chọn riêng lẻ image nào sẽ sử dụng. Để làm điều này, hãy khai báo image trong kustomization.yaml.

**kustomization.yaml**

```yaml
# ...
images:
  - name: PROJECT/IMAGE
    newName: jakousa/colorcontent
```

Điều này sẽ thay thế image "IMAGE:TAG" bằng image được định nghĩa trong _newName_. Tiếp theo, đặt một giá trị placeholder trong deployment.yaml cho image:

**deployment.yaml**

```yaml
# ...
containers:
  - name: dwk-environments
    image: PROJECT/IMAGE
```

Kiểm tra mọi thứ hoạt động

```console
$ kubectl kustomize .
  ...
    spec:
      containers:
      - image: jakousa/colorcontent
        name: dwk-environments
```

Kustomize còn có một số công cụ bổ sung bạn có thể thử nếu muốn cài đặt - nhưng chúng ta sẽ thấy cách sử dụng ở phần tiếp theo.

Tài liệu của Kustomize có thể không phải là tốt nhất. Bạn sẽ tìm thấy nhiều thông tin hữu ích hơn qua Google. Một tài liệu bạn nên xem là [Kustomize Cheat Sheet](https://itnext.io/kubernetes-kustomize-cheat-sheet-8e2d31b74d8f).

### Github Actions

[GitHub Actions](https://github.com/features/actions) sẽ là công cụ CI/CD được lựa chọn cho khóa học này. Google cũng cung cấp [Cloud Build](https://cloud.google.com/cloud-build), và một [hướng dẫn từng bước triển khai lên GKE](https://cloud.google.com/cloud-build/docs/deploying-builds/deploy-gke) với nó. Bạn có thể quay lại đây để triển khai với Cloud Build nếu còn dư tín dụng sau khóa học!

Tạo file .github/workflows/main.yaml. Chúng ta muốn workflow thực hiện 3 việc:

- build image
- publish image lên container registry
- triển khai image mới lên cụm của chúng ta

Cấu hình ban đầu sẽ như sau:

**main.yaml**

```yaml
name: Release application

on:
  push:

env:
  PROJECT_ID: ${{ secrets.GKE_PROJECT }}
  GKE_CLUSTER: dwk-cluster
  GKE_ZONE: europe-north1-b
  IMAGE: dwk-environments
  SERVICE: dwk-environments
  BRANCH: ${{ github.ref_name }}
```

Chúng ta thiết lập workflow chạy mỗi khi có thay đổi được đẩy lên repository và đặt các biến môi trường phù hợp - chúng ta sẽ cần chúng ở các bước sau.

Tiếp theo là thêm các job. Để đơn giản, chúng ta sẽ thêm mọi thứ vào một job duy nhất để build, publish và deploy.

```yaml
# ...
jobs:
  build-publish-deploy:
    name: Build, Publish and Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4
```

Điều này thiết lập môi trường cho job và kích hoạt [checkout action](https://github.com/actions/checkout) ở bước đầu tiên.

Tiếp theo, chúng ta sẽ sử dụng một số action bổ sung, chủ yếu từ [google-github-actions](https://github.com/google-github-actions) được thiết kế để hỗ trợ triển khai lên Google Cloud. Bắt đầu với [authentication](https://github.com/google-github-actions/auth), tiếp theo là [setup](https://github.com/google-github-actions/setup-gcloud):

```yaml
# ...
- uses: google-github-actions/auth@v2
  with:
    credentials_json: "${{ secrets.GKE_SA_KEY }}"

- name: "Set up Cloud SDK"
  uses: google-github-actions/setup-gcloud@v2

- name: "Use gcloud CLI"
  run: gcloud info
```

Các secrets dùng cho xác thực **không** lấy từ biến môi trường mà được thêm vào secrets của dự án trên GitHub:

<img src="../img/ghasecret.png">

Đọc thêm [tại đây](https://docs.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets) về GitHub secrets.

GKE_SA_KEY là <i>khóa tài khoản dịch vụ</i> cần thiết để truy cập dịch vụ Google Cloud - đọc hướng dẫn [tại đây](https://cloud.google.com/iam/docs/creating-managing-service-account-keys). Bạn sẽ cần tạo một tài khoản dịch vụ mới và lấy khóa của nó.

Cấp các quyền sau cho tài khoản dịch vụ:

- **Kubernetes Engine Service Agent** - Cho phép tài khoản Kubernetes Engine quản lý tài nguyên cụm. Bao gồm quyền truy cập tài khoản dịch vụ.
- **Storage Admin** - Toàn quyền kiểm soát bucket và object.
- **Artifact Registry Administrator** - Quản trị viên tạo và quản lý repository.
- **Artifact Registry Create-on-Push Repository Administrator** - Quản lý artifact trong repository, cũng như tạo repository mới khi push.

Sau khi tạo tài khoản dịch vụ cho GKE tên "github-actions", tôi tạo khóa bằng gcloud:

```console
$ gcloud iam service-accounts keys create ./private-key.json --iam-account=github-actions@dwk-gke-331210.iam.gserviceaccount.com
```

Toàn bộ file JSON sinh ra cần được thêm vào GKE_SA_KEY.

Tiếp theo, dùng lệnh _gcloud_ để cấu hình Docker.

```yaml
# ...
- run: gcloud --quiet auth configure-docker
```

Điều này cho phép chúng ta push image lên Google Container Registry, thay vì Docker Hub. Chúng ta có thể dùng Docker Hub nếu muốn, nhưng GCR là lựa chọn tuyệt vời khi đã có quyền truy cập. GCR hiệu năng cao và độ trễ mạng thấp. Giảm thời gian di chuyển image sẽ giúp triển khai nhanh hơn. Đọc thêm tại <https://cloud.google.com/container-registry/>. Lưu ý registry này [không miễn phí](https://cloud.google.com/container-registry/pricing) và bạn nên xóa image trong và sau khóa học.

Còn một bước nữa là sử dụng action
[get-gke-credentials](https://github.com/google-github-actions/get-gke-credentials) để lấy thông tin xác thực cụm Google Kubernetes:

```yaml
# ...
- name: "Get GKE credentials"
  uses: "google-github-actions/get-gke-credentials@v2"
  with:
    cluster_name: "${{ env.GKE_CLUSTER }}"
    project_id: "${{ env.PROJECT_ID }}"
    location: "${{ env.GKE_ZONE }}"
```

Và cuối cùng, hãy ghi ra image mong muốn với tag. Image sẽ là `gcr.io/PROJECT_ID/IMAGE:GITHUB_BRANCH-GITHUB_SHA`. Và build image:

```yaml
# ...
- name: Build
  run: docker build --tag "gcr.io/$PROJECT_ID/$IMAGE:$BRANCH-$GITHUB_SHA" .
```

Chúng ta dùng tên dự án (từ env _\$IMAGE_) làm tên image và tag là tên nhánh cộng với commit _sha_ từ env _\$GITHUB_SHA_ do workflow cung cấp tự động.

Publish tương tự:

```yaml
# ...
- name: Publish
  run: docker push "gcr.io/$PROJECT_ID/$IMAGE:$BRANCH-$GITHUB_SHA"
```

Bước cuối là triển khai. Đầu tiên thiết lập Kustomize:

```yaml
# ...
- name: Set up Kustomize
  uses: imranismail/setup-kustomize@v2.1.0
```

Bây giờ chúng ta có thể dùng Kustomize để đặt image mà pipeline sẽ publish. Ở đây tôi pipe output của `kustomize build .` vào `kubectl apply`, nếu bạn không chắc chuyện gì đang xảy ra, có thể xuất `kustomize build .` ra để kiểm tra giữa pipeline!

Cuối cùng, chúng ta sẽ xem trước [rollout](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/) và xác nhận release thành công. Rollout sẽ chờ đến khi deployment hoàn tất. Ở đây chúng ta dùng cùng tên image, dwk-environments, làm tên deployment nên có thể dùng biến môi trường \$IMAGE.

```yaml
# ...
- name: Deploy
  run: |-
    kustomize edit set image PROJECT/IMAGE=gcr.io/$PROJECT_ID/$IMAGE:$BRANCH-$GITHUB_SHA
    kustomize build . | kubectl apply -f -
    kubectl rollout status deployment $SERVICE
    kubectl get services -o wide
```

<exercise name='Bài tập 3.03: Dự án v1.4'>

Thiết lập triển khai tự động cho dự án.

Gợi ý:

- Nếu pod của bạn dùng Persistent Volume Claim với access mode [ReadWriteOnce](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes), bạn có thể cần cân nhắc [chiến lược triển khai](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy), vì mặc định (RollingUpdate) có thể gây lỗi. Đọc thêm trong [tài liệu](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy). Lựa chọn khác là dùng [access mode](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) cho phép nhiều pod mount volume.
- Nếu bạn dùng Ingress, nhớ rằng nó mong đợi service trả về phản hồi thành công tại đường dẫn / dù service được ánh xạ tới đường dẫn khác!

</exercise>

### Môi trường riêng cho mỗi nhánh

Một lựa chọn phổ biến khi dùng pipeline triển khai là có môi trường riêng cho mỗi nhánh - đặc biệt khi dùng feature branching.

Hãy triển khai phiên bản của riêng chúng ta. Hãy mở rộng pipeline đã định nghĩa trước đó. Trạng thái hiện tại của pipeline như sau:

**main.yaml**

```yaml
name: Release application

on:
  push:

env:
  PROJECT_ID: ${{ secrets.GKE_PROJECT }}
  GKE_CLUSTER: dwk-cluster
  GKE_ZONE: europe-north1-b
  IMAGE: dwk-environments
  DEPLOYMENT: dwk-environments
  BRANCH: ${{ github.ref_name }}

jobs:
  build-publish-deploy:
    name: Build, Publish and Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - uses: google-github-actions/auth@v2
        with:
          credentials_json: "${{ secrets.GKE_SA_KEY }}"

      - name: "Set up Cloud SDK"
        uses: google-github-actions/setup-gcloud@v2

      - name: "Use gcloud CLI"
        run: gcloud info

      - run: gcloud --quiet auth configure-docker

      - name: "Get GKE credentials"
        uses: "google-github-actions/get-gke-credentials@v2"
        with:
          cluster_name: "${{ env.GKE_CLUSTER }}"
          project_id: "${{ env.PROJECT_ID }}"
          location: "${{ env.GKE_ZONE }}"

      - name: Build and publish
        run: |-
          docker build --tag "gcr.io/$PROJECT_ID/$IMAGE:$BRANCH-$GITHUB_SHA" .
          docker push "gcr.io/$PROJECT_ID/$IMAGE:$BRANCH-$GITHUB_SHA"

      - name: Set up Kustomize
        uses: imranismail/setup-kustomize@v2

      - name: Deploy
        run: |-
          kustomize edit set image PROJECT/IMAGE=gcr.io/$PROJECT_ID/$IMAGE:$BRANCH-$GITHUB_SHA
          kustomize build . | kubectl apply -f -
          kubectl rollout status deployment $DEPLOYMENT
          kubectl get services -o wide
```

Chúng ta muốn triển khai mỗi nhánh vào một namespace riêng để mỗi nhánh có môi trường riêng biệt.

Namespace có thể thay đổi với kustomize:

```console
kustomize edit set namespace ${GITHUB_REF#refs/heads/}
```

Với lệnh này, tên namespace sẽ bằng tên nhánh.

Nếu namespace chưa tồn tại, lệnh sẽ báo lỗi, nên cần tạo namespace:

```console
kubectl create namespace ${GITHUB_REF#refs/heads/} || true
```

Chúng ta cũng cần đặt namespace cho context:

```console
kubectl config set-context --current --namespace=${GITHUB_REF#refs/heads/}
```

Vậy bước deploy sẽ thay đổi như sau:

```yaml
- name: Deploy
  run: |-
    kubectl create namespace ${GITHUB_REF#refs/heads/} || true
    kubectl config set-context --current --namespace=${GITHUB_REF#refs/heads/}
    kustomize edit set namespace ${GITHUB_REF#refs/heads/}
    kustomize edit set image PROJECT/IMAGE=gcr.io/$PROJECT_ID/$IMAGE:${GITHUB_REF#refs/heads/}-$GITHUB_SHA
    kustomize build . | kubectl apply -f -
    kubectl rollout status deployment $DEPLOYMENT
    kubectl get services -o wide
```

Để kiểm tra, hãy sửa index.html và đẩy thay đổi lên một nhánh mới.

Bước tiếp theo là cấu hình tên miền cho từng nhánh để có "www.example.com" cho production và ví dụ "feat_x.example.com" cho nhánh feat_x. Nếu còn dư tín dụng sau khóa học, bạn có thể quay lại đây để triển khai. Google Cloud DNS và [hướng dẫn này](https://cloud.google.com/kubernetes-engine/docs/tutorials/configuring-domain-name-static-ip) sẽ giúp bạn bắt đầu.

<exercise name='Bài tập 3.04: Dự án v1.4.1'>

Cải tiến quy trình triển khai để mỗi nhánh tạo một môi trường riêng. Nhánh main vẫn phải được triển khai ở namespace _default_.

</exercise>

<exercise name='Bài tập 3.05: Dự án v1.4.2'>

Cuối cùng, hãy tạo workflow mới để khi xóa một nhánh thì môi trường tương ứng cũng bị xóa.

</exercise>

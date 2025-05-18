---
path: "/part-4/3-gitops-vi"
title: "Triển khai GitOps"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này, bạn có thể

- so sánh GitOps với các phương pháp triển khai khác

- so sánh triển khai kiểu push truyền thống với triển khai kiểu pull

- triển khai GitOps trong cụm Kubernetes của bạn

</text-box>

Một pipeline triển khai đơn giản mà chúng ta đã sử dụng và học qua thường như sau:

1. Lập trình viên thực hiện git push với mã đã chỉnh sửa, ví dụ lên GitHub
2. Điều này kích hoạt một dịch vụ CI/CD bắt đầu chạy, ví dụ GitHub Actions
3. Dịch vụ CI/CD chạy kiểm thử, build image, đẩy image lên registry và triển khai image mới, ví dụ lên Kubernetes

Cách này gọi là triển khai kiểu push. Tên gọi này rất mô tả vì mọi thứ được đẩy đi bởi bước trước đó. Tuy nhiên, có một số thách thức với cách tiếp cận push. Ví dụ, nếu chúng ta có một cụm Kubernetes không cho phép kết nối từ bên ngoài (ví dụ cụm trên máy cá nhân hoặc bất kỳ cụm nào không muốn cho người ngoài truy cập), khi đó CI/CD không thể push cập nhật vào cụm.

Với cấu hình pull, mọi thứ được đảo ngược. Chúng ta có thể để cụm, chạy ở bất cứ đâu, **kéo** image mới và tự động triển khai nó. Image mới vẫn được kiểm thử và build bởi CI/CD. Chúng ta chỉ đơn giản là chuyển phần triển khai từ CI/CD sang một hệ thống khác thực hiện việc pull.

<text-box name="Watchtower" variant="hint">

Nếu bạn đã hoàn thành khóa DevOps với Docker, có thể bạn đã nghe về [watchtower](https://github.com/containrrr/watchtower), công cụ cho phép chuyển các bước cuối của pipeline triển khai từ push sang pull khi chạy dịch vụ với docker-compose.

</text-box>

GitOps chính là về sự đảo ngược này và thúc đẩy các thực tiễn tốt cho vận hành. Điều này đạt được bằng cách để trạng thái của cụm nằm trong một repository git. Như vậy ngoài việc xử lý triển khai ứng dụng, nó còn xử lý mọi thay đổi của cụm. Điều này đòi hỏi cấu hình bổ sung và thay đổi tư duy so với truyền thống cấu hình máy chủ. Nhưng khi đã làm được, GitOps sẽ là dấu chấm hết cho quản lý cụm kiểu mệnh lệnh.

[ArgoCD](https://argo-cd.readthedocs.io/en/stable/) là công cụ được lựa chọn. Cuối cùng, quy trình làm việc của chúng ta sẽ như sau:

1. Lập trình viên git push mã hoặc cấu hình đã chỉnh sửa.
2. Dịch vụ CI/CD (ở đây là GitHub Actions) bắt đầu chạy.
3. CI/CD build và đẩy image mới _và_ commit chỉnh sửa vào nhánh "release" (ở đây là main)
4. ArgoCD sẽ lấy trạng thái mô tả trong nhánh release và áp dụng cho cụm.

Bắt đầu bằng việc cài đặt ArgoCD theo [Getting started](https://argo-cd.readthedocs.io/en/stable/getting_started/) trong tài liệu:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Giờ ArgoCD đã chạy trong cụm. Chúng ta cần mở quyền truy cập cho nó. Có nhiều [tùy chọn](https://argo-cd.readthedocs.io/en/stable/getting_started/#3-access-the-argo-cd-api-server). Ở đây sẽ dùng LoadBalancer:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Sau một lúc, cụm sẽ cung cấp external IP:

```bash
$ kubectl get svc -n argocd
NAME               TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
...
argocd-server      LoadBalancer   10.7.5.82     34.88.152.2   80:32029/TCP,443:30574/TCP   17min
```

Mật khẩu ban đầu cho tài khoản _admin_ được tạo tự động và lưu dưới dạng văn bản thuần trong trường password của secret _argocd-initial-admin-secret_ trong namespace cài đặt ArgoCD. Lấy nó bằng cách giải mã base64 giá trị lấy được từ lệnh:

```bash
kubectl get -n argocd secrets argocd-initial-admin-secret -o yaml
```

Bây giờ có thể đăng nhập:

<img src="../img/argo2.png">

Hãy triển khai ứng dụng đơn giản tại <https://github.com/mluukkai/dwk-app1> bằng ArgoCD. Bắt đầu bằng cách nhấn _New app_ và điền form mở ra:

<img src="../img/argo3.png">

Ban đầu, chúng ta chọn _manual sync policy_.

<img src="../img/argo4.png">

Lưu ý vì file định nghĩa (ở đây là _kustomization.yaml_) nằm ở thư mục gốc repo, nên _path_ được đặt là ., tức là dấu chấm.

Ứng dụng được tạo nhưng đang thiếu và chưa đồng bộ, vì ta chọn manual sync:

<img src="../img/argo5.png">

Hãy đồng bộ và vào trang app:

<img src="../img/argo6.png">

Có vẻ có gì đó sai, pod có biểu tượng _broken heart_. Có thể bắt đầu debug như thường lệ với `kubectl get po`. Thông tin tương tự cũng xem được từ ArgoCD khi nhấn vào pod:

<img src="../img/argo7.png">

Chúng ta đã chỉ định một image không tồn tại. Hãy sửa trên GitHub bằng cách đổi _kustomization.yaml_ như sau:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - manifests/deployment.yaml
  - manifests/service.yaml
images:
  - name: PROJECT/IMAGE
    newName: mluukkai/dwk1
```

Khi đồng bộ lại trên ArgoCD, một pod mới khỏe mạnh sẽ được tạo và app đã chạy!

Có thể kiểm tra external IP bằng _kubectl_ hoặc nhấn vào service:

<img src="../img/argo8.png">

Bây giờ hãy đổi chế độ đồng bộ sang _automatic_ bằng cách nhấn _Details_ trên trang app trong ArgoCD.

Từ giờ mọi thay đổi cấu hình trên GitHub sẽ được ArgoCD tự động áp dụng. Hãy thử scale deployment lên 5 pod:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dwk1-app-dep
spec:
  replicas: 5
  selector:
    matchLabels:
      app: dwk1-app
  template: ...
```

Sau một lúc (chu kỳ đồng bộ mặc định của ArgoCD là 180 giây), app sẽ được đồng bộ và có 5 pod chạy:

<img src="../img/argo9.png">

Ngoài service, deployment và pod, cây cấu hình app còn hiển thị các đối tượng khác. Ta thấy có [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) (ký hiệu _rs_) nằm giữa deployment và pod. ReplicaSet là đối tượng Kubernetes đảm bảo luôn có số lượng pod ổn định cho một workload. ReplicaSet định nghĩa số pod giống hệt cần chạy, nếu pod bị loại bỏ hoặc lỗi sẽ tạo thêm pod bù vào. Người dùng thường không định nghĩa ReplicaSet trực tiếp mà dùng Deployment, Deployment sẽ tạo ReplicaSet để quản lý pod.

Các thay đổi cấu hình Kubernetes của app trên GitHub giờ đã được đồng bộ đẹp đẽ vào cụm. Vậy còn thay đổi mã nguồn app thì sao?

Để triển khai phiên bản app mới, cần đổi image dùng trong deployment. Hiện image được định nghĩa trong _kustomization.yaml_:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - manifests/deployment.yaml
  - manifests/service.yaml
images:
  - name: PROJECT/IMAGE
    newName: mluukkai/dwk1
```

Vậy để triển khai app mới, cần:

1. tạo image mới với tag mới (nếu có)
2. đổi kustomization.yaml để dùng image mới
3. push thay đổi lên GitHub

Có thể làm thủ công nhưng như vậy là chưa đủ. Hãy định nghĩa workflow GitHub Actions tự động hóa các bước này.

Bước đầu đã quen thuộc từ [phần 3](/part-3/2-deployment-pipeline)

Bước 2 có thể làm với lệnh _kustomize edit_:

```yaml
$ kustomize edit set image PROJECT/IMAGE=node:20
$ cat kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- manifests/deployment.yaml
- manifests/service.yaml
images:
- name: PROJECT/IMAGE
  newName: node
  newTag: "20"
```

Như thấy, lệnh này _thay đổi_ file _kustomization.yaml_. Việc còn lại là commit file vào repo. May mắn là đã có GitHub Action [add-and-commit](https://github.com/EndBug/add-and-commit) cho việc này!

Workflow như sau:

```yaml
name: Build and publish application

on:
  push:

jobs:
  build-publish:
    name: Build, Push, Release
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # tag image với GitHub SHA để có tag duy nhất
      - name: Build and publish backend
        run: |-
          docker build --tag "mluukkai/dwk1:$GITHUB_SHA" .
          docker push "mluukkai/dwk1:$GITHUB_SHA"

      - name: Set up Kustomize
        uses: imranismail/setup-kustomize@v2

      - name: Use right image
        run: kustomize edit set image PROJECT/IMAGE=mluukkai/dwk1:$GITHUB_SHA

      - name: commit kustomization.yaml to GitHub
        uses: EndBug/add-and-commit@v9
        with:
          add: "kustomization.yaml"
          message: New version released ${{ github.sha }}
```

Còn một việc nữa, trong cài đặt repo cần cấp quyền ghi cho GitHub Actions:

<img src="../img/gha-perm.png">

Giờ mọi thay đổi cấu hình và mã nguồn app đều được ArgoCD tự động triển khai nhờ workflow GitHub Action, thật tuyệt!

<exercise name='Bài tập 4.07: GitOps cho Project'>

Chuyển ứng dụng _Ping-pong_ sang dùng GitOps để mỗi khi bạn commit lên repo, ứng dụng sẽ tự động được cập nhật.

</exercise>

Nếu một app có nhiều môi trường như production, staging, testing, cách định nghĩa môi trường đơn giản sẽ dẫn đến nhiều copy-paste. Khi dùng Kustomize, [khuyến nghị](https://kubectl.docs.kubernetes.io/guides/example/multi_base/) là định nghĩa một _base_ chung cho mọi môi trường và các overlay riêng cho từng môi trường, nơi chỉ định phần khác biệt.

Ví dụ đơn giản như sau. Tạo cấu trúc thư mục:

```
.
├── base
│   ├── deployment.yaml
│   └── kustomization.yaml
└── overlays
    ├── prod
    │   ├── deployment.yaml
    │   └── kustomization.yaml
    └── staging
        └── kustomization.yaml
```

**base/deployment.yaml** như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: PROJECT/IMAGE
```

**base/kustomization.yaml** rất đơn giản:

```yaml
resources:
  - deployment.yaml
```

Môi trường production sửa base bằng cách đổi số replica thành 3. **overlays/prod/deployment.yaml** như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dep
spec:
  replicas: 3
```

Chỉ cần định nghĩa phần khác biệt.

**overlays/prod/kustomization.yaml** như sau:

```yaml
resources:
  - ./../../base
patches:
  - path: deployment.yaml

namePrefix: prod-

images:
  - name: PROJECT/IMAGE
    newName: nginx:1.25-bookworm
```

Nó tham chiếu base và [patches](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/) deployment để đặt số replica thành 3.

**overlays/staging/kustomization.yaml** chỉ đặt image:

```yaml
resources:
  - ./../../base
namePrefix: staging-

images:
  - name: PROJECT/IMAGE
    newName: nginx:1.25.5-alpine
```

Giờ production và staging có thể triển khai bằng ArgoCD bằng cách trỏ đến thư mục overlay tương ứng.

Chúng ta đã dùng giao diện ArgoCD để định nghĩa ứng dụng. Ngoài ra còn các cách khác. Có thể [cài đặt](https://argo-cd.readthedocs.io/en/stable/getting_started/#2-download-argo-cd-cli) và dùng [Argo CLI](https://argo-cd.readthedocs.io/en/stable/getting_started/#creating-apps-via-cli)

Một lựa chọn nữa là dùng cách [khai báo](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#declarative-setup) với [Application](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications) Custom Resource Definition:

Với phiên bản production, định nghĩa như sau:

**application.yaml**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mluukkai/gitopstest
    path: overlays/prod
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Khai báo application cho ArgoCD bằng lệnh:

```bash
kubectl apply -n argocd -f application.yaml
```

Với GitOps, chúng ta đạt được:

- Bảo mật tốt hơn

  - Không ai cần truy cập cụm, kể cả dịch vụ CI/CD. Không cần chia sẻ quyền truy cập cụm cho cộng tác viên; họ chỉ cần commit như mọi người khác.

- Minh bạch hơn

  - Mọi thứ đều được khai báo trong repository GitHub. Khi có người mới vào nhóm, chỉ cần xem repo; không cần truyền đạt kiến thức cũ hay mẹo ẩn nào cả.

- Truy vết tốt hơn

  - Mọi thay đổi với cụm đều được kiểm soát phiên bản. Bạn sẽ biết chính xác trạng thái cụm và ai đã thay đổi, thay đổi như thế nào.

- Giảm rủi ro

  - Nếu có sự cố, chỉ cần revert cụm về commit hoạt động. `git revert` và toàn bộ cụm trở lại trạng thái trước đó.

- Dễ di chuyển
  - Muốn đổi nhà cung cấp? Tạo cụm mới và trỏ về cùng repo - cụm của bạn đã ở đó.

Có một số lựa chọn cho setup GitOps. Ở đây chúng ta để cấu hình ứng dụng cùng repo với mã nguồn. Một lựa chọn khác là tách cấu hình khỏi mã nguồn. ArgoCD liệt kê nhiều lý do thuyết phục rằng tách repo là [best practice](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/#separating-config-vs-source-code-repositories).

<exercise name='Bài tập 4.08: GitOps cho Project'>

Chuyển Project sang dùng GitOps.

- Tạo hai môi trường riêng biệt, _production_ và _staging_ ở namespace riêng
- Mỗi commit lên nhánh main sẽ triển khai lên môi trường staging
- Mỗi commit _tagged_ sẽ triển khai lên môi trường production
- Ở staging, broadcaster chỉ log các tin nhắn, không chuyển tiếp ra ngoài
- Ở staging, database không cần backup
- Bạn có thể giả định _secrets_ đã được apply sẵn ngoài ArgoCD
- Bonus: dùng repo riêng biệt cho code và cấu hình

</exercise>

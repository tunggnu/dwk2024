---
path: "/part-5/2-custom-resource-definitions-vi"
title: "Định nghĩa Tài nguyên Tùy chỉnh (Custom Resource Definitions)"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này, bạn có thể

- Tạo Định nghĩa Tài nguyên Tùy chỉnh (Custom Resource Definitions) của riêng mình

</text-box>

[Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) (CRDs) là cách để mở rộng Kubernetes với các tài nguyên (Resource) của riêng chúng ta. Chúng ta đã sử dụng rất nhiều CRD rồi, ví dụ ở [phần 4](/part-4/3-gitops) chúng ta đã dùng [Application](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications) với ArgoCD.

Chúng là một phần không thể thiếu khi sử dụng Kubernetes nên rất đáng để học cách tự tạo một CRD.

Trước khi bắt đầu, chúng ta cần xác định mình muốn tạo gì. Hãy tạo một resource có thể dùng để tạo các bộ đếm ngược (countdown). Resource này sẽ tên là "Countdown". Nó sẽ có thuộc tính _length_ (độ dài) và _delay_ (khoảng thời gian giữa các lần thực thi). Việc thực thi - tức là mỗi lần _delay_ trôi qua sẽ làm gì - sẽ do một image quyết định. Như vậy, ai đó dùng CRD của chúng ta có thể tạo một countdown mà mỗi lần đếm sẽ, ví dụ, đăng một tin lên Twitter.

Làm mẫu, tôi sẽ dùng template từ [tài liệu](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/).

**resourcedefinition.yaml**

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name phải khớp với các trường spec bên dưới, và có dạng: <plural>.<group>
  name: countdowns.stable.dwk
spec:
  # group name dùng cho REST API: /apis/<group>/<version>
  group: stable.dwk
  # hoặc Namespaced hoặc Cluster
  scope: Namespaced
  names:
    # kind thường là dạng CamelCase số ít. Resource manifest sẽ dùng trường này.
    kind: Countdown
    # plural dùng trong URL: /apis/<group>/<version>/<plural>
    plural: countdowns
    # singular dùng làm alias trên CLI và hiển thị
    singular: countdown
    # shortNames cho phép dùng tên ngắn trên CLI
    shortNames:
      - cd
  # danh sách các version được hỗ trợ bởi CRD này
  versions:
    - name: v1
      # Mỗi version có thể bật/tắt bằng cờ Served.
      served: true
      # Chỉ một version được đánh dấu là storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                length:
                  type: integer
                delay:
                  type: integer
                image:
                  type: string
      additionalPrinterColumns:
        - name: Length
          type: integer
          description: Độ dài của countdown
          jsonPath: .spec.length
        - name: Delay
          type: integer
          description: Khoảng thời gian (ms) giữa các lần thực thi
          jsonPath: .spec.delay
```

Bây giờ chúng ta có thể tạo Countdown của riêng mình:

**countdown.yaml**

```yaml
apiVersion: stable.dwk/v1
kind: Countdown
metadata:
  name: doomsday
spec:
  length: 20
  delay: 1200
  image: jakousa/dwk-app10:sha-84d581d
```

Và sau đó:

```console
$ kubectl apply -f countdown.yaml
  countdown.stable.dwk/doomsday created

$ kubectl get cd
  NAME        LENGTH   DELAY
  doomsday    20       1200
```

Bây giờ chúng ta đã có một resource mới. Tiếp theo, hãy tạo một [custom controller](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#custom-controllers) mới sẽ khởi động một pod chạy container từ image và đảm bảo countdowns được xóa sau khi hoàn thành. Việc này sẽ cần phải code.

Để triển khai, tôi quyết định dùng [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/), một resource quen thuộc từ [phần 2](part-2/4-statefulsets-and-jobs#jobs-and-cronjobs).
Các pod do Job tạo ra sẽ chạy một lần cho đến khi hoàn thành. Tuy nhiên, cả Job đã hoàn thành lẫn Pod sẽ không tự động bị xóa. Chúng được giữ lại để có thể xem log sau khi thực thi.

Controller của chúng ta cần làm 3 việc:

- Tạo một Job từ một Countdown
- Lên lịch lại Job cho đến khi số lần thực thi (length) trong Countdown hoàn thành
- Xóa toàn bộ Job và Pod sau khi hoàn thành

Để triển khai controller, chúng ta cần thao tác cấp thấp và truy cập trực tiếp Kubernetes qua REST API.

Bằng cách lắng nghe API Kubernetes tại `/apis/stable.dwk/v1/countdowns?watch=true` chúng ta sẽ nhận được sự kiện ADDED cho mỗi Countdown object trong cụm. Sau đó, tạo job bằng cách phân tích dữ liệu từ message và POST payload hợp lệ tới `/apis/batch/v1/namespaces/<namespace>/jobs`.

Với job, chúng ta sẽ lắng nghe `/apis/batch/v1/jobs?watch=true` và chờ sự kiện MODIFIED khi trạng thái thành công được đặt thành true, rồi cập nhật label cho job để lưu trạng thái. Để xóa Job và Pod, gửi request delete tới `/api/v1/namespaces/<namespace>/pods/<pod_name>` và `/apis/batch/v1/namespaces/<namespace>/jobs/<job_name>`

Cuối cùng, có thể xóa countdown bằng request delete tới `/apis/stable.dwk/v1/namespaces/<namespace>/countdowns/<countdown_name>`.

Một phiên bản controller này có tại [đây](https://github.com/kubernetes-hy/material-example/tree/master/app10). Đã có sẵn image `jakousa/dwk-app10-controller:sha-4256579`. Tuy nhiên không thể chỉ deploy là chạy được vì nó chưa có quyền truy cập API. Chúng ta cần định nghĩa quyền truy cập phù hợp.

## RBAC

RBAC (Role-based access control) là phương pháp phân quyền cho phép định nghĩa quyền truy cập cho từng user, service account hoặc nhóm thông qua các role. Với trường hợp này, chúng ta sẽ định nghĩa một resource ServiceAccount.

**serviceaccount.yaml**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: countdown-controller-account
```

và sau đó chỉ định _serviceAccountName_ cho deployment

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: countdown-controller-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: countdown-controller
  template:
    metadata:
      labels:
        app: countdown-controller
    spec:
      serviceAccountName: countdown-controller-account
      containers:
        - name: countdown-controller
          image: jakousa/dwk-app10-controller:sha-4256579
```

Tiếp theo là định nghĩa role và các rule của nó. Có hai loại role: [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) và [Role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole). Role chỉ áp dụng cho một namespace, còn ClusterRole có thể truy cập tất cả namespace - trong trường hợp này, controller sẽ truy cập tất cả countdown ở mọi namespace nên cần ClusterRole.

Các rule được định nghĩa với apiGroup, resource và verbs. Ví dụ, jobs là `/apis/batch/v1/jobs?watch=true` nên thuộc apiGroup "batch", resource "jobs" và verbs xem [tài liệu](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#determine-the-request-verb). Core API group là chuỗi rỗng "" như với pods.

**clusterrole.yaml**

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: countdown-controller-role
rules:
  - apiGroups: [""]
    # ở cấp HTTP, resource để truy cập Pod là "pods"
    resources: ["pods"]
    verbs: ["get", "list", "delete"]
  - apiGroups: ["batch"]
    # ở cấp HTTP, resource để truy cập Job là "jobs"
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: ["stable.dwk"]
    resources: ["countdowns"]
    verbs: ["get", "list", "watch", "create", "delete"]
```

Và cuối cùng là bind ServiceAccount với role. Có hai loại binding: [ClusterRoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings) và [RoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings). Nếu dùng _RoleBinding_ với _ClusterRole_ thì chỉ giới hạn quyền trong một namespace. Ví dụ, nếu quyền truy cập secrets được định nghĩa ở ClusterRole và gán qua _RoleBinding_ cho namespace "test", thì chỉ truy cập được secrets trong namespace "test" - dù role là ClusterRole.

Ở đây cần _ClusterRoleBinding_ vì controller cần truy cập tất cả namespace từ namespace nó được deploy (ở đây là "default").

**clusterrolebinding.yaml**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: countdown-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: countdown-controller-role
subjects:
  - kind: ServiceAccount
    name: countdown-controller-account
    namespace: default
```

Sau khi deploy tất cả, có thể kiểm tra log sau khi apply countdown. (Có thể cần xóa pod để nó restart nếu trước đó chưa có quyền và bị treo)

```console
$ kubectl logs countdown-controller-dep-7ff598ffbf-q2rp5
  > app10@1.0.0 start /usr/src/app
  > node index.js

  Scheduling new job number 20 for countdown doomsday to namespace default
  Scheduling new job number 19 for countdown doomsday to namespace default
  ...
  Countdown ended. Removing countdown.
  Doing cleanup
```

<text-box name="Chọn ngôn ngữ cho CRDs" variant="hint">

Ví dụ controller countdown được cài đặt theo hai cách: dùng [Go](https://github.com/kubernetes-hy/material-example/tree/master/app10-go) và [JavaScript](https://github.com/kubernetes-hy/material-example/tree/master/app10).

Lựa chọn tốt nhất có lẽ là [Go](https://go.dev/) nhưng đây có thể không phải nơi phù hợp để học ngôn ngữ mới. README của [app10-go](https://github.com/kubernetes-hy/material-example/tree/master/app10-go) có hướng dẫn bắt đầu với CRD bằng Go!

</text-box>

<exercise name='Bài tập 5.01: Tự làm CRD & Controller'>

Bài tập này không phụ thuộc vào các bài trước. Bạn có thể chọn bất kỳ công nghệ nào để triển khai.

  <p style="color:firebrick;">Bài này khó!</p>

Chúng ta cần một resource _DummySite_ có thể dùng để tạo một trang HTML từ bất kỳ URL nào.

1. Tạo resource "DummySite" có thuộc tính chuỗi "website_url".

2. Tạo controller nhận object "DummySite" được tạo từ API

3. Controller tạo tất cả resource cần thiết cho chức năng này.

Tham khảo <https://kubernetes.io/docs/reference/kubernetes-api/> và <https://kubernetes.io/docs/reference/using-api/api-concepts/> để biết thêm về API Kubernetes, và
<https://kubernetes.io/docs/reference/using-api/client-libraries/> về các thư viện client.

Bạn cũng có thể tham khảo các ví dụ ứng dụng: [js](https://github.com/kubernetes-hy/material-example/tree/master/app10), [go](https://github.com/kubernetes-hy/material-example/tree/master/app10-go). Lưu ý ứng dụng JavaScript không tận dụng hết tính năng của [Kubernetes Client](https://github.com/kubernetes-client/javascript), mà gọi trực tiếp REST API.

Kiểm tra rằng khi tạo DummySite với website_url "[https://example.com/](https://example.com/)"
sẽ tạo một bản sao của website đó. Với website phức tạp, "bản sao" không cần hoàn chỉnh. Ví dụ với https://en.wikipedia.org/wiki/Kubernetes CSS có thể bị lỗi:

<img src="../img/wikipedia.png">

Controller không cần hoạt động hoàn hảo trong mọi trường hợp. Quy trình sau nên thành công:

1. apply role, account và binding.
2. apply deployment.
3. apply DummySite

</exercise>

---
path: "/part-2/2-organizing-a-cluster-vi"
title: "Tổ chức một cụm"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn

- biết các phương pháp tổ chức một cụm

- biết cách giới hạn pod vào các node cụ thể

</text-box>

Như bạn có thể tưởng tượng, có thể sẽ có rất nhiều tài nguyên bên trong một cụm. Thực tế, tại thời điểm viết tài liệu này, Kubernetes hỗ trợ hơn 100.000 pod trong một cụm duy nhất.

### Namespace

[Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) được sử dụng để tách biệt các tài nguyên. Một công ty sử dụng một cụm nhưng có nhiều dự án có thể dùng namespace để chia cụm thành các cụm ảo, mỗi dự án một namespace. Thông thường nhất, namespace được dùng để tách các môi trường như production, testing, staging. Bản ghi DNS cho service sẽ bao gồm namespace nên bạn vẫn có thể cho các project giao tiếp với nhau nếu cần thông qua địa chỉ service.namespace. Ví dụ, nếu một service tên là _cat-pictures_ nằm trong namespace _ns-test_, nó có thể được tìm thấy từ các namespace khác qua http://cat-pictures.ns-test.

Truy cập namespace với kubectl được thực hiện bằng cờ `-n`. Ví dụ, bạn có thể xem namespace kube-system có gì với

```shell
$ kubectl get pods -n kube-system
```

Để xem tất cả mọi thứ, bạn có thể dùng `--all-namespaces`.

```shell
$ kubectl get all --all-namespaces
```

Namespace nên được giữ tách biệt - ví dụ bạn có thể chạy tất cả các ví dụ và làm bài tập của khóa học này trong một cụm dùng chung với phần mềm quan trọng, nhưng đó không phải là ý hay. Quản trị viên nên đặt [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) cho namespace đó để bạn có thể chạy mọi thứ an toàn ở đó. Chúng ta sẽ tìm hiểu về giới hạn và yêu cầu tài nguyên sau.

Tạo một namespace chỉ cần một dòng lệnh (`kubectl create namespace example-namespace`). Bạn có thể chỉ định namespace sử dụng bằng cách thêm vào phần metadata của file yaml.

```yaml
# ...
metadata:
  namespace: example-namespace
  name: example
# ...
```

Nếu bạn thường xuyên sử dụng một namespace cụ thể, bạn có thể đặt namespace mặc định với `kubectl config set-context --current --namespace=<name>`.

Việc thay đổi namespace mặc định với lệnh trên khá bất tiện. May mắn thay, có một công cụ cộng đồng là _kubens_, giúp liệt kê các namespace hiện có và chuyển đổi namespace mặc định dễ dàng. Hãy cài đặt ngay [kubectx + kubens](https://github.com/ahmetb/kubectx), ngoài _kubens_ còn có lệnh _kubectx_ để chuyển đổi cụm bạn đang sử dụng. Lệnh _kubectx_ sẽ rất hữu ích khi chúng ta tạo thêm một cụm ở phần 3 trên Google Kubernetes Engine.

**Kubernetes Best Practices - Tổ chức Kubernetes với Namespace**

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/xpnZX3if9Tc" frameborder="0" allow="accelerometer; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<exercise name='Exercise 2.03: Giữ chúng tách biệt'>

Tạo một namespace cho các ứng dụng trong các bài tập. Di chuyển "Log output" và "Ping-pong" vào namespace đó và sử dụng namespace này cho tất cả các bài tập sau. Bạn có thể làm theo tài liệu trong namespace mặc định.

</exercise>

<exercise name='Exercise 2.04: Project v1.1'>

Tạo một namespace cho project và di chuyển tất cả những gì liên quan đến project vào namespace đó.

</exercise>

### Label

[Label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) được sử dụng để tách biệt một ứng dụng với các ứng dụng khác trong cùng một namespace và để nhóm các tài nguyên khác nhau lại với nhau. Label là các cặp key-value và có thể được chỉnh sửa, thêm hoặc xóa bất cứ lúc nào. Label cũng có thể được thêm vào hầu hết mọi thứ.

Label giúp chúng ta (con người) nhận diện tài nguyên và Kubernetes có thể sử dụng chúng để thao tác trên một nhóm tài nguyên. Bạn có thể truy vấn các tài nguyên có một label nhất định. Label cũng được dùng bởi [selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors) để chọn một tập hợp đối tượng.

Hãy xem label trong file yaml _Deployment_. Đây là file yaml đầu tiên chúng ta tạo ở đầu [phần 1](/part-1/1-first-deploy):

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hashgenerator-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hashgenerator
  template:
    metadata:
      labels:
        app: hashgenerator
    spec:
      containers:
        - name: hashgenerator
          image: jakousa/dwk-app1:b7fc18de2376da80ff0cfc72cf581a9f94d10e64
```

Trong trường hợp này, file yaml bao gồm cả selector và label. _selector_ và _matchLabels_ cho biết các chỉ dẫn của deployment sẽ áp dụng cho các pod có label _app: hashgenerator_. Như vậy, label là các cặp key-value. Phần metadata của template cũng bao gồm label với cặp _app: hashgenerator_.

Chúng ta có thể sử dụng cùng một label trên nhiều namespace và namespace sẽ giúp chúng không ảnh hưởng lẫn nhau.

Nhóm các đối tượng bằng label rất đơn giản. Chúng ta chỉ cần thêm label vào file yaml hoặc dùng lệnh [kubectl label](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_label/).

Ví dụ, chúng ta có thể thêm label _importance=great_ cho một pod như sau:

```shell
$ kubectl label po hashgenerator-dep-7b9b88f8bf-lvcv4 importance=great
  pod/hashgenerator-dep-7b9b88f8bf-lvcv4 labeled
```

Label có thể được dùng để giới hạn kết quả của lệnh _kubectl get_:

```shell
$ kubectl get pod -l importance=great
  NAME                                 READY   STATUS    RESTARTS   AGE
  hashgenerator-dep-7b9b88f8bf-lvcv4   1/1     Running   0          17m
```

Với label, chúng ta thậm chí có thể di chuyển pod lên các node đã được gán label. Giả sử chúng ta có một số node có đặc điểm mà ta muốn tránh, ví dụ như mạng chậm hơn. Với label và [nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) cấu hình trong deployment, chúng ta có thể làm điều đó. Đầu tiên, thêm _nodeSelector_ vào deployment rồi gán label cho node:

**deployment.yaml**

```yaml
    ...
    spec:
      containers:
        - name: hashgenerator
          image: jakousa/dwk-app1:b7fc18de2376da80ff0cfc72cf581a9f94d10e64
      nodeSelector:
        networkquality: excellent
```

Nếu pod đã chạy rồi, nó sẽ không tự động dừng pod để tránh thay đổi không mong muốn, thay vào đó một pod mới sẽ được lên lịch:

```shell
$ kubectl get po
  NAME                                 READY   STATUS    RESTARTS   AGE
  hashgenerator-dep-548d4d6c8d-mbblj   1/1     Running   0          107s
  hashgenerator-dep-7586cb6456-mktcl   0/1     Pending   0          23s
```

Trạng thái của pod mới là "Pending" vì chưa có node nào có network quality là excellent.

Tiếp theo, gán label cho _agent-1_ là node có network quality excellent và Kubernetes sẽ biết pod có thể chạy ở đâu:

```
$ kubectl label nodes k3d-k3s-default-agent-1 networkquality=excellent
  node/k3d-k3s-default-agent-1 labeled

$ kubectl get po
  NAME                                 READY   STATUS    RESTARTS   AGE
  hashgenerator-dep-7b9b88f8bf-mktcl   1/1     Running   0          5m30s
```

_nodeSelector_ là một công cụ khá thô. Nó rất phù hợp khi bạn muốn xác định các thuộc tính nhị phân, ví dụ _"đừng chạy ứng dụng này nếu node đang dùng HDD thay vì SSD"_ bằng cách gán label cho node theo loại ổ đĩa. Có những công cụ tinh vi hơn mà bạn nên dùng khi có một cụm gồm nhiều loại máy khác nhau, từ [máy bay chiến đấu](https://gcn.com/articles/2020/01/07/af-kubernetes-f16.aspx) đến máy nướng bánh mì hoặc siêu máy tính. Kubernetes có thể dùng [affinity](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/) và _anti-affinity_ để ưu tiên node cho ứng dụng nào và [taint với toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) để pod có thể tránh một số node nhất định. Ví dụ, nếu một máy có độ trễ mạng cao và bạn không muốn nó chạy các tác vụ cần độ trễ thấp.

Xem thêm [affinity và anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity) và [taint và toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) để biết thêm chi tiết. Chúng ta sẽ không gán pod vào node cụ thể trong khóa học này vì cụm của chúng ta là đồng nhất.

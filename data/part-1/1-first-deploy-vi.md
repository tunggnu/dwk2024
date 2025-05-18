---
path: "/part-1/1-first-deploy-vi"
title: "Triển khai đầu tiên"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn

- có thể tạo và chạy một cụm Kubernetes cục bộ với k3d

- có thể triển khai một ứng dụng đơn giản lên Kubernetes

</text-box>

## Microservices là gì?

Trong khóa học này, chúng ta sẽ nói về microservices và tạo ra các microservices. Trước khi bắt đầu với bất cứ điều gì khác, chúng ta cần định nghĩa microservice là gì. Hiện tại, có rất nhiều định nghĩa khác nhau về microservices.

Đối với khóa học này, chúng ta sẽ chọn định nghĩa của Sam Newman trong [Building Microservices](https://www.oreilly.com/library/view/building-microservices/9781491950340/): "**Microservices là các dịch vụ nhỏ, tự trị, phối hợp cùng nhau**". Đối lập với microservice là một dịch vụ tự chứa và độc lập gọi là [Monolith](https://en.wikipedia.org/wiki/Monolithic_application).

Khi nào nên sử dụng microservices? Trong video dưới đây, nơi Sam Newman và Martin Fowler thảo luận về microservices, câu trả lời là: "Khi bạn thực sự có lý do chính đáng".

Nó cũng bao gồm 3 lý do hàng đầu để sử dụng microservices:

- Triển khai độc lập không downtime
- Cô lập dữ liệu và xử lý xung quanh dữ liệu đó
- Sử dụng microservices để phản ánh cấu trúc tổ chức

**Khi nào nên sử dụng Microservices (và khi nào không nên!)**

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/GBTdnfD6s5Q" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Một ứng dụng theo kiến trúc microservices được cấu thành từ nhiều, thậm chí hàng chục, dịch vụ hoạt động độc lập. Việc quản lý và vận hành một hệ thống phức tạp như vậy là một thách thức. Đây chính là lúc Kubernetes phát huy vai trò của mình.

## Kubernetes là gì?

Giả sử bạn có 3 tiến trình và 2 máy tính không thể chạy cùng lúc cả 3 tiến trình. Bạn sẽ giải quyết vấn đề này như thế nào?

Bạn sẽ phải bắt đầu bằng việc quyết định 2 tiến trình nào sẽ chạy cùng một máy và tiến trình nào sẽ chạy ở máy khác. Làm thế nào để phân bổ chúng? Bằng cách cho tiến trình tốn nhiều tài nguyên nhất và ít tài nguyên nhất cùng một máy, hay để tiến trình tốn nhiều tài nguyên nhất chạy riêng? Có thể bạn muốn thêm một tiến trình và giờ phải tổ chức lại tất cả. Điều gì xảy ra khi bạn có nhiều hơn 2 máy và nhiều hơn 3 tiến trình? Một trong các tiến trình đang chiếm hết bộ nhớ và bạn cần tách nó ra khỏi "ứng dụng ngân hàng quan trọng". Chúng ta nên ảo hóa mọi thứ? Containers sẽ giải quyết vấn đề đó, đúng không? Bạn sẽ chuyển tiến trình quan trọng nhất sang máy mới? Có thể một số tiến trình cần giao tiếp với nhau và giờ bạn phải xử lý mạng lưới. Nếu một máy bị hỏng thì sao? Còn kế hoạch đi uống bia thủ công vào thứ Sáu của bạn thì sao?

Sẽ thế nào nếu bạn chỉ cần định nghĩa "Tiến trình này nên có 6 bản sao sử dụng X lượng tài nguyên." và có 2..N máy tính hoạt động như một thực thể duy nhất để đáp ứng yêu cầu của bạn? Đó chỉ là một trong những điều mà Kubernetes làm được.

<blockquote><p lang="en" dir="ltr">Về bản chất, Kubernetes là tổng hợp tất cả các bash script và các best practice mà hầu hết các quản trị viên hệ thống sẽ tự xây dựng theo thời gian, được trình bày như một hệ thống duy nhất phía sau một tập hợp API khai báo.</p>&mdash; Kelsey Hightower (@kelseyhightower) <a href="https://twitter.com/kelseyhightower/status/1125440400355782657?ref_src=twsrc%5Etfw">Ngày 6 tháng 5, 2019</a></blockquote>

Hoặc chính thức hơn:

“Kubernetes (K8s) là một hệ thống mã nguồn mở để tự động hóa việc triển khai, mở rộng và quản lý các ứng dụng container hóa. Nó nhóm các container tạo thành một ứng dụng thành các đơn vị logic để dễ dàng quản lý và khám phá.” - [kubernetes.io](https://kubernetes.io/)

Một hệ thống điều phối container như Kubernetes thường là cần thiết khi duy trì các ứng dụng container hóa. Trách nhiệm chính của một hệ thống điều phối là khởi động và dừng các container. Ngoài ra, chúng còn cung cấp mạng lưới giữa các container và giám sát sức khỏe. Thay vì phải thủ công chạy `docker run critical-bank-application` mỗi khi ứng dụng bị crash, hoặc khởi động lại nếu nó không phản hồi, chúng ta muốn hệ thống tự động giữ cho ứng dụng luôn khỏe mạnh.

Bạn có thể đã biết một hệ thống điều phối, _docker compose_, cũng thực hiện các nhiệm vụ tương tự: khởi động và dừng, mạng lưới và giám sát sức khỏe. Điều làm cho Kubernetes đặc biệt là bộ tính năng mạnh mẽ để tự động hóa tất cả những điều đó.

Hãy đọc [truyện tranh này](https://cloud.google.com/kubernetes-engine/kubernetes-comic/) và xem video dưới đây để có cái nhìn nhanh. Bạn có thể muốn xem lại chúng sau phần này!

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/Q4W8Z-D-gcQ" frameborder="0" allow="accelerometer; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Chúng ta sẽ bắt đầu với một bản phân phối Kubernetes nhẹ. [K3s - 5 nhỏ hơn K8s](https://k3s.io/), cung cấp cho chúng ta một cụm Kubernetes thực sự có thể chạy trong các container bằng [k3d](https://github.com/rancher/k3d).

### Cụm Kubernetes với k3d

#### Cụm là gì?

Cụm là một nhóm các máy, _node_, làm việc cùng nhau - trong trường hợp này, chúng là một phần của cụm Kubernetes. Cụm Kubernetes có thể có bất kỳ kích thước nào - một cụm một node sẽ bao gồm một máy chủ chứa control-plane của Kubernetes (cung cấp API và duy trì cụm) và cụm đó có thể mở rộng lên đến 5000 node, tính đến Kubernetes v1.18.

Chúng ta sẽ sử dụng thuật ngữ "server node" để chỉ các node có control-plane và "agent node" để chỉ các node không có vai trò đó. Một cụm kubernetes cơ bản có thể trông như sau:

<img src="../img/without_k3d.png">

#### Khởi tạo cụm với k3d

Chúng ta sẽ sử dụng k3d để tạo một nhóm các container Docker chạy k3s. Hướng dẫn cài đặt, hoặc ít nhất là liên kết đến chúng, nằm ở [phần 0](/part-0#installing-k3d). Lý do sử dụng k3d là nó cho phép chúng ta tạo một cụm mà không cần lo lắng về máy ảo hay máy vật lý. Với k3d, cụm cơ bản của chúng ta sẽ trông như sau:

<img src="../img/with_k3d.png">

Vì các node là container nên chúng ta sẽ cần cấu hình một chút để chúng hoạt động như mong muốn. Chúng ta sẽ làm điều đó sau. Việc tạo cụm Kubernetes của riêng chúng ta với k3d chỉ cần một lệnh duy nhất.

```shell
$ k3d cluster create -a 2
```

Lệnh này tạo ra một cụm Kubernetes với 2 agent node. Vì chúng nằm trong Docker, bạn có thể xác nhận sự tồn tại của chúng bằng `docker ps`.

```shell
$ docker ps
  CONTAINER ID   IMAGE                            COMMAND                  CREATED          STATUS          PORTS                             NAMES
  b25a9bb6c42f   ghcr.io/k3d-io/k3d-tools:5.4.1   "/app/k3d-tools noop"    56 seconds ago   Up 55 seconds                                     k3d-k3s-default-tools
  19f992606131   ghcr.io/k3d-io/k3d-proxy:5.4.1   "/bin/sh -c nginx-pr…"   56 seconds ago   Up 32 seconds   80/tcp, 0.0.0.0:50122->6443/tcp   k3d-k3s-default-serverlb
  7a8bf6a44099   rancher/k3s:v1.22.7-k3s1         "/bin/k3d-entrypoint…"   56 seconds ago   Up 43 seconds                                     k3d-k3s-default-agent-1
  c85fbcbcf9b2   rancher/k3s:v1.22.7-k3s1         "/bin/k3d-entrypoint…"   56 seconds ago   Up 43 seconds                                     k3d-k3s-default-agent-0
  7191a3bdae7a   rancher/k3s:v1.22.7-k3s1         "/bin/k3d-entrypoint…"   56 seconds ago   Up 52 seconds                                     k3d-k3s-default-server-0
```

Kéo sang trái một chút, bạn cũng sẽ thấy cổng 6443 được mở cho "k3d-k3s-default-serverlb", một proxy "load balancer" hữu ích, sẽ chuyển tiếp kết nối đến 6443 vào server node, và đó là cách chúng ta truy cập nội dung của cụm. Cổng trên máy của chúng ta, ví dụ 50122, được chọn ngẫu nhiên. Chúng ta có thể bỏ qua load balancer với `k3d cluster create -a 2 --no-lb` và cổng sẽ mở trực tiếp đến server node. Có load balancer sẽ cung cấp cho chúng ta một số tính năng mà nếu không sẽ không có, nên hãy giữ nó.

K3d cũng đã thiết lập một [kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), một file dùng để tổ chức thông tin về các cụm, người dùng, namespace và cơ chế xác thực. Nội dung của file có thể xem bằng lệnh `k3d kubeconfig get k3s-default`.

Công cụ khác mà chúng ta sẽ sử dụng trong khóa học này là [kubectl](https://kubernetes.io/docs/reference/kubectl/). Kubectl là công cụ dòng lệnh của Kubernetes và cho phép chúng ta tương tác với cụm. Kubectl sẽ đọc kubeconfig từ biến môi trường KUBECONFIG hoặc mặc định từ `~/.kube/config` và sử dụng thông tin để kết nối với cụm. Nội dung bao gồm chứng chỉ, mật khẩu và địa chỉ API của cụm. Bạn có thể đặt context bằng `kubectl config use-context k3d-k3s-default`.

Giờ kubectl sẽ có thể truy cập cụm

```shell
$ kubectl cluster-info
  Kubernetes control plane is running at https://0.0.0.0:50122
  CoreDNS is running at https://0.0.0.0:50122/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
  Metrics-server is running at https://0.0.0.0:50122/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

Chúng ta có thể thấy kubectl đang kết nối đến container _k3d-k3s-default-serverlb_ qua (trong trường hợp này) cổng 50122.

Nếu bạn muốn dừng / khởi động lại cụm, chỉ cần chạy

```shell
$ k3d cluster stop
  INFO[0000] Stopping cluster 'k3s-default'
  INFO[0011] Stopped cluster 'k3s-default'

$ k3d cluster start
  INFO[0000] Using the k3d-tools node to gather environment information
  INFO[0000] Starting existing tools node k3d-k3s-default-tools...
  INFO[0000] Starting Node 'k3d-k3s-default-tools'
  INFO[0001] Starting new tools node...
  INFO[0001] Starting Node 'k3d-k3s-default-tools'
  INFO[0003] Starting cluster 'k3s-default'
  INFO[0003] Starting servers...
  INFO[0003] Starting Node 'k3d-k3s-default-server-0'
  INFO[0010] Starting agents...
  INFO[0010] Starting Node 'k3d-k3s-default-agent-1'
  INFO[0011] Starting Node 'k3d-k3s-default-agent-0'
  INFO[0027] Starting helpers...
  INFO[0027] Starting Node 'k3d-k3s-default-serverlb'
  INFO[0027] Starting Node 'k3d-k3s-default-tools'
  INFO[0035] Injecting records for hostAliases (incl. host.k3d.internal) and for 5 network members into CoreDNS configmap...
  INFO[0038] Started cluster 'k3s-default'
```

Hiện tại, **chúng ta cần cụm đang chạy**, nhưng nếu muốn xóa cụm, bạn có thể chạy `k3d cluster delete`.

## Triển khai đầu tiên

### Chuẩn bị cho lần triển khai đầu tiên

Trước khi có thể triển khai bất cứ thứ gì, chúng ta cần một ứng dụng nhỏ để triển khai. Trong suốt khóa học, bạn sẽ phát triển ứng dụng của riêng mình. Công nghệ sử dụng cho ứng dụng không quan trọng - trong ví dụ này chúng ta sẽ dùng [Node.js](https://nodejs.org/en/) nhưng ứng dụng mẫu cũng sẽ được cung cấp qua GitHub cũng như Docker Hub.

Hãy khởi chạy một ứng dụng tạo và xuất ra một hash mỗi 5 giây.

Tôi đã chuẩn bị sẵn một ứng dụng [tại đây](https://github.com/kubernetes-hy/material-example/tree/master/app1), bạn có thể thử với `docker run jakousa/dwk-app1`.

Để triển khai một image, chúng ta cần cụm có thể truy cập image đó. Mặc định, Kubernetes được thiết kế để sử dụng với một registry. Chúng ta sẽ sử dụng registry quen thuộc _Docker Hub_, mà chúng ta cũng đã dùng trong [DevOps with Docker](http://devopswithdocker.com/). Nếu bạn chưa từng dùng Docker Hub, đó là nơi mà Docker client mặc định truy cập. Ví dụ khi bạn chạy `docker pull nginx`, image nginx đến từ Docker Hub. Xem khóa học [DevOps for Docker](https://devopswithdocker.com/) để biết thêm chi tiết, ví dụ về việc push image lên Docker Hub nếu cần.

Cũng có thể sử dụng image local với k3d, nhưng điều đó sẽ không hoạt động với các giải pháp không phải k3d. Nếu muốn thử dùng image local, hãy dùng `k3d image import <image-name>`. Sau đó bạn phải chỉnh sửa [imagePullPolicy](https://kubernetes.io/docs/concepts/containers/images/#image-pull-policy) của deployment từ mặc định `Always` thành `IfNotPresent` hoặc `Never` để cho phép sử dụng image local. Deployment có thể chỉnh sửa sau khi tạo bằng `kubectl edit deployment <deployment-name>`.

<text-box name="Ứng dụng ví dụ" variant="hint">
Trong tương lai, tài liệu sẽ sử dụng các ứng dụng được cung cấp trong các lệnh. Bạn có thể làm theo bằng cách thay đổi image thành ứng dụng của bạn. Hầu hết mọi thứ đều có trong cùng một repository <a href="https://github.com/kubernetes-hy/material-example">https://github.com/kubernetes-hy/material-example</a>.
</text-box>

Giờ chúng ta đã sẵn sàng để triển khai ứng dụng đầu tiên vào Kubernetes!

### Triển khai

Để triển khai một ứng dụng, chúng ta cần tạo một đối tượng [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) với image.

```shell
$ kubectl create deployment hashgenerator-dep --image=jakousa/dwk-app1
  deployment.apps/hashgenerator-dep created
```

Lệnh này tạo ra một vài thứ để chúng ta xem xét

- một resource [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) và
- một resource [pod](https://kubernetes.io/docs/concepts/workloads/pods/).

#### Pod là gì?

_Pod_ là một lớp trừu tượng bao quanh một hoặc nhiều container. Pod cung cấp một ngữ cảnh cho 1..N container để chúng có thể chia sẻ lưu trữ và mạng. Nó rất giống với cách bạn đã sử dụng container để định nghĩa môi trường cho một tiến trình đơn. Có thể coi nó như một container của các container. _Hầu hết_ các quy tắc tương tự đều áp dụng: nó sẽ bị xóa nếu các container bên trong dừng chạy và các file chứa trong đó sẽ bị mất cùng với nó.

<img src="../img/pods.png">

Đọc tài liệu hoặc tìm kiếm trên internet không phải là cách duy nhất để tìm thông tin về các resource khác nhau của Kubernetes. Chúng ta có thể truy cập các giải thích đơn giản ngay từ dòng lệnh bằng lệnh `kubectl explain RESOURCE`.

Ví dụ, để có mô tả về Pod và các trường bắt buộc, chúng ta có thể dùng lệnh sau.

```shell
$ kubectl explain pod
  KIND:     Pod
  VERSION:  v1

  DESCRIPTION:
       Pod là một tập hợp các container có thể chạy trên một host. Resource này được tạo bởi client và được lên lịch trên các host.
```

Trong Kubernetes, tất cả các thực thể tồn tại đều được gọi là [object](https://kubernetes.io/docs/concepts/overview/working-with-objects/). Bạn có thể liệt kê tất cả object của một resource với `kubectl get RESOURCE`.

```shell
$ kubectl get pods
  NAME                               READY   STATUS    RESTARTS   AGE
  hashgenerator-dep-6965c5c7-2pkxc   1/1     Running   0          2m1s
```

#### Deployment resource là gì?

Một resource [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) chịu trách nhiệm triển khai. Nó là cách để nói với Kubernetes bạn muốn container nào, chúng nên chạy như thế nào và bao nhiêu bản sao nên chạy.

Khi chúng ta tạo Deployment, chúng ta cũng tạo một object [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/). ReplicaSet dùng để chỉ định số lượng bản sao của một Pod bạn muốn. Nó sẽ xóa hoặc tạo Pod cho đến khi số lượng Pod bạn muốn đang chạy. ReplicaSet được quản lý bởi Deployment và bạn không cần phải tự định nghĩa hay chỉnh sửa chúng. Nếu muốn quản lý số lượng replica, bạn có thể chỉnh sửa Deployment và nó sẽ tự động cập nhật ReplicaSet.

Bạn có thể xem các deployment như sau:

```shell
$ kubectl get deployments
  NAME                READY   UP-TO-DATE   AVAILABLE   AGE
  hashgenerator-dep   1/1     1            1           54s
```

1/1 replica đã sẵn sàng! Chúng ta sẽ thử nhiều replica hơn sau. Nếu trạng thái của bạn không giống như vậy, hãy kiểm tra [trang này](/known-problems-solutions#first-deployment).

Để xem output, chúng ta có thể chạy `kubectl logs -f hashgenerator-dep-6965c5c7-2pkxc`

<text-box name="Tự động hoàn thành" variant="hint">
Bạn có thể dùng `source <(kubectl completion bash)` để tiết kiệm rất nhiều thời gian. Thêm nó vào .bashrc để tự động tải. (Cũng có cho zsh với `source <(kubectl completion zsh)`). Nếu bạn chưa từng dùng completion, hãy đọc <a href="https://iridakos.com/programming/2018/03/01/bash-programmable-completion-tutorial">hướng dẫn này</a>. Thông tin thêm về lệnh kubectl completion có thể tìm thấy <a href="https://kubernetes.io/docs/reference/kubectl/generated/kubectl_completion/">tại đây</a>.
</text-box>

Một danh sách hữu ích các lệnh docker-cli được chuyển sang kubectl có tại [https://kubernetes.io/docs/reference/kubectl/docker-cli-to-kubectl/](https://kubernetes.io/docs/reference/kubectl/docker-cli-to-kubectl/)

<exercise name='Bài tập 1.01: Bắt đầu'>

**Bài tập có thể làm với bất kỳ ngôn ngữ và framework nào bạn muốn.**

Tạo một ứng dụng sinh ra một chuỗi ngẫu nhiên khi khởi động, lưu chuỗi này vào bộ nhớ, và xuất ra mỗi 5 giây kèm timestamp. Ví dụ:

```text
2020-03-30T12:15:17.705Z: 8523ecb1-c716-4cb6-a044-b9e83bb98e43
2020-03-30T12:15:22.705Z: 8523ecb1-c716-4cb6-a044-b9e83bb98e43
```

Triển khai nó vào cụm Kubernetes của bạn và xác nhận rằng nó đang chạy với `kubectl logs ...`

Bạn sẽ tiếp tục xây dựng ứng dụng này trong các bài tập tiếp theo. Ứng dụng này sẽ được gọi là "Log output".

</exercise>

<exercise name='Bài tập 1.02: Dự án v0.1'>

**Dự án có thể làm với bất kỳ ngôn ngữ và framework nào bạn muốn**

Dự án sẽ là một ứng dụng todo đơn giản với các tính năng quen thuộc: tạo, đọc, cập nhật và xóa (CRUD). Chúng ta sẽ phát triển nó trong suốt các phần của khóa học. Kiểm tra tiêu đề bài tập để biết "Project vX.Y" là về xây dựng dự án.

Todo là một đoạn văn bản như "Tôi cần dọn nhà" có thể ở trạng thái chưa hoàn thành hoặc đã hoàn thành.

  <img src="../img/project.png" alt="Quá trình phát triển dự án" />

Các đường nét đứt phân tách các khác biệt lớn qua các phần của khóa học. Một số bài tập không được đưa vào hình. Kết nối giữa hầu hết các pod cũng không được đưa vào. Bạn có thể làm theo cách bạn muốn.

Hãy nhớ điều này nếu bạn muốn tránh làm nhiều việc hơn cần thiết.

Bắt đầu thôi!

Tạo một web server xuất ra "Server started in port NNNN" khi khởi động và triển khai nó vào cụm Kubernetes của bạn. Hãy đảm bảo rằng biến môi trường PORT có thể được sử dụng để chọn port đó. Bạn sẽ chưa thể truy cập port khi nó chạy trong Kubernetes. Chúng ta sẽ cấu hình truy cập khi đến phần networking.

</exercise>

## Cấu hình khai báo với YAML

Chúng ta đã tạo deployment với

```shell
$ kubectl create deployment hashgenerator-dep --image=jakousa/dwk-app1
```

Nếu muốn scale lên 4 lần và cập nhật image:

```shell
$ kubectl scale deployment/hashgenerator-dep --replicas=4

$ kubectl set image deployment/hashgenerator-dep dwk-app1=jakousa/dwk-app1:b7fc18de2376da80ff0cfc72cf581a9f94d10e64
```

Mọi thứ bắt đầu trở nên rất rối rắm. Thật khó tưởng tượng ai đó có thể duy trì nhiều ứng dụng như vậy bằng cách này. May mắn thay, chúng ta sẽ sử dụng cách _khai báo_ nơi chúng ta định nghĩa trạng thái mong muốn thay vì cách thay đổi từng bước. Cách này bền vững hơn về lâu dài và giúp chúng ta giữ được sự tỉnh táo.

Trước khi làm lại các bước trước bằng cách khai báo, hãy xóa deployment hiện tại.

```shell
$ kubectl delete deployment hashgenerator-dep
  deployment.apps "hashgenerator-dep" deleted
```

và tạo một thư mục mới tên là `manifests` và đặt một file tên là deployment.yaml với nội dung sau (bạn có thể xem ví dụ [tại đây](https://github.com/kubernetes-hy/material-example/tree/master/app1)):

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

<text-box name="Trình soạn thảo văn bản yêu thích" variant="hint">
  Cá nhân tôi sử dụng Visual Studio Code để tạo các file yaml này. Nó có tính năng autofill, định nghĩa và kiểm tra cú pháp hữu ích cho Kubernetes với extension Kubernetes của Microsoft. Ngay cả bây giờ nó cũng cảnh báo rằng chúng ta chưa định nghĩa giới hạn tài nguyên. Tôi sẽ chưa quan tâm đến cảnh báo đó, nhưng bạn có thể tự tìm hiểu nếu muốn.
</text-box>

File này trông rất giống với các file docker-compose.yaml mà chúng ta đã từng viết. Hãy bỏ qua những gì chúng ta chưa biết, chủ yếu là labels, và tập trung vào những gì chúng ta biết:

- Chúng ta khai báo loại resource (kind: Deployment)
- Chúng ta khai báo tên trong metadata (name: hashgenerator-dep)
- Chúng ta khai báo số lượng (replicas: 1)
- Chúng ta khai báo container sử dụng image nào và tên gì

Áp dụng deployment với lệnh apply:

```shell
$ kubectl apply -f manifests/deployment.yaml
  deployment.apps/hashgenerator-dep created
```

Vậy là xong, nhưng để ôn lại, hãy xóa và tạo lại:

```shell
$ kubectl delete -f manifests/deployment.yaml
  deployment.apps "hashgenerator-dep" deleted

$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app1/manifests/deployment.yaml
  deployment.apps/hashgenerator-dep created
```

Wow! Việc bạn có thể apply manifest từ internet như vậy sẽ rất hữu ích.

Thay vì xóa deployment, chúng ta chỉ cần apply một deployment đã chỉnh sửa lên cái đang có. Kubernetes sẽ tự động rollout phiên bản mới. Bằng cách sử dụng tag (ví dụ `dwk/image:tag`) trong deployment, mỗi lần cập nhật image bạn chỉ cần chỉnh sửa và apply lại file deployment yaml. Trước đây bạn có thể luôn dùng tag 'latest', hoặc không quan tâm đến tag. Từ tag, Kubernetes sẽ biết image là mới và pull về.

Khi cập nhật bất cứ thứ gì trong Kubernetes, việc sử dụng delete thực ra là một anti-pattern và chỉ nên dùng khi không còn cách nào khác. Miễn là bạn không xóa resource, Kubernetes sẽ thực hiện rolling update, đảm bảo downtime tối thiểu (hoặc không có) cho ứng dụng. Nói về anti-pattern: bạn cũng nên tránh làm mọi thứ theo kiểu imperative! Nếu file của bạn không nói cho Kubernetes và nhóm của bạn biết trạng thái nên là gì mà thay vào đó bạn chạy các lệnh chỉnh sửa trạng thái, bạn chỉ đang làm giảm [bus factor](https://en.wikipedia.org/wiki/Bus_factor) cho cụm và ứng dụng của mình.

<exercise name='Bài tập 1.03: Cách tiếp cận khai báo'>

Trong ứng dụng "Log output" của bạn, hãy tạo một thư mục cho manifests và chuyển deployment sang file khai báo.

Đảm bảo mọi thứ vẫn hoạt động bằng cách khởi động lại và theo dõi logs.

</exercise>

<exercise name='Bài tập 1.04: Dự án v0.2'>

Tạo deployment.yaml cho dự án.

Bạn sẽ chưa thể truy cập port nhưng điều đó sẽ đến sớm thôi.

</exercise>

Lưu ý việc apply một deployment mới sẽ không cập nhật ứng dụng trừ khi tag được cập nhật. Bạn sẽ không cần xóa deployment nếu luôn dùng tag mới.

Quy trình cơ bản của bạn có thể như sau:

```shell
$ docker build -t <image>:<new_tag>

$ docker push <image>:<new_tag>
```

Sau đó chỉnh sửa deployment.yaml để tag được cập nhật thành \<new_tag\> và chạy lệnh sau

```shell
$ kubectl apply -f manifests/deployment.yaml
```

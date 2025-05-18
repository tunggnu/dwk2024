---
path: "/part-1/3-introduction-to-networking-vi"
title: "Giới thiệu về Mạng"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn

- có thể sử dụng Service để truy cập ứng dụng từ bên ngoài cụm

- có thể sử dụng Ingress để truy cập ứng dụng từ bên ngoài cụm

</text-box>

Quay lại với việc phát triển! Khởi động lại và theo dõi log thật thú vị. Tiếp theo, chúng ta sẽ mở một endpoint cho ứng dụng và truy cập nó qua HTTP.

#### Ứng dụng mạng đơn giản

Hãy phát triển ứng dụng của chúng ta sao cho nó có một máy chủ HTTP phản hồi với hai giá trị băm: một giá trị băm được lưu cho đến khi tiến trình kết thúc và một giá trị băm dành riêng cho mỗi yêu cầu. Nội dung phản hồi có thể là "Application abc123. Request 94k9m2". Bạn có thể chọn bất kỳ cổng nào để lắng nghe.

Tôi đã chuẩn bị sẵn một ví dụ [tại đây](https://github.com/kubernetes-hy/material-example/tree/master/app2). Mặc định, nó sẽ lắng nghe ở cổng 3000.

```shell
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app2/manifests/deployment.yaml
  deployment.apps/hashresponse-dep được tạo
```

### Kết nối từ bên ngoài cụm

Chúng ta có thể xác nhận rằng hashresponse-dep đang hoạt động bằng lệnh `port-forward`. Hãy xem tên của pod trước rồi chuyển tiếp cổng đến đó:

```shell
$ kubectl get po
  NAME                                READY   STATUS    RESTARTS   AGE
  hashgenerator-dep-5cbbf97d5-z2ct9   1/1     Running   0          20h
  hashresponse-dep-57bcc888d7-dj5vk   1/1     Running   0          19h

$ kubectl port-forward hashresponse-dep-57bcc888d7-dj5vk 3003:3000
  Đang chuyển tiếp từ 127.0.0.1:3003 -> 3000
  Đang chuyển tiếp từ [::1]:3003 -> 3000
```

Bây giờ chúng ta có thể xem phản hồi từ http://localhost:3003 và xác nhận rằng nó hoạt động như mong đợi.

<exercise name='Exercise 1.05: Project v0.3'>

Hãy làm cho project phản hồi lại một điều gì đó khi nhận được yêu cầu GET gửi đến project. Một trang html đơn giản là đủ hoặc bạn có thể triển khai thứ gì đó phức tạp hơn như một single-page-application.

Xem [tại đây](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/) để biết cách định nghĩa biến môi trường cho container.

Sử dụng `kubectl port-forward` để xác nhận rằng project có thể truy cập và hoạt động trong cụm bằng cách dùng trình duyệt để truy cập project.

</exercise>

Kết nối bên ngoài với Docker sử dụng cờ -p `-p 3003:3000` hoặc trong Docker compose là khai báo ports. Đáng tiếc, Kubernetes không đơn giản như vậy. Chúng ta sẽ sử dụng hoặc _Service_ hoặc _Ingress_.

#### Trước hết

Vì chúng ta đang chạy cụm bên trong Docker với k3d nên sẽ cần một số chuẩn bị.

Mở một đường dẫn từ bên ngoài cụm đến pod sẽ không đủ nếu chúng ta không có cách truy cập vào cụm bên trong các container!

```shell
$ docker ps
  CONTAINER ID    IMAGE                      COMMAND                  CREATED             STATUS              PORTS                             NAMES
  b60f6c246ebb    ghcr.io/k3d-io/k3d-proxy:5 "/bin/sh -c nginx-pr…"   2 giờ trước         Đang chạy 2 giờ     80/tcp, 0.0.0.0:58264->6443/tcp   k3d-k3s-default-serverlb
  553041f96fc6    rancher/k3s:latest         "/bin/k3s agent"         2 giờ trước         Đang chạy 2 giờ                                        k3d-k3s-default-agent-1
  aebd23c2ef99    rancher/k3s:latest         "/bin/k3s agent"         2 giờ trước         Đang chạy 2 giờ                                        k3d-k3s-default-agent-0
  a34e49184d37    rancher/k3s:latest         "/bin/k3s server --t…"   2 giờ trước         Đang chạy 2 giờ                                        k3d-k3s-default-server-0
```

Kéo sang phải một chút, chúng ta thấy K3d đã chuẩn bị sẵn cổng 6443 để truy cập API. Ngoài ra, cổng 80 cũng đã mở. Tất cả các yêu cầu đến load balancer này sẽ được chuyển tiếp đến cùng cổng của tất cả các node server trong cụm. Tuy nhiên, cho mục đích kiểm thử, chúng ta sẽ muốn _mở một cổng riêng cho một node cụ thể_. Hãy xóa cụm cũ và tạo một cụm mới với một số cổng được mở.

Tài liệu [K3d](https://k3d.io/v5.3.0/usage/commands/k3d_cluster_create/) cho biết cách mở cổng, chúng ta sẽ mở cổng local 8081 tới 80 ở k3d-k3s-default-serverlb và local 8082 tới 30080 ở k3d-k3s-default-agent-0. Cổng 30080 được chọn gần như ngẫu nhiên, nhưng cần nằm trong khoảng 30000-32767 cho bước tiếp theo:

```shell
$ k3d cluster delete
  INFO[0000] Đang xóa cụm 'k3s-default'
  ...
  INFO[0002] Đã xóa thành công cụm k3s-default!

$ k3d cluster create --port 8082:30080@agent:0 -p 8081:80@loadbalancer --agents 2
  INFO[0000] Đã tạo mạng 'k3d-k3s-default'
  ...
  INFO[0021] Đã tạo cụm 'k3s-default' thành công!
  INFO[0021] Bây giờ bạn có thể sử dụng như sau:
  kubectl cluster-info

$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app2/manifests/deployment.yaml
  deployment.apps/hashresponse-dep được tạo
```

Ở trên, "agent:0" và "loadbalancer" dựa trên tài liệu [K3d](https://github.com/rancher/k3d/blob/main/docs/usage/exposing_services.md) và đọc mã nguồn từ [đây](https://github.com/rancher/k3d/blob/11cc7979228f304025d61254eb0c0cb2745b9444/cmd/util/filter.go#L119) và [đây](https://github.com/rancher/k3d/blob/main/pkg/types/types.go#L65)

Bây giờ chúng ta có thể truy cập qua cổng 8081 tới node server (thực ra là tất cả các node) và 8082 tới cổng 30080 của một node agent. Chúng sẽ được dùng để trình diễn các phương pháp giao tiếp khác nhau với các server.

<text-box name="Giới hạn của môi trường cục bộ" variant="hint">

Thiết lập này không hoàn hảo, trong tương lai chúng ta sẽ chỉ có một số lượng cổng giới hạn. Tuy nhiên, điều này là đủ cho các trường hợp sử dụng của chúng ta.

Hệ điều hành của bạn có thể hỗ trợ sử dụng mạng host nên không cần mở cổng. Tuy nhiên, tôi chưa có kinh nghiệm với điều này.

</text-box>

#### Service là gì?

Giống như _Deployment_ lo việc triển khai cho chúng ta. _Service_ sẽ lo việc phục vụ ứng dụng cho các kết nối từ bên ngoài (và cả bên trong!) cụm.

Tạo một file service.yaml trong thư mục manifests và chúng ta cần service thực hiện các việc sau:

1. Khai báo rằng chúng ta muốn một Service
2. Khai báo cổng sẽ lắng nghe
3. Khai báo ứng dụng mà yêu cầu sẽ được chuyển đến
4. Khai báo cổng mà yêu cầu sẽ được chuyển đến

Điều này sẽ thành một file yaml như sau

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hashresponse-svc
spec:
  type: NodePort
  selector:
    app: hashresponse # Đây là app đã khai báo trong deployment.
  ports:
    - name: http
      nodePort: 30080 # Đây là cổng sẽ mở ra ngoài. Giá trị nodePort phải nằm trong khoảng 30000-32767
      protocol: TCP
      port: 1234 # Đây là cổng dùng trong cụm, có thể là bất kỳ giá trị nào
      targetPort: 3000 # Đây là cổng đích
```

```shell
$ kubectl apply -f manifests/service.yaml
  service/hashresponse-svc được tạo
```

Vì chúng ta đã publish 8082 thành 30080 nên giờ có thể truy cập qua http://localhost:8082.

Chúng ta vừa định nghĩa một nodeport với `type: NodePort`. [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) đơn giản là các cổng được Kubernetes mở ra cho **tất cả các node** và service sẽ xử lý các yêu cầu trên cổng đó. NodePort không linh hoạt và bạn phải gán một cổng khác nhau cho mỗi ứng dụng. Vì vậy NodePort không được dùng trong môi trường production nhưng rất hữu ích để tìm hiểu.

Thay vì NodePort, chúng ta nên dùng _LoadBalancer_ nhưng loại này "chỉ" hoạt động với các nhà cung cấp đám mây vì nó cấu hình một load balancer (có thể tốn phí). Chúng ta sẽ tìm hiểu về nó ở phần 3.

Có một tài nguyên nữa sẽ giúp chúng ta phục vụ ứng dụng, đó là _Ingress_.

<exercise name='Exercise 1.06: Project v0.4'>

Sử dụng NodePort Service để cho phép truy cập vào project.

</exercise>

#### Ingress là gì?

Tài nguyên truy cập mạng vào [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) là một loại tài nguyên hoàn toàn khác với _Service_. Nếu bạn nhớ mô hình [OSI](https://en.wikipedia.org/wiki/OSI_model), nó hoạt động ở tầng 7 còn service ở tầng 4. Bạn có thể thấy chúng được dùng cùng nhau: đầu tiên là _LoadBalancer_ rồi đến Ingress để xử lý định tuyến. Trong trường hợp của chúng ta, vì không có load balancer nên có thể dùng Ingress như điểm dừng đầu tiên. Nếu bạn quen với reverse proxy như Nginx, Ingress sẽ rất quen thuộc.

Ingress được triển khai bởi nhiều "controller" khác nhau. Điều này nghĩa là ingress không tự động hoạt động trong cụm mà bạn có thể chọn controller phù hợp nhất. K3s đã cài sẵn [Traefik](https://containo.us/traefik/). Các lựa chọn khác gồm Istio và Nginx Ingress Controller, [xem thêm tại đây](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

Chuyển sang Ingress sẽ yêu cầu tạo một tài nguyên Ingress. Ingress sẽ định tuyến lưu lượng đến _Service_, nhưng Service kiểu _NodePort_ cũ sẽ không dùng được.

```shell
$ kubectl delete -f manifests/service.yaml
  service "hashresponse-svc" đã bị xóa
```

Một Service kiểu [ClusterIP](https://kubernetes.io/docs/concepts/services-networking/service/#type-clusterip) sẽ cấp cho Service một IP nội bộ chỉ truy cập được trong cụm.

Sau đây sẽ cho phép lưu lượng TCP từ cổng 2345 đến cổng 3000.

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hashresponse-svc
spec:
  type: ClusterIP
  selector:
    app: hashresponse
  ports:
    - port: 2345
      protocol: TCP
      targetPort: 3000
```

Tài nguyên thứ hai chúng ta cần là _Ingress_ mới.

1. Khai báo đây là Ingress
2. Định tuyến tất cả lưu lượng đến service của chúng ta

**ingress.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dwk-material-ingress
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hashresponse-svc
                port:
                  number: 2345
```

Sau đó chúng ta có thể apply tất cả và xem kết quả

```shell
$ kubectl apply -f manifests/
  ingress.networking.k8s.io/dwk-material-ingress được tạo
  service/hashresponse-svc đã được cấu hình

$ kubectl get svc,ing
  NAME                       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
  service/kubernetes         ClusterIP   10.43.0.1    <none>        443/TCP    5m22s
  service/hashresponse-svc   ClusterIP   10.43.0.61   <none>        2345/TCP   45s

  NAME                                             CLASS    HOSTS   ADDRESS                            PORTS   AGE
  ingress.networking.k8s.io/dwk-material-ingress   <none>   *       172.21.0.3,172.21.0.4,172.21.0.5   80      16s
```

Chúng ta có thể thấy ingress đang lắng nghe ở cổng 80. Vì chúng ta đã mở cổng này nên có thể truy cập ứng dụng tại http://localhost:8081.

<exercise name='Exercise 1.07: External access with Ingress'>

Ứng dụng "Log output" hiện tại ghi ra log một timestamp và một chuỗi ngẫu nhiên.

Thêm một endpoint để lấy trạng thái hiện tại (timestamp và chuỗi) và một ingress để có thể truy cập nó bằng trình duyệt.

Bạn chỉ cần lưu chuỗi và timestamp vào bộ nhớ.

</exercise>

<exercise name='Exercise 1.08: Project v0.5'>

Chuyển sang sử dụng Ingress thay vì NodePort để truy cập project. Bạn có thể xóa ingress của ứng dụng "Log output" để tránh gây nhiễu cho bài tập này. Chúng ta sẽ tìm hiểu thêm về đường dẫn và định tuyến ở bài tiếp theo và khi đó bạn có thể cấu hình project chạy song song với ứng dụng "Log output".

</exercise>

<exercise name='Exercise 1.09: More services'>

Phát triển một ứng dụng thứ hai chỉ đơn giản trả về "pong 0" cho một yêu cầu GET và tăng bộ đếm (số 0) để bạn có thể thấy đã gửi bao nhiêu yêu cầu. Bộ đếm nên lưu trong bộ nhớ nên có thể bị reset bất cứ lúc nào.
Tạo một deployment mới cho nó và cho nó dùng chung ingress với ứng dụng "Log output". Định tuyến các yêu cầu gửi đến '/pingpong' cho ứng dụng này.

Trong các bài tập sau, ứng dụng thứ hai này sẽ được gọi là "ứng dụng ping-pong". Nó sẽ được sử dụng cùng với ứng dụng "Log output".

Ứng dụng ping-pong sẽ cần lắng nghe các yêu cầu tại '/pingpong', vì vậy bạn có thể phải thay đổi mã nguồn của nó. Điều này có thể tránh được bằng cách cấu hình ingress để viết lại đường dẫn, nhưng chúng ta sẽ để đó như một bài tập tùy chọn. Bạn có thể tham khảo https://kubernetes.io/docs/concepts/services-networking/ingress/#the-ingress-resource

</exercise>

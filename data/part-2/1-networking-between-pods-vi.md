---
path: "/part-2/1-networking-between-pods-vi"
title: "Kết nối mạng giữa các pod"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này

- Bạn biết về DNS của Kubernetes

- Gửi yêu cầu từ một pod đến một pod khác trong cùng cụm

</text-box>

Ở phần 1, chúng ta đã cấu hình mạng để cho phép định tuyến lưu lượng từ bên ngoài cụm vào một container bên trong pod. Ở Phần 2, chúng ta sẽ tập trung vào giao tiếp giữa các pod.

Kubernetes bao gồm một dịch vụ DNS nên việc giao tiếp giữa các pod và container trong Kubernetes khá giống với các container trong Docker compose. Các container trong một pod chia sẻ cùng mạng. Do đó, mọi container khác trong cùng một pod đều có thể truy cập qua `localhost`. Để giao tiếp giữa các Pod, một _Service_ được sử dụng vì nó sẽ expose các Pod như một dịch vụ mạng.

Service sau đây, lấy từ một bài tập ở phần trước, tạo một IP nội bộ cụm cho phép các pod khác trong cụm truy cập cổng 3000 của ứng dụng _todo-backend_ tại http://todo-backend-svc:2345.

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: todo-backend-svc
spec:
  type: ClusterIP
  selector:
    app: todo-backend
  ports:
    - port: 2345
      protocol: TCP
      targetPort: 3000
```

Ngoài ra, mỗi Pod cũng có một địa chỉ IP được Kubernetes tạo ra.

### Một pod để debug

Đôi khi, cách tốt nhất để debug là kiểm tra thủ công xem điều gì đang xảy ra. Bạn có thể vào bên trong một pod hoặc gửi yêu cầu thủ công từ một pod khác. Bạn có thể sử dụng [busybox](https://en.wikipedia.org/wiki/BusyBox), một bản phân phối Linux nhẹ dùng cho debug.

Hãy khởi động một pod busybox bằng cách apply file yaml sau:

**podfordebugging.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-busybox
  labels:
    app: my-busybox
spec:
  containers:
    - image: busybox
      command:
        - sleep
        - "3600"
      imagePullPolicy: IfNotPresent
      name: busybox
  restartPolicy: Always
```

Bây giờ chúng ta có thể thực thi một lệnh:

```bash
$ kubectl exec -it my-busybox -- wget -qO - http://todo-backend-svc:2345
```

Chúng ta sử dụng lệnh [wget](https://www.gnu.org/software/wget/) vì công cụ thường dùng là curl không có sẵn trong busybox.

Chúng ta cũng có thể mở một shell vào pod với lệnh [kubectl exec](https://kubernetes.io/docs/tasks/debug/debug-application/get-shell-running-container/) để chạy nhiều lệnh:

```yaml
$ kubectl exec -it my-busybox -- sh
/ # wget -qO - http://todo-backend-svc:2345
<html>
   <body>
      <h1>Kube todo</h1>
      <img src="/picture.jpg" alt="rndpic" width="200" height="200">

      <form action="/create-todo" method="POST">
         <input type="text" name="todo" maxlength="140" placeholder="Enter todo">
         <button type="submit">Create</button>
      </form>

      <ul>
            <li>Mua sữa</li>
            <li>Gửi thư</li>
            <li>Thanh toán hóa đơn</li>
      </ul>
   </body>
</html>/ #
$ exit
```

Bạn cũng có thể nhận kết quả tương tự bằng cách sử dụng địa chỉ IP nội bộ của service. Hãy kiểm tra địa chỉ:

```bash
$ kubectl get svc
NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
todo-backend-svc   ClusterIP   10.43.89.182   <none>        2345/TCP   2d1h
```

Lệnh sau sẽ hiển thị trang chính của ứng dụng todo-app:

```bash
$ kubectl exec -it my-busybox -- wget -qO - http://10.43.89.182:2345
```

Hãy thử truy cập trực tiếp một pod. Chúng ta có thể lấy địa chỉ IP bằng lệnh _kubectl describe_:

```bash
$ kubectl describe pod todo-backend-dep-84fcdff4cc-2x9wl
Name:             todo-backend-dep-84fcdff4cc-2x9wl
Namespace:        default
Priority:         0
Service Account:  default
Node:             k3d-k3s-default-agent-0/192.168.176.5
Start Time:       Mon, 08 Apr 2024 23:27:00 +0300
Labels:           app=todo-backend
                  pod-template-hash=84fcdff4cc
Annotations:      <none>
Status:           Running
IP:               10.42.0.63
```

Vậy một trong các pod đang chạy ứng dụng todo-app có địa chỉ IP nội bộ cụm là _10.42.0.63_. Chúng ta có thể wget đến địa chỉ này, nhưng lần này cần nhớ rằng cổng trong pod là 3000:

```bash
$ kubectl exec -it my-busybox wget -qO - http://10.42.0.63:3000
```

Lưu ý rằng khác với phần trước, lần này chúng ta đã tạo một pod độc lập trong cụm, không có đối tượng deployment nào cả. Khi xong, bạn nên xóa pod này, ví dụ bằng lệnh

```yaml
$ kubectl delete pod/my-busybox
```

Nói chung, các pod "độc lập" kiểu này chỉ nên dùng để debug, còn các pod ứng dụng nên được tạo qua deployment. Lý do là nếu node chứa pod bị crash, các pod độc lập sẽ biến mất. Khi pod được quản lý bởi deployment, Kubernetes sẽ tự động triển khai lại nếu node gặp sự cố.

</text-box>

<exercise name='Exercise 2.01: Kết nối các pod'>

Kết nối ứng dụng "Log output" và ứng dụng "Ping-pong". Thay vì chia sẻ dữ liệu qua file, hãy sử dụng các HTTP endpoint để trả về số lượng pong. Tạm thời loại bỏ tất cả các volume giữa hai ứng dụng.

Kết quả đầu ra sẽ giữ nguyên:

```
2020-03-30T12:15:17.705Z: 8523ecb1-c716-4cb6-a044-b9e83bb98e43.
Ping / Pongs: 3
```

</exercise>

<exercise name='Exercise 2.02: Project v1.0'>

Hãy quay lại Project của chúng ta. Ở [phần trước](/part-1/4-introduction-to-storage) chúng ta đã thêm một ảnh ngẫu nhiên và một form để tạo todo vào ứng dụng. Bước tiếp theo là tạo một container mới để xử lý việc lưu các todo.

Dịch vụ mới này, hãy gọi là todo-backend, nên có endpoint GET /todos để lấy danh sách todo và endpoint POST /todos để tạo một todo mới. Các todo có thể lưu vào bộ nhớ, sau này chúng ta sẽ thêm cơ sở dữ liệu.

Sử dụng ingress routing để cho phép truy cập vào todo-backend.

Sau bài tập này, project của bạn sẽ trông như sau:

<img src="../img/p2-2.png">

Vai trò của dịch vụ mà chúng ta đã tạo ở các bài tập trước (Todo-app trong hình) là phục vụ HTML và có thể cả JavaScript cho trình duyệt. Ngoài ra, logic phục vụ ảnh ngẫu nhiên và cache ảnh vẫn nằm ở dịch vụ này.

Dịch vụ mới sẽ xử lý các mục todo.

Sau bài tập này, bạn sẽ có thể tạo các todo mới bằng form, và các todo đã tạo sẽ được hiển thị trên trình duyệt.

</exercise>

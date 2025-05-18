---
path: "/part-1/4-introduction-to-storage-vi"
title: "Giới thiệu về Lưu trữ"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn

- có thể sử dụng volume để chia sẻ dữ liệu giữa hai container trong một pod

- biết về persistent volume để lưu dữ liệu trên ổ đĩa của một node

</text-box>

Có hai điều được biết là khó khăn với Kubernetes. Đầu tiên là mạng. May mắn thay, chúng ta có thể tránh được hầu hết các khó khăn về mạng trừ khi bạn muốn tự thiết lập một cụm riêng. Nếu bạn quan tâm, bạn có thể xem Webinar "[Kubernetes and Networks: Why is This So Dang Hard?](https://www.youtube.com/watch?v=GgCA2USI5iQ)" nhưng chúng ta sẽ bỏ qua hầu hết các chủ đề được thảo luận trong video. Điều khó khăn còn lại là **lưu trữ**.

Trong phần 1, chúng ta sẽ tìm hiểu một phương pháp rất cơ bản để sử dụng lưu trữ và sẽ quay lại chủ đề này sau. Trong khi hầu hết mọi thứ khác trong Kubernetes đều rất linh hoạt, có thể di chuyển giữa các node và sao chép dễ dàng, thì lưu trữ lại không có những khả năng như vậy. "[Why Is Storage On Kubernetes So Hard?](https://softwareengineeringdaily.com/2019/01/11/why-is-storage-on-kubernetes-is-so-hard/)" cung cấp cho chúng ta một góc nhìn rộng về những khó khăn và các lựa chọn khác nhau để vượt qua chúng.

Có nhiều loại volume khác nhau và chúng ta sẽ bắt đầu với hai loại.

### Volume

Một [volume](https://docs.docker.com/storage/volumes/) trong Docker và Docker compose là cách để lưu trữ dữ liệu mà các container sử dụng. Với Kubernetes, [các volume đơn giản](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/) lại không hoàn toàn như vậy.

Các volume trong Kubernetes, về mặt kỹ thuật là _emptyDir_, là các hệ thống file chia sẻ _bên trong một pod_, nghĩa là vòng đời của chúng gắn liền với pod. Khi pod bị xóa thì dữ liệu cũng mất. Ngoài ra, chỉ cần di chuyển pod sang node khác cũng sẽ làm mất dữ liệu trong volume vì không gian lưu trữ được lấy từ node mà pod đang chạy. Vì vậy, bạn chắc chắn không nên dùng [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) để sao lưu cơ sở dữ liệu. Dù có hạn chế, nó vẫn có thể dùng làm cache vì dữ liệu tồn tại giữa các lần khởi động lại container hoặc dùng để chia sẻ file giữa hai container trong một pod.

Trước khi bắt đầu, chúng ta cần một ứng dụng chia sẻ dữ liệu với ứng dụng khác. Trong trường hợp này, nó sẽ hoạt động như một phương pháp để chia sẻ file log đơn giản với nhau. Chúng ta cần phát triển:

- Ứng dụng 1 sẽ kiểm tra xem /usr/src/app/files/image.jpg có tồn tại không, nếu không thì tải một ảnh ngẫu nhiên và lưu thành image.jpg. Bất kỳ yêu cầu HTTP nào cũng sẽ tạo ảnh mới.
- Ứng dụng 2 sẽ kiểm tra file /usr/src/app/files/image.jpg và hiển thị nó nếu có.

Hai ứng dụng này dùng chung một deployment để cả hai **nằm trong cùng một pod**. Phiên bản mẫu của tôi có sẵn cho bạn dùng [tại đây](https://github.com/kubernetes-hy/material-example/blob/b9ff709b4af7ca13643635e07df7367b54f5c575/app3/manifests/deployment.yaml). Ví dụ này bao gồm cả ingress và service để truy cập ứng dụng.

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: images-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: images
  template:
    metadata:
      labels:
        app: images
    spec:
      volumes: # Định nghĩa volume
        - name: shared-image
          emptyDir: {}
      containers:
        - name: image-finder
          image: jakousa/dwk-app3-image-finder:b7fc18de2376da80ff0cfc72cf581a9f94d10e64
          volumeMounts: # Gắn volume
            - name: shared-image
              mountPath: /usr/src/app/files
        - name: image-response
          image: jakousa/dwk-app3-image-response:b7fc18de2376da80ff0cfc72cf581a9f94d10e64
          volumeMounts: # Gắn volume
            - name: shared-image
              mountPath: /usr/src/app/files
```

Vì việc hiển thị phụ thuộc vào volume nên chúng ta có thể xác nhận nó hoạt động bằng cách truy cập image-response và lấy ảnh. Ingress cung cấp sẵn sử dụng cổng 8081 đã mở <http://localhost:8081>

Lưu ý rằng tất cả dữ liệu sẽ mất khi pod bị tắt.

<exercise name='Exercise 1.10: Even more services'>

Tách ứng dụng "Log output" thành hai container khác nhau trong cùng một pod:

Một container tạo timestamp mới mỗi 5 giây và lưu vào file.

Container còn lại đọc file đó và xuất ra cùng với hash cho người dùng xem.

Bất kỳ ứng dụng nào cũng có thể tạo hash, có thể là reader hoặc writer.

Bạn có thể thấy [hướng dẫn này](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/) hữu ích vì bây giờ có nhiều hơn một container chạy trong pod.

</exercise>

### Persistent Volume

Khác với emptyDir, một [Persistent Volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) (PV) là thứ mà bạn có thể nghĩ đến khi nói về volume.

Persistent Volume (PV) là tài nguyên ở cấp cụm, đại diện cho một phần lưu trữ trong cụm được quản trị viên cung cấp hoặc được [tạo động](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/). PV có thể dựa trên nhiều loại lưu trữ như ổ đĩa cục bộ, NFS, lưu trữ đám mây, v.v.

PV có vòng đời độc lập với bất kỳ pod nào sử dụng nó. Điều này nghĩa là dữ liệu trong PV có thể tồn tại lâu hơn pod đã gắn nó.

Khi sử dụng nhà cung cấp đám mây như Google Kubernetes Engine (chúng ta sẽ dùng ở phần 3 và 4), nhà cung cấp đám mây sẽ lo việc lưu trữ và các Persistent Volume mà bạn có thể dùng. Nếu bạn tự vận hành cụm hoặc dùng cụm cục bộ như k3s để phát triển, bạn cần tự lo hệ thống lưu trữ và Persistent Volume.

Một lựa chọn dễ dàng với K3s là [local](https://kubernetes.io/docs/concepts/storage/volumes/#local) PersistentVolume sử dụng một đường dẫn trên node làm nơi lưu trữ. Cách này gắn volume với một node cụ thể và nếu node đó không khả dụng thì lưu trữ cũng không dùng được.

Vì vậy local Persistent Volume **không** phải là giải pháp cho môi trường production!

Để _PersistentVolume_ hoạt động, trước tiên bạn cần tạo đường dẫn local trên node mà bạn sẽ gắn nó vào. Vì cụm của chúng ta chạy qua Docker nên hãy tạo thư mục `/tmp/kube` trong container `k3d-k3s-default-agent-0`. Có thể thực hiện đơn giản bằng lệnh `docker exec k3d-k3s-default-agent-0 mkdir -p /tmp/kube`

Định nghĩa Persistent Volume như sau:

**persistentvolume.yaml**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  storageClassName: my-example-pv # tên này sẽ dùng để claim volume sau
  capacity:
    storage: 1Gi # Có thể là 500Gi. Dùng dung lượng nhỏ để tiết kiệm không gian khi test cục bộ
  volumeMode: Filesystem # Khai báo sẽ mount vào pod như một thư mục
  accessModes:
    - ReadWriteOnce
  local:
    path: /tmp/kube
  nodeAffinity: ## Chỉ cần cho local, xác định node nào có thể truy cập
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - k3d-k3s-default-agent-0
```

[Persistent Volume Claim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) (PVC) là yêu cầu lưu trữ của người dùng.

Khi người dùng tạo PVC, Kubernetes sẽ tìm PV phù hợp với yêu cầu và gán chúng với nhau. Nếu không có PV phù hợp, tùy cấu hình, cụm có thể tự động tạo PV đáp ứng yêu cầu.

Khi đã gán, PersistentVolumeClaim sẽ "khóa" và chỉ được dùng bởi một Pod (tùy access mode). Điều này đảm bảo tài nguyên PV chỉ được pod đó sử dụng.

Nếu không có Persistent Volume phù hợp, PVC sẽ ở trạng thái "Pending" chờ PV phù hợp.

Về mặt khái niệm, bạn có thể coi PV là ổ đĩa vật lý (lưu trữ thực tế), còn PVC là cách pod yêu cầu sử dụng lưu trữ đó.

Bây giờ hãy tạo một claim cho ứng dụng:

**persistentvolumeclaim.yaml**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-claim # tên của volume claim, sẽ dùng trong deployment
spec:
  storageClassName: my-example-pv # tên của persistent volume mà chúng ta claim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Chỉnh sửa deployment đã giới thiệu trước đó để sử dụng nó:

**deployment.yaml**

```yaml
# ...
spec:
  volumes:
    - name: shared-image
      persistentVolumeClaim:
        claimName: image-claim
  containers:
    - name: image-finder
      image: jakousa/dwk-app3-image-finder:b7fc18de2376da80ff0cfc72cf581a9f94d10e64
      volumeMounts:
        - name: shared-image
          mountPath: /usr/src/app/files
    - name: image-response
      image: jakousa/dwk-app3-image-response:b7fc18de2376da80ff0cfc72cf581a9f94d10e64
      volumeMounts:
        - name: shared-image
          mountPath: /usr/src/app/files
```

Và apply cùng với persistentvolume.yaml và persistentvolumeclaim.yaml.

```console
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app3/manifests/deployment-persistent.yaml
```

Với service và ingress trước đó, chúng ta có thể truy cập ứng dụng tại http://localhost:8081. Để xác nhận dữ liệu được lưu bền, hãy chạy

```console
$ kubectl delete -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app3/manifests/deployment-persistent.yaml
  deployment.apps "images-dep" đã bị xóa
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app3/manifests/deployment-persistent.yaml
  deployment.apps/images-dep đã được tạo
```

File vẫn còn nguyên!

Nếu bạn muốn tìm hiểu thêm về việc tự vận hành lưu trữ, có thể tham khảo:

- [Rook](https://rook.io/)

- [OpenEBS](https://openebs.io/)

- [Longhorn](https://longhorn.io/)

<exercise name='Exercise 1.11: Persisting data'>

Hãy chia sẻ dữ liệu giữa hai ứng dụng "Ping-pong" và "Log output" bằng persistent volume. Tạo cả _PersistentVolume_ và _PersistentVolumeClaim_ rồi chỉnh sửa _Deployment_ để sử dụng chúng. Vì PersistentVolume thường do quản trị viên cụm quản lý và không gắn với ứng dụng cụ thể nên bạn nên để định nghĩa của chúng riêng biệt, có thể trong một thư mục riêng.

Lưu số lượng request gửi đến ứng dụng "Ping-pong" vào một file trong volume và xuất ra cùng với timestamp và hash khi gửi request đến ứng dụng "Log output". Cuối cùng, hai pod nên chia sẻ một persistent volume giữa hai ứng dụng. Khi truy cập ứng dụng "Log output" trên trình duyệt, bạn sẽ thấy:

```plaintext
2020-03-30T12:15:17.705Z: 8523ecb1-c716-4cb6-a044-b9e83bb98e43.
Ping / Pongs: 3
```

</exercise>

<exercise name='Exercise 1.12: Project v0.6'>

Vì project hiện tại khá nhàm chán, hãy thêm một bức ảnh!

Mục tiêu là thêm một ảnh theo giờ vào project.

Lấy một ảnh ngẫu nhiên từ Lorem Picsum như `https://picsum.photos/1200` và hiển thị trong project. Tìm cách lưu ảnh để giữ nguyên trong 60 phút.

Đảm bảo cache ảnh vào volume để API không bị gọi mỗi lần truy cập ứng dụng hoặc container bị crash.

Cách tốt nhất để kiểm tra khi container tắt là tắt container, bạn cũng có thể thêm logic cho việc này để kiểm thử.

</exercise>

<exercise name='Exercise 1.13: Project v0.7'>

Đối với project, chúng ta sẽ cần code thêm để thấy kết quả ở phần tiếp theo.

1. Thêm một trường nhập liệu. Trường này không nhận todo dài quá 140 ký tự.

2. Thêm nút gửi. Chưa cần gửi todo ngay.

3. Thêm danh sách các todo hiện có với một số todo mẫu cứng.

Có thể giống như sau:
<img src="../img/project-ex-113.png">

</exercise>

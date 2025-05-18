---
path: "/part-5/3-service-mesh-vi"
title: "Service Mesh"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này, bạn có thể

- Thiết lập một service mesh và sử dụng nó để giám sát lưu lượng mạng

</text-box>

Bạn sẽ thường xuyên nghe về một khái niệm gọi là _Service Mesh_. Service mesh là một hệ thống khá phức tạp, cung cấp rất nhiều tính năng cho ứng dụng. Trong các phần 1 đến 4, chúng ta đã triển khai một số tính năng mà service mesh có thể cung cấp sẵn. Video sau đây của Microsoft Developer là một hướng dẫn tuyệt vời về tất cả các tính năng mà một service mesh có.

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/izVWk7rYqWI" frameborder="0" allow="accelerometer; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Bạn cũng nên đọc bài viết [What is a service mesh?](https://linkerd.io/what-is-a-service-mesh/) của Linkerd.

Đối với lưu lượng vào, ra và giao tiếp giữa các dịch vụ, một service mesh có thể:

- bảo mật giao tiếp
- quản lý lưu lượng
- giám sát lưu lượng, gửi log và metric tới ví dụ như Prometheus

Vì vậy, service mesh là một công cụ **cực kỳ** mạnh mẽ. Nếu chúng ta dùng service mesh như [Istio](https://istio.io/) ngay từ phần 1, có thể đã không cần dùng [Traefik](https://doc.traefik.io/traefik/providers/kubernetes-ingress/), không cần tự làm giải pháp monitoring, và có thể thực hiện canary release mà không cần [Argo Rollouts](https://argoproj.github.io/rollouts/). Tuy nhiên, chúng ta đã làm được tất cả mà không cần service mesh.

Hãy cài đặt một service mesh và thử nghiệm các tính năng. Ở đây, chúng ta chọn [Linkerd](https://linkerd.io/) chủ yếu vì nó nhẹ hơn [Istio](https://istio.io/).

Linkerd có một công cụ CLI hỗ trợ, hãy làm theo [hướng dẫn bắt đầu](https://linkerd.io/2/getting-started/) đến Bước 4.

<text-box name="Nguồn tham khảo khác" variant="hint">
 Thực ra chúng ta chỉ đang làm theo toàn bộ hướng dẫn bắt đầu, bạn có thể đọc qua nếu muốn.
</text-box>

Hãy xem ứng dụng của chúng ta, lần này là ứng dụng microservice cho việc bình chọn emoji: [https://github.com/BuoyantIO/emojivoto](https://github.com/BuoyantIO/emojivoto).

```console
$ kubectl apply -f https://raw.githubusercontent.com/BuoyantIO/emojivoto/main/kustomize/deployment/ns.yml \
                -f https://raw.githubusercontent.com/BuoyantIO/emojivoto/main/kustomize/deployment/web.yml \
                -f https://raw.githubusercontent.com/BuoyantIO/emojivoto/main/kustomize/deployment/emoji.yml \
                -f https://raw.githubusercontent.com/BuoyantIO/emojivoto/main/kustomize/deployment/voting.yml \
                -f https://raw.githubusercontent.com/BuoyantIO/emojivoto/main/kustomize/deployment/vote-bot.yml

$ kubectl get all -n emojivoto
  NAME                            READY   STATUS    RESTARTS   AGE
  pod/web-7bcb54cb8b-cjw7d        1/1     Running   1          3d21h
  pod/emoji-686f74d889-rcdsh      1/1     Running   1          3d21h
  pod/vote-bot-74d97c76c6-pcsfl   1/1     Running   1          3d21h
  pod/voting-56847f699b-2nzqn     1/1     Running   1          3d21h

  NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
  service/web-svc      ClusterIP   10.43.248.111   <none>        80/TCP              3d21h
  service/emoji-svc    ClusterIP   10.43.110.235   <none>        8080/TCP,8801/TCP   3d21h
  service/voting-svc   ClusterIP   10.43.111.57    <none>        8080/TCP,8801/TCP   3d21h

  NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/emoji      1/1     1            1           3d21h
  deployment.apps/vote-bot   1/1     1            1           3d21h
  deployment.apps/web        1/1     1            1           3d21h
  deployment.apps/voting     1/1     1            1           3d21h
```

Ở đây chúng ta thấy deployment "vote-bot" tự động tạo lưu lượng truy cập. README cho biết nó sẽ bình chọn Donut 🍩 15% thời gian và phần còn lại chọn ngẫu nhiên.

Vì đã có service, chúng ta chỉ thiếu ingress.

**ingress.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: emojivoto
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
```

Và ứng dụng sẽ có thể truy cập tại [http://localhost:8081](http://localhost:8081). Tuy nhiên, có điều gì đó lạ đang xảy ra! Bạn có thể phát hiện ra bằng cách xem bảng xếp hạng và biết các lượt bình chọn đi đâu, hoặc tự mình nhấn vào từng emoji.

Hãy xem có cách nào tốt hơn để phát hiện hành vi này và tìm ra vấn đề. Linkerd cung cấp một dashboard thông qua extension tên là `viz`.

```
$ linkerd viz install | kubectl apply -f -
  ...

$ linkerd viz dashboard
```

Nó sẽ mở trình duyệt của bạn. Nhấn vào namespace "emojivoto" (để vào /namespaces/emojivoto) bạn sẽ thấy các resource trong namespace emojivoto chưa nằm trong service mesh. Nguyên nhân là các pod chưa có `sidecar container`. [Sidecar containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/) là một mẫu phổ biến, nơi một container mới được thêm vào pod để bổ sung chức năng. Hãy thêm sidecar của Linkerd vào emojivoto.

Trạng thái pod trước khi thêm:

```
$ kubectl get po -n emojivoto
  NAME                        READY   STATUS    RESTARTS   AGE
  voting-f999bd4d7-r4mct      1/1     Running   0          10m
  web-79469b946f-ksprv        1/1     Running   0          10m
  emoji-66ccdb4d86-rhcnf      1/1     Running   0          10m
  vote-bot-69754c864f-g24jt   1/1     Running   0          10m
```

Câu lệnh để thêm Linkerd vào các deployment và apply lại:

```
$ kubectl get -n emojivoto deploy -o yaml \
    | linkerd inject - \
    | kubectl apply -f -
```

Bạn có thể chạy từng lệnh riêng để xem chúng làm gì. Lệnh đầu tiên, `kubectl get -n emojivoto deploy -o yaml`, sẽ xuất ra toàn bộ deployment trong namespace emojivoto. `linkerd inject -` sẽ thêm annotation để Linkerd thêm sidecar proxy container. Cuối cùng, _kubectl apply_ sẽ áp dụng các deployment đã chỉnh sửa. Bây giờ các pod sẽ như sau:

```
kubectl get po -n emojivoto
NAME                        READY   STATUS    RESTARTS   AGE
vote-bot-6d7677bb68-qxfx9   2/2     Running   0          3m17s
web-5f86686c4d-qgxtv        2/2     Running   0          3m17s
emoji-696d9d8f95-sgzqs      2/2     Running   0          3m17s
voting-ff4c54b8d-sf99j      2/2     Running   0          3m18s
```

Nếu bạn xem lại dashboard, sẽ thấy nhiều thông tin hơn vì các deployment cũ đã được thay thế bằng các deployment đã "meshed". Ta cũng nhận thấy tỷ lệ thành công (success rate) không hoàn hảo.

Hai service có tỷ lệ thành công dưới 100%. Vì _web_ có thể chỉ chuyển tiếp lỗi từ _voting_, bạn có thể nhấn vào bất kỳ service nào để nhanh chóng thấy request nào đang lỗi.

Service mesh là công cụ mạnh mẽ vì giúp bạn kết nối và quan sát các dịch vụ. Đọc tiếp
[bài này](https://linkerd.io/2.15/tasks/debugging-your-service/) để xem Linkerd có thể dùng để debug lỗi trong ứng dụng emoji voting như thế nào.

<exercise name='Bài tập 5.02: Project, phiên bản Service Mesh'>

Kích hoạt service mesh Linkerd cho _Project_.

Việc chuyển deployment sang Linkerd khá đơn giản. Đọc [hướng dẫn này](https://linkerd.io/2/tasks/adding-your-service/), và thêm các manifest đã chỉnh sửa (qua Linkerd inject) vào repository để nộp bài.

</exercise>

<exercise name='Bài tập 5.03: Học từ tài liệu ngoài'>

Để minh họa cách canary release hoạt động trong Service Mesh, hãy làm theo bài tập tại: https://linkerd.io/2/tasks/canary-release/

Chỉ cần làm theo một ví dụ [Flagger](https://linkerd.io/2.15/tasks/flagger/#flagger) hoặc [Argo Rollouts](https://linkerd.io/2.15/tasks/flagger/#argo-rollouts), mà bạn đã quen ở [phần 4](/part-4/1-update-strategies-and-prometheus#canary-release).

Sử dụng lệnh <a href="https://man7.org/linux/man-pages/man1/script.1.html">script</a> trong quá trình làm bài để có file nộp. Hoặc chỉ cần chụp màn hình kết quả cuối cùng.

</exercise>

Ok, chúng ta tạm dừng ở đây. Bạn có cần service mesh cho ứng dụng của mình không? Phần lớn là không... trừ khi bạn làm ở môi trường doanh nghiệp lớn.

### Một điều nữa... init containers và sidecars

Bên dưới, Linkerd dựa rất nhiều vào Init containers và Sidecar containers để thực hiện "phép thuật" của mình.
Hãy cùng tìm hiểu kỹ hơn về hai khái niệm quan trọng này.

Một pod có thể có bất kỳ số lượng [init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) nào, là các container chạy trước khi container chính của pod khởi động. Có nhiều mục đích cho init container, ví dụ:

- tạo hoặc chỉnh sửa file cấu hình, lấy dữ liệu hoặc cấu hình từ nguồn ngoài, hoặc thực hiện các bước tiền xử lý cần thiết trước khi ứng dụng chính khởi động
- chờ các dịch vụ, database hoặc thành phần hạ tầng khác sẵn sàng trước khi ứng dụng chính khởi động. Điều này đảm bảo container chính chỉ chạy khi mọi phụ thuộc đã sẵn sàng
- cài đặt hoặc thiết lập các tiện ích, toolchain hoặc phần mềm cần thiết cho ứng dụng chính lúc runtime nhưng không có sẵn trong image chính, giúp image chính gọn nhẹ và tối ưu

[Sidecar](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/) là các container phụ chạy cùng container chính trong cùng một Pod. Chúng được dùng để bổ sung hoặc mở rộng chức năng cho container chính bằng cách cung cấp thêm dịch vụ, như logging, monitoring, bảo mật, đồng bộ dữ liệu... mà không cần thay đổi mã nguồn ứng dụng chính.

<exercise name='Bài tập 5.04: Wikipedia với init và sidecar'>

Viết một ứng dụng phục vụ các trang Wikipedia. Ứng dụng cần có:

- container chính dựa trên image _nginx_, chỉ phục vụ nội dung có trong thư mục www công khai
- init container dùng curl để tải trang <https://en.wikipedia.org/wiki/Kubernetes> và lưu nội dung vào thư mục www công khai cho container chính
- một sidecar container chờ ngẫu nhiên từ 5 đến 15 phút, sau đó curl một trang Wikipedia ngẫu nhiên tại URL <https://en.wikipedia.org/wiki/Special:Random> và lưu nội dung vào thư mục www công khai cho container chính

Gợi ý: bạn có thể cần đọc lại [phần này](/part-1/4-introduction-to-storage#volumes) ở phần 1 của khóa học.

</exercise>

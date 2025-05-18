---
path: "/known-problems-solutions"
title: "Các vấn đề bạn có thể gặp phải"
hidden: false
information_page: true
# hide_in_sidebar: true
---

Trang này liệt kê một số vấn đề đã xảy ra trong quá trình làm theo các ví dụ và cách giải quyết. Câu nói cũ "bạn đã thử tắt đi bật lại chưa" luôn là nơi tốt để bắt đầu, nhưng nếu vẫn không được thì có lẽ ai đó cũng đã từng gặp vấn đề này.

## Triển khai đầu tiên

Có một vài điều có thể xảy ra lỗi khi tạo cụm và triển khai ứng dụng đầu tiên. Vì bạn chưa biết các công cụ debug (chúng sẽ được giới thiệu ở phần sau), bạn có thể phải hỏi trợ giúp trên kênh telegram.

Bạn có thể xóa deployment bằng lệnh `kubectl delete deployment hashgenerator-dep` và/hoặc xóa cụm bằng `k3d cluster delete` để thử lại.

Một số nguyên nhân có thể dẫn đến thất bại là

1.  không đủ dung lượng cho cụm
2.  không đủ dung lượng cho các pod.

Nếu bạn nhận được kết quả sau khi kiểm tra pod:

```console
$ kubectl get pods
  NAME                                READY   STATUS    RESTARTS   AGE
  hashgenerator-dep-cdcc6d567-jd8jr   0/1     Pending   0          43s
```

Hãy kiểm tra "Events" của pod đó bằng lệnh `describe`. Nếu các sự kiện (events) bao gồm thông báo sau thì vấn đề là do thiếu dung lượng.

```console
$ kubectl describe pod hashgenerator-dep-cdcc6d567-jd8jr
  (...)
  Events:
    Type     Reason            Age        From               Message
    ----     ------            ----       ----               -------
    Warning  FailedScheduling  <unknown>  default-scheduler  0/3 nodes are available: 3 node(s) had taint {node.kubernetes.io/disk-pressure: }, that the pod didn't tolerate.
```

Bạn có thể đọc các giải pháp tại đây https://k3d.io/v5.4.4/faq/faq/#pods-evicted-due-to-lack-of-disk-space. Tùy vào hệ điều hành mà các bước xử lý có thể khác nhau.

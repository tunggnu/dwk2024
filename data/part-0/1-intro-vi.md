---
path: "/part-0/1-intro-vi"
title: "Giới thiệu khóa học"
overview: true
hidden: false
---

### Yêu cầu tiên quyết

Người học được kỳ vọng đã hoàn thành [DevOps with Docker](https://devopswithdocker.com) hoặc có kinh nghiệm với Docker và docker compose. Ngoài ra, cần có kinh nghiệm phát triển web, ví dụ như [Full Stack Web Development](https://fullstackopen.com/en/) hoặc tương đương. Người học cần có quyền admin/superuser để hoàn thành các bài tập và ví dụ trong tài liệu trên máy tính của mình.

### Tài liệu khóa học

Tài liệu khóa học được thiết kế để đọc từng phần từ đầu đến cuối. Tài liệu chứa các bài tập, được đặt sao cho phần trước đó cung cấp đủ thông tin để giải quyết từng bài tập. Hãy làm bài tập khi bạn đọc qua tài liệu. Khi tiến xa hơn, bạn sẽ ngày càng phải tìm kiếm thông tin trên internet. Bạn cũng nên bổ sung kiến thức bằng [tài liệu chính thức](https://kubernetes.io/docs/home/)!

Tài liệu khóa học được viết trên Mac, vì vậy một số hướng dẫn có thể thiếu chi tiết dành riêng cho từng nền tảng. Nếu bạn phát hiện lỗi hoặc muốn bổ sung gì đó, hãy gửi pull request cho tài liệu khóa học. Bạn cũng có thể tạo issue qua GitHub nếu phát hiện vấn đề với tài liệu.

## Hoàn thành và chấm điểm

Để hoàn thành khóa học, hãy hoàn thành các bài tập ở phần 1-5 trước hạn chót là ngày 31 tháng 1 năm 2025.

Tổng khối lượng công việc của khóa học khoảng 100 giờ tùy theo nền tảng của bạn.

Khóa học có 5 tín chỉ ECTS. Tổng số bài tập là 49, điểm số sẽ phụ thuộc vào số lượng bài tập bạn đã nộp:

| điểm | số bài tập |
| ---- | ---------- |
| 5    | 47         |
| 4    | 42         |
| 3    | 37         |
| 2    | 32         |
| 1    | 27         |

### Bài tập

Hãy tạo một repository trên GitHub và đăng các lời giải của bạn vào các file/thư mục được sắp xếp rõ ràng. Nếu bạn cần trợ giúp sử dụng Git, hãy tham khảo [hướng dẫn](https://guides.github.com/activities/hello-world/). Đảm bảo rằng repository của bạn có thể truy cập được với tôi. Chúng tôi ưu tiên repository công khai, nhưng nếu bạn muốn giữ bí mật, bạn có thể tạo repository riêng tư và thêm [mluukkai](https://github.com/mluukkai) làm cộng tác viên.

Hầu hết các bài tập sẽ yêu cầu bạn viết mã hoặc xuất bản gì đó lên Docker Hub. Nếu bạn không chắc nên nộp gì, hãy hỏi trong kênh chat của khóa học.

Một hệ thống phát hiện đạo văn sẽ được sử dụng để kiểm tra các bài tập nộp lên GitHub. Nếu nhiều sinh viên nộp cùng một mã, vấn đề sẽ được xử lý theo [quy định về gian lận và đạo văn](https://studies.helsinki.fi/instructions/article/what-cheating-and-plagiarism) của Đại học Helsinki.

Mỗi phần có nhiều bài tập. Sau khi hoàn thành một phần, hãy sử dụng [hệ thống nộp bài](https://studies.cs.helsinki.fi/stats/courses/kubernetes2024). Lưu ý rằng bạn **không thể** chỉnh sửa bài nộp sau đó, vì vậy hãy cân nhắc trước số lượng bài tập bạn muốn hoàn thành cho mỗi phần.

## Tín dụng Google Cloud

Ở phần 3, chúng ta sẽ sử dụng Google Kubernetes Engine. Dịch vụ này không miễn phí, nhưng mỗi người bắt đầu với Google Cloud sẽ có 300 đô la tín dụng miễn phí. Xem các lựa chọn của bạn [tại đây](https://cloud.google.com/free).

Nếu bạn đã sử dụng hết 300 đô la tín dụng Google Cloud, chúng tôi không thể hỗ trợ thêm. Bạn có thể hoàn thành phần 1-2 và có thể cả phần 4-5, nhưng phần 3 là bắt buộc để hoàn thành khóa học với điểm cao hơn.

Bạn cũng có thể sử dụng nhà cung cấp đám mây khác nhưng chúng tôi không có hướng dẫn cho các nhà cung cấp khác.

## Bắt đầu

### Discord

Khóa học này có một nhóm Discord nơi chúng ta thảo luận mọi thứ về khóa học. Hỗ trợ gần như 24/7, với các cuộc thảo luận bằng cả tiếng Anh và tiếng Phần Lan.

Tham gia kênh Discord DevOps with Kubernetes: <https://study.cs.helsinki.fi/discord/join/kubernetes>.

**Mọi** bình luận không phù hợp, xúc phạm hoặc phân biệt đối xử trên kênh đều bị cấm và sẽ bị xử lý.

### Cài đặt kubectl

Kubectl là công cụ dòng lệnh mà chúng ta sẽ sử dụng để giao tiếp với cụm Kubernetes. [Hướng dẫn cài đặt](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

### Cài đặt k3d

Chúng ta cũng sẽ sử dụng k3d để thực hành. [Hướng dẫn cài đặt](https://github.com/rancher/k3d#get) ở đây. Tôi đã kiểm thử tài liệu khóa học với phiên bản k3d 5.4.1.

#### Lưu ý về lỗi phân quyền với k3d

Bạn có thể gặp lỗi `Permission denied` khi sử dụng `k3d` với người dùng thông thường.

Hãy chắc chắn đã thực hiện [bước cấu hình sau cài đặt docker](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user)

Chạy `k3d` với `sudo` sẽ dẫn đến các vấn đề như tạo kubeconfig ở sai vị trí, ví dụ không phải ở _~/.kube_.

## Lỗi và đóng góp

Bạn có phát hiện lỗi, vấn đề, lỗi chính tả, hoặc thiếu sót gì không? Có thể bạn nghĩ rằng một phần nào đó chưa được viết tốt và bạn có thể làm tốt hơn? Đang là Hacktoberfest? Hoặc bạn muốn chia sẻ một bài blog hay? Hãy đóng góp nhé!

Vì khóa học là mã nguồn mở, bạn có thể fork, chỉnh sửa và gửi pull request. Nếu bạn chưa biết fork là gì hoặc cách gửi pull request, hãy tham khảo [hướng dẫn GitHub](https://guides.github.com/activities/hello-world/). Bạn có thể thực hành tại đây.

Nếu bạn không muốn tên mình xuất hiện trong danh sách [contributors](https://github.com/kubernetes-hy/kubernetes-hy.github.io/graphs/contributors) bạn cũng có thể tạo issue. Hướng dẫn tạo issue trên GitHub [ở đây](https://help.github.com/en/articles/creating-an-issue).

Đây là liên kết tới repository để bạn tìm tab issues và pull requests: [https://github.com/kubernetes-hy/kubernetes-hy.github.io](https://github.com/kubernetes-hy/kubernetes-hy.github.io)

# AS02 — Security Misconfiguration

[OWASP Top 10](./README.md)

**Lỗi cấu hình bảo mật** xảy ra khi hệ thống, máy chủ hoặc ứng dụng được triển khai với các thiết lập mặc định không an toàn, cài đặt chưa đầy đủ, hoặc dịch vụ bị lộ ra ngoài không cần thiết. Đây không phải lỗi trong code — mà là sai sót trong cách môi trường, phần mềm hoặc mạng được thiết lập. Và chính sự phân biệt đó khiến nó dễ bị bỏ qua: developer không thấy vấn đề khi đọc code vì vấn đề không nằm trong code.

---

## Tại sao quan trọng

Stack hiện đại phức tạp — một ứng dụng production chạm đến Nginx, framework backend, database, Redis, S3, Kubernetes, CI/CD, và hàng loạt API bên thứ ba. Mỗi lớp có cấu hình riêng, và đội ngũ thường chỉ quen với một vài trong số đó. Chỉ cần một admin panel bị lộ, một bucket mở, hay quyền truy cập được cấp sai cũng có thể làm tổn hại toàn bộ hệ thống.

Ngay cả cấu hình sai nhỏ cũng có thể dẫn đến: lộ dữ liệu nhạy cảm, leo thang đặc quyền, hoặc cho attacker điểm vào mà không cần khai thác vulnerability nào.

---

## Ví dụ thực tế

**Uber 2017:** Uber để lộ bản backup AWS S3 bucket chứa dữ liệu nhạy cảm của người dùng — thông tin tài xế và hành khách — vì bucket được cấu hình public access. Attacker tải dữ liệu trực tiếp mà không cần credential. Không cần exploit, không cần bypass authentication — chỉ cần đúng URL là đủ. Đây là ví dụ điển hình cho thấy một sai sót trong deployment có thể dẫn đến compromise nghiêm trọng đến mức nào.

---

## Biểu hiện phổ biến

**Default credential chưa được đổi:** Admin panel truy cập được với `admin/admin` hay `admin/password`. Phổ biến ở router, CMS, dashboard monitoring chưa được hardening sau khi cài đặt.

**Dịch vụ và endpoint không cần thiết bị phơi ra internet:** Port 22, 3306, 5432, 6379, hay debug endpoint mở mà không có firewall. Attacker quét liên tục và thử exploit tự động.

**Cloud storage misconfiguration:** S3 bucket, Azure Blob, GCP bucket được cấp public read mà không có lý do. Hàng nghìn vụ rò rỉ dữ liệu lớn trong vài năm qua đến từ đây.

**Thiếu kiểm soát truy cập API:** API không yêu cầu authentication, không có authorization, hoặc rate limit không tồn tại — bất kỳ ai cũng có thể gọi.

**Error message lộ thông tin hệ thống:** Stack trace đầy đủ với tên class, tên file, số dòng trả về cho client. Developer mode bật trên production là nguyên nhân phổ biến nhất.

**Phần mềm, framework, container lỗi thời:** Version có CVE đã biết nhưng chưa được patch vì "đang chạy ổn". Lỗ hổng đã public → exploit tool sẵn có → bị quét tự động.

**AI/ML endpoint không có access control:** Model inference endpoint, embedding API, hay automation service được expose mà không có authentication hay rate limit — cho phép enumerate, abuse, hoặc extract thông tin từ model.

**Directory listing:** Web server trả về danh sách file khi không có `index.html`. Attacker duyệt qua để tìm file backup, config, source code bị để lộ.

---

## Ví dụ tấn công

```bash
# Phát hiện phpinfo() bị để lộ
GET /info.php

# Duyệt S3 bucket public
GET https://s3.amazonaws.com/company-backups/?list-type=2

# Admin panel với default credential
POST /admin/login
username=admin&password=admin
```

Spring Boot Actuator endpoint bị exposed:

```bash
GET /actuator/env       # lộ biến môi trường, secrets
GET /actuator/dump      # thread dump
GET /actuator/heapdump  # toàn bộ heap — có thể chứa plaintext password
```

---

## Phát hiện

```bash
# Kiểm tra security header thiếu
curl -I https://target.com | grep -E 'X-Frame|Content-Security|X-Content'

# Tìm directory listing
curl https://target.com/uploads/

# Kiểm tra Spring Boot actuator
curl https://target.com/actuator/health
curl https://target.com/actuator/env
```

Công cụ: **Nikto**, **Nuclei** (có template sẵn cho misconfiguration), **ScoutSuite** (cloud config audit), **Trivy** (container image scan).

---

## Phòng chống

Hardening là quy trình liên tục, không phải việc làm một lần.

- Tăng cường cấu hình mặc định, tắt mọi tính năng và dịch vụ không dùng trước khi deploy
- Đổi tất cả default credential ngay khi cài đặt; áp dụng xác thực mạnh và principle of least privilege trên toàn hệ thống
- Hạn chế network exposure, phân vùng tài nguyên nhạy cảm
- Luôn cập nhật phần mềm, framework và container với bản vá mới nhất
- Ẩn stack trace và thông tin hệ thống khỏi error message — trả về generic error cho client
- Audit cloud storage permission và access control định kỳ
- Đảm bảo AI/ML endpoint và automation service có authentication và monitoring phù hợp
- Tích hợp security config review và automated scan vào CI/CD pipeline — bắt misconfiguration từ sớm thay vì đến production

---

## Tham khảo

- OWASP: https://owasp.org/Top10/A05_2021-Security_Misconfiguration/
- CIS Benchmarks: https://www.cisecurity.org/cis-benchmarks

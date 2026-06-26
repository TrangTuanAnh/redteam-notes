# AS02 — Security Misconfiguration

[← OWASP Top 10](./README.md)

**Lỗi cấu hình bảo mật** — hạng mục AS02 trong OWASP Top 10 2025. Đây là loại lỗ hổng không đến từ code mà đến từ cách hệ thống được thiết lập, triển khai và duy trì. Phổ biến đến mức có thể gặp ở bất kỳ lớp nào trong stack: web server, application framework, database, container, cloud storage, hay header HTTP.

---

## Root cause

Hệ thống hiện đại có hàng trăm tùy chọn cấu hình. Giá trị mặc định thường được thiết kế cho tiện dụng khi development, không phải cho an toàn khi production. Khi không có quy trình hardening cụ thể, các default đó được giữ nguyên và trở thành bề mặt tấn công.

Ngoài ra, stack ngày càng phức tạp — một ứng dụng hiện đại có thể có Nginx, Node.js, Redis, PostgreSQL, S3, Docker, Kubernetes — mỗi lớp đều có cấu hình riêng, và đội ngũ thường chỉ quen với một vài lớp trong số đó.

---

## Biểu hiện phổ biến

**Default credentials:** Admin panel truy cập được với `admin/admin` hay `admin/password`. Phổ biến ở router, CMS, dashboard monitoring chưa được đổi mật khẩu sau khi cài đặt.

**Directory listing:** Web server trả về danh sách file khi không có `index.html`. Attacker duyệt qua để tìm file backup, config, hoặc source code bị để lộ.

**Stack trace lộ ra ngoài:** Ứng dụng trả về full exception với tên class, tên file, số dòng khi gặp lỗi. Developer mode bật trên production là nguyên nhân phổ biến.

**Security header bị thiếu:** `X-Frame-Options`, `Content-Security-Policy`, `X-Content-Type-Options` không có trong response. Mở đường cho clickjacking, XSS inline, MIME sniffing.

**Cloud storage public:** S3 bucket, Azure Blob, GCS bucket được cấp quyền public read mà không có lý do. Hàng nghìn vụ rò rỉ dữ liệu lớn trong vài năm qua đến từ đây.

**Feature không dùng vẫn bật:** Debug endpoint, Swagger UI, phpinfo(), actuator endpoint của Spring Boot — nếu không cần thì không nên expose.

**Unnecessary ports/services:** Port 22, 3306, 5432, 6379 mở ra internet mà không có firewall. Attacker quét liên tục và thử exploit tự động.

---

## Ví dụ tấn công

```
# Phát hiện phpinfo() bị để lộ
GET /info.php HTTP/1.1

# Duyệt S3 bucket public
GET https://s3.amazonaws.com/company-backups/?list-type=2

# Admin panel với default credential
POST /admin/login
username=admin&password=admin
```

Spring Boot Actuator endpoint bị exposed:

```
GET /actuator/env      # lộ biến môi trường, secrets
GET /actuator/dump     # thread dump
GET /actuator/heapdump # toàn bộ heap — có thể chứa plaintext password
```

---

## Phát hiện

```bash
# Quét header thiếu
curl -I https://target.com | grep -E 'X-Frame|Content-Security|X-Content'

# Tìm directory listing
curl https://target.com/uploads/

# Kiểm tra default Spring Boot actuator
curl https://target.com/actuator/health
curl https://target.com/actuator/env
```

Công cụ tự động: **Nikto**, **Nuclei** (có template cho misconfiguration), **ScoutSuite** (cloud), **Trivy** (container image).

---

## Phòng chống

Không có silver bullet — hardening là một quy trình, không phải một lần làm.

- Tắt mọi feature không dùng đến trước khi deploy production
- Đổi tất cả default credential ngay khi cài đặt
- Cấu hình security header ở reverse proxy hoặc middleware
- Audit cloud storage permission định kỳ — chặn public access ở cấp organization policy
- Có môi trường staging giống production để catch misconfiguration sớm
- Tự động scan misconfiguration trong CI/CD pipeline
- Review lại config khi nâng cấp version — update đôi khi reset default về giá trị mới

---

## Tham khảo

- OWASP: https://owasp.org/Top10/A05_2021-Security_Misconfiguration/
- CIS Benchmarks: https://www.cisecurity.org/cis-benchmarks

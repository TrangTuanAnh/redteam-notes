# OWASP Top 10

[← Web Attacks](../README.md)

OWASP Top 10 là danh sách 10 rủi ro bảo mật ứng dụng web nghiêm trọng nhất, được OWASP (Open Worldwide Application Security Project) công bố định kỳ dựa trên dữ liệu thực tế từ hàng trăm tổ chức và hàng nghìn ứng dụng. Phiên bản hiện hành là **2021**.

Danh sách này không phải checklist đầy đủ — nó là bản đồ rủi ro phổ biến nhất. Hiểu từng mục giúp nhận diện lỗ hổng khi đọc code, khi pentest, và khi review kiến trúc.

---

## Danh sách 2021

| # | Tên | Bài viết |
|---|---|---|
| A01 | [Broken Access Control](./A01-broken-access-control.md) | _Chưa có_ |
| A02 | [Cryptographic Failures](./A02-cryptographic-failures.md) | _Chưa có_ |
| A03 | [Injection](./A03-injection.md) | _Chưa có_ |
| A04 | [Insecure Design](./A04-insecure-design.md) | _Chưa có_ |
| A05 | [Security Misconfiguration](./A05-security-misconfiguration.md) | _Chưa có_ |
| A06 | [Vulnerable and Outdated Components](./A06-vulnerable-components.md) | _Chưa có_ |
| A07 | [Identification and Authentication Failures](./A07-auth-failures.md) | _Chưa có_ |
| A08 | [Software and Data Integrity Failures](./A08-integrity-failures.md) | _Chưa có_ |
| A09 | [Security Logging and Monitoring Failures](./A09-logging-failures.md) | _Chưa có_ |
| A10 | [Server-Side Request Forgery](./A10-ssrf.md) | _Chưa có_ |

---

## Tổng quan nhanh

**A01 — Broken Access Control** đứng đầu lần đầu tiên trong lịch sử OWASP Top 10. Ứng dụng không kiểm tra đúng xem user có quyền thực hiện hành động không — dẫn đến leo thang đặc quyền ngang (xem dữ liệu user khác) hoặc dọc (user thường làm được việc của admin).

**A02 — Cryptographic Failures** trước đây gọi là "Sensitive Data Exposure". Tên mới nhấn mạnh root cause: dùng cipher yếu, không dùng TLS, lưu plaintext, key management tệ — tất cả đều là lỗi mật mã trước khi là lỗi lộ dữ liệu.

**A03 — Injection** từng đứng số 1 nhiều năm liên tiếp. SQL injection là ví dụ kinh điển nhưng category này bao gồm OS command injection, LDAP injection, template injection và nhiều biến thể khác. Root cause chung là không tách biệt data và code.

**A04 — Insecure Design** là mục mới nhất trong danh sách 2021. Không phải lỗi implementation mà là lỗi ở tầng thiết kế — threat model sai, business logic không có kiểm soát đúng, flow không được xem xét từ góc độ tấn công.

**A05 — Security Misconfiguration** ngày càng phổ biến khi stack ngày càng phức tạp. Default credential, directory listing bật, stack trace lộ ra ngoài, cloud storage public — phần lớn là configuration để mặc định không được review.

**A06 — Vulnerable and Outdated Components** là rủi ro supply chain. Dùng library có CVE đã biết, không có process theo dõi dependency, dùng version EOL — một component lỗi thời có thể kéo sập toàn bộ ứng dụng.

**A07 — Identification and Authentication Failures** trước đây là "Broken Authentication". Mật khẩu yếu, thiếu MFA, session không expire, credential stuffing không bị chặn — ứng dụng không verify đúng "người này là ai".

**A08 — Software and Data Integrity Failures** cũng mới trong 2021, bao gồm insecure deserialization và CI/CD pipeline không được bảo vệ. Ứng dụng tin tưởng update, plugin, hoặc data từ nguồn không được verify.

**A09 — Security Logging and Monitoring Failures** thường bị coi là "không phải lỗ hổng thật" nhưng attacker biết khai thác điều này. Không có log → không phát hiện attack → attacker có thời gian dài để lateral move. Thời gian trung bình phát hiện breach vẫn đang tính bằng tháng.

**A10 — Server-Side Request Forgery** vào danh sách từ survey cộng đồng, dù tỉ lệ xuất hiện còn thấp. Khi ứng dụng fetch URL do user cung cấp mà không validate, attacker có thể đọc metadata cloud, quét nội mạng, hoặc relay request đến service nội bộ.

---

## Cấu trúc mỗi bài viết

```
## Tổng quan
## Root cause
## Cơ chế tấn công
## Ví dụ / PoC
## Phát hiện
## Phòng chống
## Tham khảo
```

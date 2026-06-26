# AS08 — Software and Data Integrity Failures

[← OWASP Top 10](./README.md)

**Software and Data Integrity Failures** xảy ra khi ứng dụng tin tưởng code, bản cập nhật, hoặc dữ liệu mà không xác minh tính toàn vẹn hay nguồn gốc của chúng. Đây là lỗ hổng xuất hiện trong hai phiên bản OWASP Top 10 gần nhất — phản ánh thực tế rằng hệ sinh thái phần mềm hiện đại phụ thuộc quá nhiều vào nguồn bên ngoài mà không có cơ chế verify đủ mạnh.

Điểm phân biệt với AS03 (Supply Chain Failures): AS03 tập trung vào dependency và thư viện bị xâm phạm; AS08 tập trung vào cơ chế verify tính toàn vẹn của artifact, data, và pipeline — tức là lỗi ở tầng trust model chứ không phải lỗi ở việc chọn dependency.

---

## Root cause

Ứng dụng tạo ra **trust assumption ngầm định** — rằng dữ liệu đến từ nguồn X là hợp lệ, rằng file đã tải về chưa bị thay đổi, rằng bản cập nhật từ server là chính thống. Khi không có cơ chế xác minh độc lập (chữ ký số, checksum, integrity check), assumption này trở thành điểm tấn công.

---

## Các kiểu lỗi phổ biến

**Tải script/config từ nguồn không đáng tin cậy mà không verify:**

```html
<!-- Tải thẳng từ CDN không có SRI check -->
<script src="https://cdn.example.com/library.js"></script>

<!-- Đúng: Subresource Integrity buộc browser verify hash -->
<script src="https://cdn.example.com/library.js"
        integrity="sha256-abc123..."
        crossorigin="anonymous"></script>
```

Nếu CDN bị xâm phạm hoặc bị MITM, script độc hại được load và thực thi mà ứng dụng không hay biết.

**Auto-update không có chữ ký số:**

Ứng dụng tự động tải và cài bản cập nhật từ server. Nếu kênh update bị intercept (DNS poisoning, MITM) hoặc server bị compromise, attacker đẩy bản cập nhật độc hại vào hàng triệu client. SolarWinds là ví dụ cực đoan của pattern này.

**Insecure deserialization:**

```python
# Dễ bị tấn công — deserialize object từ user input
import pickle
data = pickle.loads(user_supplied_bytes)  # RCE nếu payload được craft

# JSON là safe hơn nhưng vẫn cần validate schema
import json
obj = json.loads(user_input)  # Không RCE nhưng cần validate structure
```

Pickle, Java serialization, PHP `unserialize()` — tất cả đều cho phép attacker craft object độc hại để trigger RCE khi deserialize.

**CI/CD pipeline không được bảo vệ:**

```yaml
# GitHub Actions — dễ bị tấn công nếu secret không được kiểm soát
jobs:
  deploy:
    steps:
      - run: curl https://external-script.sh | bash  # Không verify source
      - run: npm install                               # Không có lockfile
```

Nếu attacker có thể inject step vào pipeline (qua PR, branch rule bypass, hoặc compromised action), họ có thể chèn code vào artifact trước khi deploy.

**Dữ liệu ảnh hưởng logic không được verify:**

```python
# JWT với thuật toán "none" — kinh điển
# Header: {"alg": "none"}
# Signature bị bỏ qua → attacker tự ký token bất kỳ

# Hoặc dữ liệu từ message queue không được xác thực nguồn gốc
message = queue.receive()
process_payment(message)  # Không verify message có bị tamper không
```

---

## Ví dụ thực tế

**SolarWinds Orion (2020):** Attacker xâm nhập build system, chèn backdoor SUNBURST vào source code trước khi compile. Binary được ký số bởi SolarWinds → trusted hoàn toàn bởi 18,000+ tổ chức. Không có integrity check nào ở phía client phát hiện được vì signature hợp lệ.

**event-stream / flatmap-stream (2018):** npm package maintainer chuyển quyền, kẻ xấu thêm dependency độc hại. Package được tải về và install tự động mà không có verify gì ngoài "package tồn tại trên registry".

**PHP Generic Gadget Chains:** Nhiều framework PHP có "gadget chains" — khi `unserialize()` nhận payload được craft, PHP tự động gọi magic method (`__wakeup`, `__destruct`) dẫn đến RCE. Đây là lý do tại sao không bao giờ deserialize user input.

---

## Phát hiện

```bash
# Tìm insecure deserialization trong Python
grep -rn "pickle\.loads\|yaml\.load(" --include="*.py" .
# yaml.load an toàn hơn khi dùng Loader=yaml.SafeLoader

# Tìm script load không có SRI
grep -rn '<script src=' --include="*.html" . | grep -v integrity

# Kiểm tra CI/CD action dùng external script không pin version
grep -rn "curl.*|.*bash\|wget.*|.*sh" .github/workflows/

# Kiểm tra npm lockfile tồn tại
ls package-lock.json yarn.lock  # Không có → dependency không được pin
```

Dấu hiệu nguy hiểm:
- Deserialize data từ client mà không validate
- CDN script load không có `integrity` attribute
- CI/CD action dùng `@latest` thay vì pin commit hash
- Update mechanism không verify chữ ký số

---

## Phòng chống

**Verify integrity của mọi artifact:**
- Dùng checksum (SHA-256) hoặc chữ ký số (GPG, Sigstore) cho package và binary
- Subresource Integrity (SRI) cho mọi script/style load từ CDN
- Pin dependency bằng lockfile (`package-lock.json`, `poetry.lock`, `Cargo.lock`)

**Không deserialize untrusted data:**
- Thay thế pickle/Java serialization bằng JSON với schema validation
- Nếu buộc phải deserialize, dùng allowlist class nghiêm ngặt
- Validate cả structure lẫn content sau khi deserialize

**Bảo vệ CI/CD pipeline:**
- Pin GitHub Action theo commit hash, không dùng `@v1` hay `@latest`
- Tách biệt quyền giữa build, test, và deploy environment
- Không chạy script từ URL không được verify trong pipeline
- Require code review cho mọi thay đổi workflow file

**Thiết lập trust boundary rõ ràng:**
- Chỉ nguồn đã được xác thực mới có thể sửa đổi component quan trọng
- Ký số artifact trước khi deploy, verify signature trước khi chạy
- Message queue và event stream cần authentication và integrity check

**Với update mechanism:**
- Dùng kênh update có chữ ký số (code signing certificate)
- Verify signature trước khi install, không chỉ sau khi download
- Staged rollout với monitoring — phát hiện anomaly trước khi toàn bộ fleet bị ảnh hưởng

---

## Tham khảo

- OWASP: https://owasp.org/Top10/A08_2021-Software_and_Data_Integrity_Failures/
- OWASP Deserialization Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html
- Subresource Integrity (MDN): https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity
- SLSA Supply Chain Levels: https://slsa.dev

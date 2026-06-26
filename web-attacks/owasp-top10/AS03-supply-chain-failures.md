# AS03 — Software Supply Chain Failures

[← OWASP Top 10](./README.md)

**Thất bại trong chuỗi cung ứng phần mềm** — hạng mục AS03 trong OWASP Top 10 2025. Trong phiên bản 2021, rủi ro này nằm chung dưới "Vulnerable and Outdated Components". Đến 2025, OWASP tách ra thành hạng mục riêng — phản ánh thực tế rằng supply chain attack đã trở thành một vector tấn công có chủ đích, không chỉ là vấn đề quản lý dependency lỗi thời.

Ứng dụng hiện đại không còn là code bạn viết. Nó là code bạn viết cộng với hàng trăm package từ npm/PyPI/Maven, build tool, CI/CD pipeline, container base image, và cloud provider. Mỗi mắt xích trong chuỗi đó là một điểm có thể bị tấn công.

---

## Root cause

Đội phát triển thường tin tưởng dependency một cách ngầm định — nếu package đã có trên registry và có nhiều download thì mặc định coi là an toàn. Không verify tính toàn vẹn, không audit source code, không theo dõi khi maintainer thay đổi.

Thêm vào đó, chuỗi dependency ngày càng sâu. Bạn install 5 package trực tiếp nhưng kéo theo hàng trăm transitive dependency. Bề mặt tấn công lớn hơn nhiều so với những gì bạn nhìn thấy trong `package.json`.

---

## Các kiểu tấn công

**Dependency confusion:** Attacker publish package lên public registry (npm, PyPI) với tên trùng với package nội bộ của công ty nhưng version số cao hơn. Package manager ưu tiên public registry → pull về package độc hại thay vì internal package.

```
# Internal package: @company/utils v1.0.0 (trên Artifactory nội bộ)
# Attacker publish: company-utils v9.9.9 (trên npmjs.com)
# npm install lấy public package vì version cao hơn
```

**Typosquatting:** Publish package tên gần giống package phổ biến. Developer gõ nhầm một ký tự → cài package độc hại.

```
requests  →  request (PyPI)
lodash    →  loadash
express   →  expres
```

**Account takeover / maintainer compromise:** Chiếm tài khoản npm/PyPI của maintainer package phổ biến, publish version mới chứa malware. Vụ `event-stream` 2018 là ví dụ kinh điển: maintainer chuyển quyền cho người lạ, người đó inject code đánh cắp crypto wallet vào version mới.

**Malicious package được publish thẳng:** Attacker tạo package mới với description hữu ích, SEO tốt, chờ developer vô tình cài. Thường nhắm vào developer tool (formatter, linter, test helper).

**Build pipeline compromise:** Tấn công vào CI/CD server thay vì package. Inject code vào artifact trong quá trình build mà source code vẫn clean. SolarWinds là ví dụ ở quy mô lớn — attacker modify binary trong build process của SUNBURST.

**Malicious container base image:** Base image trên Docker Hub có thể chứa backdoor. Nhiều image unofficial có malware được cài sẵn.

---

## Ví dụ thực tế

**event-stream (2018):** Package có 2 triệu download/tuần. Maintainer chuyển ownership, kẻ xấu thêm dependency `flatmap-stream` có chứa code obfuscated đánh cắp Bitcoin wallet. Tồn tại trong 2 tháng trước khi bị phát hiện.

**XZ Utils (2024):** Kẻ tấn công dành 2 năm build trust với project, trở thành maintainer, rồi inject backdoor vào quá trình build của version 5.6.0–5.6.1. Nhắm vào systemd để chiếm SSH trên Linux. Bị phát hiện tình cờ nhờ Andres Freund nhận thấy SSH login chậm bất thường.

**ua-parser-js (2021):** NPM account của maintainer bị chiếm, 3 version độc hại được publish trong vài giờ, chứa cryptominer và credential stealer. Package có 8 triệu download/tuần.

---

## Phát hiện

```bash
# Kiểm tra integrity với npm audit
npm audit

# Verify checksum của package (npm)
npm pack <package> && shasum -a 256 <package>.tgz

# Python: kiểm tra package với pip-audit
pip-audit

# Tìm dependency không có trong lockfile
npm ls --depth=0
```

Theo dõi:
- CVE mới cho dependency đang dùng (Dependabot, Renovate, Snyk)
- Thay đổi maintainer của package quan trọng
- Spike bất thường trong download/install time của build

---

## Phòng chống

**Lockfile nghiêm ngặt:** Commit `package-lock.json` / `poetry.lock` / `Cargo.lock`. Chỉ update có chủ đích, không để auto-update không kiểm soát.

**Verify integrity:** npm và yarn hỗ trợ `integrity` field trong lockfile (SRI hash). Không bỏ qua warning khi hash không khớp.

**Private registry với allowlist:** Chạy Artifactory hoặc Nexus, chỉ cho phép pull từ public registry qua proxy đã được scan. Giải quyết dependency confusion.

**Scope internal package:** Đặt namespace cho internal package (`@company/name`) và cấu hình registry để scope đó chỉ resolve từ internal registry.

**Tối thiểu hóa dependency:** Mỗi dependency là một rủi ro. Trước khi thêm package mới, cân nhắc xem có thể viết 20 dòng code thay thế không.

**Scan container image:** Dùng Trivy, Grype, hoặc Snyk Container để scan base image và layer trước khi deploy.

**SLSA framework:** Google's Supply-chain Levels for Software Artifacts — framework để verify provenance của artifact qua toàn bộ pipeline.

---

## Tham khảo

- OWASP: https://owasp.org/Top10/A06_2021-Vulnerable_and_Outdated_Components/
- SLSA: https://slsa.dev
- OpenSSF Scorecard: https://securityscorecards.dev

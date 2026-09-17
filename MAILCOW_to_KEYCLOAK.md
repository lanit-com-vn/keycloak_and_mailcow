# MAILCOW to KEYCLOAK

Tài liệu cô đọng cấu hình **Keycloak làm Identity Provider (SSO) cho Mailcow**, đồng thời hỗ trợ tự tạo mailbox và xác thực IMAP/SMTP bằng `mailcow_password`.

---

## 1. Mục tiêu kiến trúc

```text
                    KEYCLOAK
              User / Password / SSO
                       |
                 OpenID Connect
                       |
                       v
                    MAILCOW
          Mailbox / SOGo / IMAP / SMTP
```

Keycloak chịu trách nhiệm:

- Quản lý user.
- Đăng nhập SSO cho Mailcow/SOGo.
- Cung cấp `mailcow_template`.
- Nếu bật Mailpassword Flow: cung cấp `mailcow_password`.

Mailcow chịu trách nhiệm:

- Mail domain.
- Mailbox.
- Quota.
- Mailbox Template.
- SMTP / IMAP / POP3 / SIEVE.
- SOGo.

> **Lưu ý:** Keycloak không tự tạo Mail Domain trên Mailcow. Domain phải tồn tại trên Mailcow trước thì mailbox mới có thể được auto-create.

---

## 2. Tạo client Mailcow trên Keycloak

Trong realm cần sử dụng:

```text
Clients
→ Create client
```

Cấu hình:

```text
Client type: OpenID Connect
Client ID:   mailcow
```

### Capability config

```text
Client authentication: ON
Standard flow:         ON

Implicit flow:         OFF
Direct access grants:  OFF
```

Nếu sử dụng:

- Import Users
- Periodic Full Sync
- Mailpassword Flow

thì bật:

```text
Service account roles: ON
```

---

## 3. Login settings của client

Ví dụ Mailcow:

```text
https://mail.example.com
```

Cấu hình:

```text
Root URL:
https://mail.example.com

Home URL:
https://mail.example.com

Valid redirect URIs:
https://mail.example.com/*

Valid post logout redirect URIs:
https://mail.example.com/*

Web origins:
https://mail.example.com

Admin URL:
để trống
```

Sau khi cấu hình, vào:

```text
Clients
→ mailcow
→ Credentials
```

lấy:

```text
Client Secret
```

để điền sang Mailcow.

---

# 4. `mailcow_template` là gì?

`mailcow_template` là attribute Keycloak dùng để cho Mailcow biết user sẽ áp dụng **Mailbox Template** nào.

Ví dụ:

```text
Keycloak                           Mailcow

mailcow_template = default   →    Default
mailcow_template = basic     →    Basic
mailcow_template = business  →    Business
mailcow_template = vip       →    VIP
```

Có thể hiểu đây là cách gán "gói mailbox" cho user.

Ví dụ:

```text
Basic
- Quota 5 GB

Business
- Quota 20 GB

VIP
- Quota 100 GB
```

Quota và chính sách thực tế vẫn cấu hình ở **Mailbox Template của Mailcow**.

---

## 5. Tạo `mailcow_template` trong Keycloak User Profile

Vào:

```text
Realm settings
→ User profile
→ Create attribute
```

Cấu hình:

```text
Attribute Name:
mailcow_template

Display name:
Mailcow Template

Multivalued:
OFF

Enabled when:
Always

Required field:
OFF
```

Nếu mọi user mặc định dùng cùng một template thì có thể đặt:

```text
Default value:
default
```

Như vậy user mới sẽ mặc định có:

```text
mailcow_template = default
```

### Permission

Nên để:

```text
Admin:
View + Edit

User:
không cần Edit
```

---

# 6. Tạo Mapper cho `mailcow_template`

Vào:

```text
Clients
→ mailcow
→ Client scopes
→ mailcow-dedicated
→ Mappers
→ Configure a new mapper
→ User Attribute
```

Cấu hình:

```text
Name:
mailcow_template

User Attribute:
mailcow_template

Token Claim Name:
mailcow_template

Claim JSON Type:
String

Add to ID token:
ON

Add to access token:
ON

Add to userinfo:
ON

Multivalued:
OFF
```

Điểm quan trọng:

```text
User Attribute   = mailcow_template
Token Claim Name = mailcow_template
Add to userinfo  = ON
```

Mailcow đọc `mailcow_template` từ OIDC UserInfo.

---

# 7. Cấu hình Keycloak trên Mailcow

Vào:

```text
System
→ Configuration
→ Access
→ Identity Provider
→ Keycloak
```

Điền:

```text
Server URL:
https://login.example.com

Realm:
<realm-name>

Client ID:
mailcow

Client Secret:
<client-secret>
```

Sau đó dùng:

```text
Test Connection
```

để kiểm tra kết nối.

---

# 8. Attribute Mapping trên Mailcow

Ví dụ Keycloak gửi:

```json
{
  "email": "user@example.com",
  "mailcow_template": "default"
}
```

Trong Mailcow cấu hình:

```text
Default Template:
Default
```

và:

```text
Attribute Mapping

Attribute: default
Template:  Default
```

**Không nhập `mailcow_template` vào ô Attribute.**

Ô `Attribute` ở Mailcow là **giá trị của `mailcow_template`**, ví dụ:

```text
default
basic
business
vip
```

Ví dụ đầy đủ:

```text
Keycloak:
mailcow_template = business

Mailcow mapping:
business → Business
```

---

# 9. Auto-create mailbox khi login

Bật trên Mailcow:

```text
Auto-create users on login: ON
```

Luồng:

```text
User tồn tại trên Keycloak
        |
        | mailcow_template = default
        v
User login Mailcow bằng SSO
        |
        v
Mailcow đọc UserInfo
        |
        v
mailcow_template = default
        |
        v
Mapping:
default → Default
        |
        v
Mailcow tự tạo mailbox
```

## Điều kiện bắt buộc

Mail domain phải tồn tại trước trên Mailcow.

Ví dụ user:

```text
user@company.com
```

thì Mailcow phải có sẵn:

```text
company.com
```

Nếu domain chưa tồn tại, auto-create mailbox sẽ thất bại.

---

# 10. Keycloak không tự tạo Mail Domain

Keycloak chỉ quản lý Identity/User.

Không có cơ chế mặc định:

```text
Tạo user@newdomain.com trên Keycloak
→ Mailcow tự tạo newdomain.com
```

Muốn tự động hóa domain cần dùng một lớp riêng, ví dụ:

```text
Admin Portal
      |
      +------> Keycloak API
      |        - Create Group
      |        - Create User
      |
      +------> Mailcow API
               - Create Domain
               - Set Quota
               - Set Max Mailboxes
```

Sau đó Keycloak mới dùng cho provisioning user.

---

# 11. SSO Password và IMAP/SMTP Password là hai việc khác nhau

Keycloak OIDC xử lý tốt cho:

```text
Browser
Mailcow UI
SOGo
```

Nhưng:

```text
IMAP
SMTP
POP3
SIEVE
Outlook
Thunderbird
Apple Mail
```

không tự dùng OIDC password của Keycloak.

Có hai cách:

### Cách 1 — App Password của Mailcow

User login SSO rồi tạo App Password trong Mailcow.

Dùng cho:

```text
IMAP / SMTP / Outlook / Mobile mail client
```

Đây là cách đơn giản và ổn định.

### Cách 2 — Mailpassword Flow

Mailcow đọc password hash từ Keycloak attribute:

```text
mailcow_password
```

---

# 12. `mailcow_password` là attribute mặc định của Mailcow

Khi bật:

```text
Mailpassword Flow: ON
```

Mailcow mặc định tìm đúng attribute:

```text
mailcow_password
```

Không cần:

- OIDC Mapper cho `mailcow_password`.
- Attribute Mapping trong Mailcow cho `mailcow_password`.

Đây là tên attribute Mailcow đã quy ước sẵn.

Luồng:

```text
User login IMAP/SMTP
        |
        v
Mailcow thấy Mailpassword Flow = ON
        |
        v
Mailcow dùng Service Account
        |
        v
Keycloak Admin REST API
        |
        v
Tìm user
        |
        v
Đọc:
mailcow_password
        |
        v
Kiểm tra password
```

---

# 13. Bật quyền cho Service Account

Trong Keycloak:

```text
Clients
→ mailcow
→ Settings
```

bật:

```text
Service account roles: ON
```

Sau đó:

```text
Clients
→ mailcow
→ Service account roles
→ Assign role
→ Filter by clients
→ realm-management
→ view-users
→ Assign
```

Mailcow cần quyền này để đọc thông tin user qua Keycloak Admin REST API.

---

# 14. Tạo `mailcow_password` trong User Profile

Vào:

```text
Realm settings
→ User profile
→ Create attribute
```

Cấu hình:

```text
Attribute Name:
mailcow_password

Display name:
Mailcow Password

Multivalued:
OFF

Default value:
để trống

Enabled when:
Always

Required field:
OFF
```

Permission nên để:

```text
Admin:
View + Edit

User:
không cho Edit
```

---

# 15. Giá trị `mailcow_password` phải là hash

Không nhập password plaintext.

Ví dụ:

```text
mailcow_password =
{BLF-CRYPT}$2y$...
```

Có thể tạo bcrypt trên Linux bằng:

```bash
apt update
apt install whois -y
```

Sau đó:

```bash
mkpasswd -m bcrypt | sed 's/^/{BLF-CRYPT}/'
```

Lệnh sẽ hỏi password và trả về chuỗi dạng:

```text
{BLF-CRYPT}$2y$05$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Copy toàn bộ chuỗi vào:

```text
mailcow_password
```

---

# 16. Password Keycloak không tự đồng bộ sang `mailcow_password`

Đây là điểm rất quan trọng.

Keycloak có:

```text
Credential Password
```

và riêng biệt:

```text
User Attribute:
mailcow_password
```

Hai giá trị này không tự đồng bộ.

Ví dụ user đổi:

```text
Keycloak password:
OldPass123
→ NewPass456
```

thì:

```text
mailcow_password
```

không tự đổi theo.

Muốn "một password dùng cho cả SSO và IMAP/SMTP" thì phải xây thêm cơ chế đồng bộ/update `mailcow_password`.

---

# 17. User test

Tạo user trong realm:

```text
Users
→ Create new user
```

Ví dụ:

```text
Username:
test@example.com

Email:
test@example.com

Enabled:
ON
```

Set password:

```text
Users
→ test@example.com
→ Credentials
→ Set password

Temporary:
OFF
```

User cần có:

```text
mailcow_template = default
```

Nếu dùng Mailpassword Flow thì thêm:

```text
mailcow_password = {BLF-CRYPT}$2y$...
```

---

# 18. Checklist cấu hình

## Keycloak Client

```text
[✓] Client ID = mailcow
[✓] Client authentication = ON
[✓] Standard flow = ON
[✓] Service account roles = ON (nếu dùng Mailpassword Flow/API)
[✓] Valid redirect URI
[✓] Web origins
[✓] Client Secret
```

## Keycloak User Profile

```text
[✓] mailcow_template
[✓] mailcow_password (nếu dùng Mailpassword Flow)
```

## Mapper

```text
[✓] User Attribute = mailcow_template
[✓] Token Claim Name = mailcow_template
[✓] Add to userinfo = ON
```

## Service Account

```text
[✓] realm-management
    └── view-users
```

## Mailcow

```text
[✓] Keycloak Identity Provider
[✓] Server URL
[✓] Realm
[✓] Client ID
[✓] Client Secret
[✓] Attribute mapping
[✓] Auto-create users on login = ON
[✓] Mailpassword Flow = ON (nếu cần)
```

## Domain

```text
[✓] Domain phải tồn tại trong Mailcow trước
```

---

# 19. Luồng hoàn chỉnh

```text
               KEYCLOAK
                   |
                   | user@example.com
                   |
                   | mailcow_template=default
                   | mailcow_password={BLF-CRYPT}...
                   |
          +--------+--------+
          |                 |
          | OIDC            | Admin REST API
          v                 v
      Mailcow UI        Mailpassword Flow
          |                 |
          v                 v
      SSO / SOGo       IMAP / SMTP
          |
          v
   Auto-create mailbox
          |
          v
 default → Default Template
```

Điều kiện:

```text
example.com
```

phải tồn tại trong Mailcow trước.

---

# 20. Ghi nhớ nhanh

```text
mailcow_template
→ chọn Mailbox Template
→ cần OIDC Mapper
→ cần Attribute Mapping

mailcow_password
→ xác thực IMAP/SMTP
→ không cần OIDC Mapper
→ không cần Attribute Mapping
→ Mailpassword Flow tự đọc

Mail Domain
→ phải tạo bên Mailcow
→ Keycloak không tự tạo

Auto-create users on login
→ chỉ tạo mailbox
→ không tạo domain
```

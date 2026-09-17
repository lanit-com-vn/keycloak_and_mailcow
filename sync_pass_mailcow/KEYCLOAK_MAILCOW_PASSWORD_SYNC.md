# KEYCLOAK_MAILCOW_PASSWORD_SYNC.md

# Hướng dẫn đồng bộ Password Keycloak → Mailcow

Tài liệu này mô tả cách triển khai plugin `mailcow-password-sync` để khi user đổi mật khẩu tại Keycloak Account Console thì Keycloak tự cập nhật attribute:

```text
mailcow_password
```

Mailcow sẽ đọc attribute này qua **Mailpassword Flow** để dùng cùng mật khẩu cho:

- IMAP
- SMTP
- POP3
- SIEVE
- Outlook
- Thunderbird
- Apple Mail

---

## 1. Mục tiêu

User chỉ cần đổi mật khẩu tại:

```text
https://login.techfarm.com.vn/realms/techfarm/account/
```

Luồng:

```text
User đổi password
        |
        v
Keycloak UPDATE_PASSWORD
        |
        +----------------------+
        |                      |
        v                      v
Keycloak Credential      Plugin tạo bcrypt
                               |
                               v
                       mailcow_password
                               |
                               v
                     Mailcow Mailpassword Flow
                               |
                               v
                     IMAP / SMTP / POP3
```

User chỉ cần nhớ **một password**.

---

# 2. Điều kiện trước khi cài

Keycloak:

```text
/opt/keycloak
```

Kiểm tra version:

```bash
/opt/keycloak/bin/kc.sh --version
```

Ví dụ:

```text
Keycloak 26.7.4
```

Plugin nên được build đúng version Keycloak đang chạy.

---

# 3. Cấu trúc project

```text
mailcow-password-sync/
├── pom.xml
├── build.sh
├── README.md
└── src/
    └── main/
        ├── java/
        │   └── vn/
        │       └── techfarm/
        │           └── keycloak/
        │               └── mailcow/
        │                   └── MailcowSyncUpdatePassword.java
        └── resources/
            └── META-INF/
                └── services/
                    └── org.keycloak.authentication.RequiredActionFactory
```

---

# 4. Build plugin

Vào thư mục:

```bash
cd /opt/mailcow-password-sync
```

Cấp quyền:

```bash
chmod +x build.sh
```

Chạy:

```bash
./build.sh
```

`build.sh` sẽ tự:

```text
- kiểm tra Ubuntu/Debian
- cài Java 21 nếu thiếu
- cài Maven nếu thiếu
- phát hiện version Keycloak
- build plugin đúng version Keycloak
```

Nếu thành công sẽ có:

```text
target/mailcow-password-sync-1.0.0.jar
```

Kiểm tra:

```bash
ls -lh target/mailcow-password-sync-1.0.0.jar
```

---

# 5. Cài plugin vào Keycloak

Copy JAR:

```bash
cp target/mailcow-password-sync-1.0.0.jar \
  /opt/keycloak/providers/
```

Build lại Keycloak:

```bash
cd /opt/keycloak
bin/kc.sh build
```

Restart:

```bash
systemctl restart keycloak
```

Kiểm tra:

```bash
systemctl status keycloak --no-pager
```

Theo dõi log:

```bash
journalctl -u keycloak -f
```

---

# 6. Realm mặc định

Plugin mặc định chỉ hoạt động trên realm:

```text
techfarm
```

Không tác động tới realm:

```text
master
```

hoặc realm khác.

Có thể đổi mà **không cần build lại plugin**.

Chạy:

```bash
systemctl edit keycloak
```

Thêm:

```ini
[Service]
Environment="MAILCOW_SYNC_REALM=techfarm"
```

Sau đó:

```bash
systemctl daemon-reload
systemctl restart keycloak
```

Ví dụ đổi sang realm `lanit`:

```ini
[Service]
Environment="MAILCOW_SYNC_REALM=lanit"
```

Không cần chạy lại:

```text
./build.sh
```

---

# 7. Điều kiện `mailcow_template`

Mặc định plugin có:

```text
MAILCOW_SYNC_REQUIRE_TEMPLATE=true
```

Có nghĩa là plugin chỉ sync password cho user có attribute:

```text
mailcow_template
```

Ví dụ:

```text
mailcow_template = default
```

hoặc:

```text
mailcow_template = basic
```

hoặc:

```text
mailcow_template = business
```

hoặc:

```text
mailcow_template = enterprise
```

Plugin **không quan tâm giá trị cụ thể**.

Nó chỉ kiểm tra:

```text
User có mailcow_template hay không?
```

Luồng:

```text
User đổi password
       |
       v
Realm đúng?
       |
       v
Có mailcow_template?
       |
       +---- NO ----> bỏ qua
       |
       +---- YES ---> cập nhật mailcow_password
```

Như vậy `mailcow_template` đồng thời đóng vai trò đánh dấu:

```text
User này có sử dụng Mailcow
```

---

# 8. Nếu muốn sync mọi user trong realm

Nếu muốn mọi user trong realm đều được sync, kể cả không có `mailcow_template`:

```bash
systemctl edit keycloak
```

Thêm:

```ini
[Service]
Environment="MAILCOW_SYNC_REQUIRE_TEMPLATE=false"
```

Sau đó:

```bash
systemctl daemon-reload
systemctl restart keycloak
```

Không cần build lại JAR.

---

# 9. Bcrypt cost

Plugin mặc định:

```text
MAILCOW_BCRYPT_COST=12
```

Có thể chỉnh:

```bash
systemctl edit keycloak
```

Ví dụ:

```ini
[Service]
Environment="MAILCOW_BCRYPT_COST=12"
```

Sau đó:

```bash
systemctl daemon-reload
systemctl restart keycloak
```

---

# 10. Attribute cần có trong Keycloak

Trong:

```text
Realm settings
→ User profile
```

nên có:

```text
mailcow_template
mailcow_password
```

## mailcow_template

Ví dụ:

```text
mailcow_template = default
```

Dùng để Mailcow chọn Mailbox Template.

## mailcow_password

Ví dụ:

```text
{BLF-CRYPT}$2a$12$...
```

Đây là bcrypt hash.

Không lưu plaintext password.

---

# 11. Permission cho `mailcow_password`

Khuyến nghị:

```text
Admin:
View = ON
Edit = ON

User:
View = OFF
Edit = OFF
```

User không cần nhìn thấy password hash.

Không tạo OIDC Mapper cho:

```text
mailcow_password
```

---

# 12. Mailcow cần bật gì?

Trong Mailcow:

```text
System
→ Configuration
→ Access
→ Identity Provider
→ Keycloak
```

Bật:

```text
Mailpassword Flow = ON
```

Client `mailcow` bên Keycloak cần:

```text
Service account roles = ON
```

và Service Account cần role:

```text
realm-management
└── view-users
```

Mailcow dùng quyền này để đọc:

```text
mailcow_password
```

qua Keycloak Admin REST API.

---

# 13. `mailcow_password` không cần Mapper

Không cần tạo Mapper OIDC cho:

```text
mailcow_password
```

Không cần Attribute Mapping bên Mailcow cho:

```text
mailcow_password
```

Mailcow Mailpassword Flow mặc định đã biết tên attribute:

```text
mailcow_password
```

Khác với:

```text
mailcow_template
```

`mailcow_template` cần Mapper OIDC.

---

# 14. Plugin có cập nhật nhầm user không?

Không.

Plugin không tìm user bằng:

```text
username
email
search query
```

Nó lấy trực tiếp user đang thực hiện `UPDATE_PASSWORD`:

```java
UserModel user = context.getUser();
```

Sau đó ghi:

```java
user.setSingleAttribute(
    "mailcow_password",
    mailcowHash
);
```

Luồng:

```text
user1 login Keycloak
        |
        v
user1 đổi password
        |
        v
context.getUser()
        |
        v
UserModel của user1
        |
        v
mailcow_password của user1
```

Không có đoạn:

```text
search user
→ lấy user khác
→ update
```

Do đó user A đổi mật khẩu sẽ không tự cập nhật `mailcow_password` của user B.

---

# 15. Realm cũng là một lớp bảo vệ

Plugin kiểm tra:

```text
realm hiện tại
```

phải đúng với:

```text
MAILCOW_SYNC_REALM
```

Ví dụ:

```text
MAILCOW_SYNC_REALM=techfarm
```

Nếu user ở:

```text
realm=test
```

thì plugin bỏ qua.

Do đó ngay cả khi:

```text
techfarm/user@example.com
```

và:

```text
test/user@example.com
```

có cùng username thì plugin vẫn không nhầm realm.

---

# 16. Test không nhầm user

Tạo hai user:

```text
user1@techfarm.com.vn
user2@techfarm.com.vn
```

Cả hai có:

```text
mailcow_template = default
```

Ghi lại `mailcow_password` hiện tại của cả hai.

Sau đó chỉ login:

```text
user1@techfarm.com.vn
```

vào:

```text
https://login.techfarm.com.vn/realms/techfarm/account/
```

và đổi password.

Kết quả mong đợi:

```text
user1:
mailcow_password thay đổi

user2:
mailcow_password giữ nguyên
```

---

# 17. Kiểm tra log

Sau khi đổi password:

```bash
journalctl -u keycloak -n 100 --no-pager | grep -i mailcow
```

Nếu thành công:

```text
Mailcow password hash synchronized for user 'user1@techfarm.com.vn' in realm 'techfarm'
```

Plugin không log:

```text
plaintext password
password hash
```

---

# 18. Test thực tế IMAP/SMTP

Sau khi user đổi password trong Keycloak Account Console:

```text
Old Password:
OldPass123

New Password:
NewPass456
```

Keycloak:

```text
Credential Password = NewPass456
```

Attribute:

```text
mailcow_password = bcrypt(NewPass456)
```

Sau đó Outlook / IMAP / SMTP dùng:

```text
Username:
user@domain.com

Password:
NewPass456
```

Password cũ:

```text
OldPass123
```

phải không còn dùng được.

---

# 19. Plugin chỉ bắt luồng `UPDATE_PASSWORD`

Plugin hoạt động khi user đổi password qua:

```text
Keycloak Account Console
```

hoặc luồng Required Action:

```text
UPDATE_PASSWORD
```

Ví dụ:

```text
https://login.techfarm.com.vn/realms/techfarm/account/
```

→ Change Password.

---

# 20. Admin reset password

Nếu Admin trực tiếp set credential bằng một luồng khác thì không đảm bảo plugin được gọi.

Cách nên dùng khi Admin cần reset:

```text
Admin tạo Temporary Password
        |
        v
User login
        |
        v
Keycloak yêu cầu UPDATE_PASSWORD
        |
        v
User nhập password mới
        |
        v
Plugin sync mailcow_password
```

Như vậy password cuối cùng do user tự đặt vẫn được đồng bộ.

---

# 21. Logic plugin

Logic cơ bản:

```text
UPDATE_PASSWORD
      |
      v
Keycloak xử lý password policy
      |
      v
Password update thành công?
      |
      +---- NO ----> không sync
      |
      +---- YES
              |
              v
      Realm có đúng?
              |
              +---- NO ----> bỏ qua
              |
              +---- YES
                     |
                     v
          Có mailcow_template?
                     |
                     +---- NO ----> bỏ qua
                     |
                     +---- YES
                            |
                            v
                   bcrypt(password mới)
                            |
                            v
                   mailcow_password
```

---

# 22. Cấu hình systemd khuyến nghị

Chạy:

```bash
systemctl edit keycloak
```

Thêm:

```ini
[Service]
Environment="MAILCOW_SYNC_REALM=techfarm"
Environment="MAILCOW_BCRYPT_COST=12"
Environment="MAILCOW_SYNC_REQUIRE_TEMPLATE=true"
```

Apply:

```bash
systemctl daemon-reload
systemctl restart keycloak
```

Kiểm tra:

```bash
systemctl show keycloak --property=Environment
```

---

# 23. Update plugin

Sau khi chỉnh code:

```bash
cd /opt/mailcow-password-sync
./build.sh
```

Copy JAR mới:

```bash
cp target/mailcow-password-sync-1.0.0.jar \
  /opt/keycloak/providers/
```

Sau đó:

```bash
cd /opt/keycloak
bin/kc.sh build
systemctl restart keycloak
```

---

# 24. Rollback plugin

Nếu plugin có lỗi:

```bash
rm -f /opt/keycloak/providers/mailcow-password-sync-1.0.0.jar
```

Build lại Keycloak:

```bash
cd /opt/keycloak
bin/kc.sh build
```

Restart:

```bash
systemctl restart keycloak
```

Keycloak sẽ quay lại `UPDATE_PASSWORD` built-in.

---

# 25. Checklist Production

## Keycloak

```text
[ ] Plugin JAR đã nằm trong /opt/keycloak/providers/
[ ] kc.sh build thành công
[ ] Keycloak restart thành công
[ ] MAILCOW_SYNC_REALM đúng
[ ] mailcow_template tồn tại
[ ] mailcow_password tồn tại
[ ] User không được xem mailcow_password
```

## Mailcow

```text
[ ] Keycloak Identity Provider hoạt động
[ ] Mailpassword Flow = ON
[ ] Service account roles = ON
[ ] realm-management/view-users đã cấp
```

## Test

```text
[ ] User đổi password Keycloak thành công
[ ] mailcow_password đổi
[ ] User khác không bị đổi
[ ] IMAP login bằng password mới thành công
[ ] SMTP login bằng password mới thành công
[ ] Password cũ không còn dùng được
```

---

# 26. Kiến trúc cuối

```text
                           KEYCLOAK
                              |
               +--------------+--------------+
               |                             |
               v                             v
         SSO Password                 mailcow_password
               |                        bcrypt hash
               |                             |
               v                             v
        Mailcow / SOGo              Mailpassword Flow
                                             |
                                   +---------+---------+
                                   |         |         |
                                   v         v         v
                                  IMAP      SMTP      POP3
```

User:

```text
đổi password một lần tại Keycloak
```

Hệ thống:

```text
Keycloak SSO Password
        =
Password dùng cho IMAP/SMTP
```

nhưng mỗi hệ thống vẫn lưu **hash riêng**, không copy hash Keycloak sang Mailcow.

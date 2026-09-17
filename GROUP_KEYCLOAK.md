# GROUP_KEYCLOAK.md

# Hướng dẫn tạo Group và Domain Admin theo từng domain trên Keycloak

Mục tiêu là mỗi khách hàng có một domain riêng, ví dụ:

```text
techfarm.com.vn
abc.com
xyz.vn
```

và mỗi domain có một tài khoản quản trị riêng:

```text
admin@techfarm.com.vn
admin@abc.com
admin@xyz.vn
```

Domain Admin chỉ được quản lý user thuộc domain của mình, không được quản lý user của domain khác.

---

## 1. Cấu trúc Group

Trong realm:

```text
techfarm
```

vào:

```text
Groups
```

Tạo group cha:

```text
Domain
```

Sau đó mỗi domain là một group con:

```text
Domain
├── techfarm.com.vn
├── abc.com
└── xyz.vn
```

Ví dụ với:

```text
techfarm.com.vn
```

tạo tiếp hai group con:

```text
Domain
└── techfarm.com.vn
    ├── Admins
    └── Users
```

Ý nghĩa:

```text
Admins
→ chứa Domain Admin

Users
→ chứa các mailbox/user thông thường
```

---

## 2. Thêm metadata cho domain

Vào:

```text
Groups
→ Domain
→ techfarm.com.vn
→ Attributes
```

Thêm 4 attribute:

```text
domain = techfarm.com.vn
package = email_hosting_4
max_users = 20
domain_quota_mb = 25600
```

Kết quả:

```text
Domain
└── techfarm.com.vn
    │
    ├── domain = techfarm.com.vn
    ├── package = email_hosting_4
    ├── max_users = 20
    ├── domain_quota_mb = 25600
    │
    ├── Admins
    └── Users
```

Trong đó:

```text
package
→ mã gói khách đang sử dụng

max_users
→ số mailbox tối đa

domain_quota_mb
→ tổng dung lượng domain theo MB
```

Hiện tại các giá trị này là metadata. Sau này plugin Keycloak có thể đọc chúng để enforce quota.

---

## 3. Tạo Domain Admin

Vào:

```text
Users
→ Create new user
```

Tạo theo chuẩn:

```text
Username:
admin@techfarm.com.vn

Email:
admin@techfarm.com.vn

Enabled:
ON
```

Sau đó vào:

```text
Credentials
→ Set password
```

đặt:

```text
Temporary = OFF
```

---

## 4. Đưa Domain Admin vào group Admins

Vào:

```text
Users
→ admin@techfarm.com.vn
→ Groups
→ Join Group
```

chọn:

```text
/Domain/techfarm.com.vn/Admins
```

Kết quả:

```text
Domain
└── techfarm.com.vn
    ├── Admins
    │   └── admin@techfarm.com.vn
    │
    └── Users
```

---

## 5. Bật Fine-Grained Admin Permissions

Vào:

```text
Realm settings
→ Admin Permissions
```

bật:

```text
Admin Permissions = ON
```

Sau đó Keycloak sẽ có phần:

```text
Permissions
```

---

## 6. Tạo policy nhận diện Domain Admin

Vào:

```text
Permissions
→ Policies
→ Create new policy
```

Cấu hình:

```text
Name:
techfarm-domain-admins

Policy type:
Group
```

Group:

```text
/Domain/techfarm.com.vn/Admins
```

```text
Groups claim:
để trống

Logic:
Positive
```

Ý nghĩa:

```text
Ai thuộc:
/Domain/techfarm.com.vn/Admins

→ được xem là Domain Admin của techfarm.com.vn
```

---

## 7. Permission quản lý Users

Tạo permission:

```text
Name:
techfarm-users-management
```

Enforce access to:

```text
Specific Groups
```

Group:

```text
/Domain/techfarm.com.vn/Users
```

Authorization scopes:

```text
view
view-members
manage-members
manage-membership
```

Policy:

```text
techfarm-domain-admins
```

Kết quả:

```text
Admins
        ↓
techfarm-domain-admins
        ↓
techfarm-users-management
        ↓
/Domain/techfarm.com.vn/Users
```

Domain Admin được quyền xem và quản lý user trong group `Users`.

---

## 8. Permission để duyệt cây Group

Do `Users` nằm sâu:

```text
Domain
└── techfarm.com.vn
    └── Users
```

nên Domain Admin phải có quyền `view` các group cha thì giao diện Keycloak mới duyệt xuống được.

Tạo permission:

```text
Name:
techfarm-group-navigation
```

Enforce access:

```text
Specific Groups
```

thêm:

```text
/Domain
/Domain/techfarm.com.vn
```

Authorization scope chỉ chọn:

```text
view
```

Policy:

```text
techfarm-domain-admins
```

Không cấp:

```text
manage-members
manage-membership
```

cho group cha.

Kết quả:

```text
admin@techfarm.com.vn

VIEW:
 /Domain
 /Domain/techfarm.com.vn

MANAGE:
 /Domain/techfarm.com.vn/Users
```

---

## 9. Cấp role tối thiểu để vào Admin Console

Vào:

```text
Users
→ admin@techfarm.com.vn
→ Role mapping
→ Assign role
```

lọc:

```text
Filter by clients
```

Client:

```text
realm-management
```

chỉ cấp:

```text
query-users
query-groups
```

Không cấp:

```text
manage-users
view-users
realm-admin
manage-realm
```

Kết quả Role Mapping:

```text
realm-management
├── query-users
└── query-groups
```

---

## 10. Domain Admin đăng nhập

Domain Admin truy cập:

```text
https://login.techfarm.com.vn/admin/techfarm/console/
```

đăng nhập:

```text
admin@techfarm.com.vn
```

Menu chính chỉ cần:

```text
Users
Groups
```

---

## 11. Domain Admin tạo user

Domain Admin vào:

```text
Users
→ Create new user
```

Ví dụ:

```text
Username:
dungnt@techfarm.com.vn

Email:
dungnt@techfarm.com.vn

Enabled:
ON
```

Ở phần:

```text
Groups
→ Join Groups
```

chọn:

```text
/Domain/techfarm.com.vn/Users
```

sau đó mới bấm:

```text
Create
```

User mới sẽ nằm:

```text
Domain
└── techfarm.com.vn
    └── Users
        └── dungnt@techfarm.com.vn
```

---

## 12. Tại sao phải chọn Group khi tạo user?

Domain Admin không có:

```text
manage-users
```

toàn realm.

Vì vậy khi tạo user, Keycloak cần biết user đó thuộc resource mà Domain Admin được phép quản lý:

```text
/Domain/techfarm.com.vn/Users
```

Nếu không chọn group, có thể nhận:

```text
HTTP 403 Forbidden
```

Đây là đúng logic bảo mật.

---

## 13. Cấu trúc cuối cùng

```text
Realm: techfarm

Domain
│
├── techfarm.com.vn
│   │
│   ├── Attributes
│   │   ├── domain = techfarm.com.vn
│   │   ├── package = email_hosting_4
│   │   ├── max_users = 20
│   │   └── domain_quota_mb = 25600
│   │
│   ├── Admins
│   │   └── admin@techfarm.com.vn
│   │
│   └── Users
│       ├── dungnt@techfarm.com.vn
│       ├── sale@techfarm.com.vn
│       └── info@techfarm.com.vn
│
├── abc.com
│   ├── Admins
│   │   └── admin@abc.com
│   └── Users
│
└── xyz.vn
    ├── Admins
    │   └── admin@xyz.vn
    └── Users
```

Mỗi domain sẽ có bộ permission riêng:

```text
techfarm-domain-admins
techfarm-users-management
techfarm-group-navigation
```

Ví dụ `abc.com`:

```text
abc-domain-admins
abc-users-management
abc-group-navigation
```

---

## 14. Phần đã hoàn thành

```text
✓ Một Domain Admin riêng cho từng domain

✓ Domain Admin đăng nhập trực tiếp Keycloak Admin Console

✓ Không cần viết trang Admin riêng

✓ Domain Admin chỉ quản lý Users của domain mình

✓ Có thể tạo/reset/quản lý user thuộc group được cấp

✓ Không cần cấp manage-users toàn realm

✓ Có metadata package/quota/max_users cho từng domain
```

---

## 15. Phần sẽ làm tiếp

Plugin/extension để enforce:

```text
max_users = 20
```

và tự chặn:

```text
admin@techfarm.com.vn
→ tạo user thứ 21
→ DENY
```

Đồng thời ép user mới phải đúng domain:

```text
*@techfarm.com.vn
```

không được tạo:

```text
user@gmail.com
user@abc.com
```

Ngoài ra có thể dùng metadata:

```text
package
domain_quota_mb
```

để đồng bộ chính sách xuống Mailcow.

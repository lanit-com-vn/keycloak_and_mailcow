# Hướng dẫn cấu hình Mailcow Template dạng dropdown trên Keycloak

Mục tiêu: thay vì để Domain Admin tự gõ giá trị `mailcow_template`, Keycloak sẽ hiển thị danh sách chọn sẵn:

```text
1GB
2GB
5GB
10GB
20GB
50GB
```

Giá trị được chọn sẽ được lưu vào:

```text
mailcow_template
```

và gửi sang Mailcow để map sang Mailbox Template tương ứng.

---

## 1. Ý nghĩa của `mailcow_template`

Ví dụ Domain Admin tạo:

```text
sale@techfarm.com.vn
```

và chọn:

```text
Mailcow Template:
5GB
```

Keycloak lưu:

```text
mailcow_template = 5GB
```

Mailcow nhận giá trị:

```text
5GB
```

và map:

```text
5GB → Template 5GB
```

Ví dụ:

```text
Keycloak              Mailcow
---------------------------------
1GB              →    Template 1GB
2GB              →    Template 2GB
5GB              →    Template 5GB
10GB             →    Template 10GB
20GB             →    Template 20GB
50GB             →    Template 50GB
```

> Đây là template/quota của từng mailbox, không phải quota tổng của domain.

---

## 2. Mở User Profile

Vào:

```text
Realm settings
→ User profile
→ mailcow_template
```

---

## 3. Permission

Giữ:

```text
Who can edit?
User   OFF
Admin  ON

Who can view?
User   OFF
Admin  ON
```

User thường không tự đổi quota/template của mình.

---

## 4. Thêm validator `options`

Ở:

```text
Validations
→ Add validator
```

chọn:

```text
options
```

thêm các giá trị:

```text
1GB
2GB
5GB
10GB
20GB
50GB
```

Kết quả:

```text
Validations
└── options
    ├── 1GB
    ├── 2GB
    ├── 5GB
    ├── 10GB
    ├── 20GB
    └── 50GB
```

Validator `options` xác định danh sách giá trị hợp lệ.

---

## 5. Đổi input thành dropdown

Ở:

```text
Annotations
→ Add Annotations
```

thêm:

```text
Key:
inputType

Value:
select
```

Sau khi Save, `Mailcow Template` sẽ thành dropdown:

```text
Mailcow Template:
[ 5GB ▼ ]
```

---

## 6. Default value

Nếu muốn Domain Admin tự chọn cho từng mailbox:

```text
Default value:
để trống
```

Nếu muốn mặc định, ví dụ:

```text
2GB
```

thì đặt:

```text
Default value:
2GB
```

---

## 7. Tạo Mailbox Template trên Mailcow

Tạo các template tương ứng:

```text
Template 1GB  → 1024 MB
Template 2GB  → 2048 MB
Template 5GB  → 5120 MB
Template 10GB → 10240 MB
Template 20GB → 20480 MB
Template 50GB → 51200 MB
```

---

## 8. Attribute Mapping bên Mailcow

Vào:

```text
System
→ Configuration
→ Access
→ Identity Provider
→ Keycloak
```

ở phần:

```text
Attribute Mapping
```

thêm:

```text
Attribute    Template
---------------------
1GB          1GB
2GB          2GB
5GB          5GB
10GB         10GB
20GB         20GB
50GB         50GB
```

Ví dụ:

```text
mailcow_template = 5GB
```

thì Mailcow map:

```text
5GB → Template 5GB
```

---

## 9. Luồng Domain Admin tạo user

```text
admin@techfarm.com.vn
        ↓
Create User
        ↓
sale@techfarm.com.vn
        ↓
Join Group:
/Domain/techfarm.com.vn/Users
        ↓
Mailcow Template:
5GB
        ↓
mailcow_template = 5GB
        ↓
Mailcow
        ↓
5GB → Template 5GB
```

---

## 10. Phân biệt quota mailbox và quota domain

Ví dụ domain:

```text
techfarm.com.vn
```

có:

```text
max_users = 20
domain_quota_mb = 25600
```

thì:

```text
Max mailbox:
20

Total domain quota:
25GB
```

Trong khi từng mailbox có thể là:

```text
ceo@techfarm.com.vn      = 10GB
sale@techfarm.com.vn     = 5GB
hr@techfarm.com.vn       = 2GB
support@techfarm.com.vn  = 5GB
info@techfarm.com.vn     = 3GB
                         -----
                         25GB
```

Mailcow vẫn là nơi enforce quota tổng.

---

## 11. Cấu hình cuối cùng

Keycloak:

```text
mailcow_template
├── Validations
│   └── options
│       ├── 1GB
│       ├── 2GB
│       ├── 5GB
│       ├── 10GB
│       ├── 20GB
│       └── 50GB
│
└── Annotations
    └── inputType = select
```

Mailcow:

```text
1GB  → 1GB
2GB  → 2GB
5GB  → 5GB
10GB → 10GB
20GB → 20GB
50GB → 50GB
```

---

## 12. Ghi nhớ nhanh

```text
mailcow_template
→ quota/template từng mailbox

domain_quota_mb
→ tổng quota domain

max_users
→ số mailbox tối đa
```

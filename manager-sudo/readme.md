# Lab Toàn Diện: Quản Lý User, Group, Sudo và Phân Quyền Trên Linux

Tài liệu này là hướng dẫn thực hành chuyên sâu từ cơ bản đến nâng cao về quản trị người dùng (**User**), nhóm người dùng (**Group**), đặc quyền (**Sudo**) và phân quyền thư mục dự án (**Permissions & Setgid**) trên Linux.

---

## Mục lục
1. [Bản chất kiến trúc User & Group trong Linux](#1-bản-chất-kiến-trúc-user--group-trong-linux)
2. [Phần 1: Xem và kiểm tra thông tin User](#2-phần-1-xem-và-kiểm-tra-thông-tin-user)
3. [Phần 2: Quản trị User toàn diện (Tạo, Sửa, Khóa, Xóa)](#3-phần-2-quản-trị-user-toàn-diện)
4. [Phần 3: Quản trị Group toàn diện (Tạo, Thêm/Bớt thành viên, Xóa)](#4-phần-3-quản-trị-group-toàn-diện)
5. [Phần 4: Quản trị quyền Sudo chuyên sâu](#5-phần-4-quản-trị-quyền-sudo-chuyên-sâu)
6. [Phần 5: Kịch bản Lab thực hành tích hợp (/opt/project)](#6-phần-5-kịch-bản-lab-thực-hành-tích-hợp)
7. [Phần 6: Khắc phục sự cố thường gặp (Troubleshooting)](#7-phần-6-khắc-phục-sự-cố-thường-gặp)
8. [Phần 7: Bảng tra cứu lệnh nhanh (Cheat Sheet)](#8-phần-7-bảng-tra-cứu-lệnh-nhanh-cheat-sheet)
9. [Phần 8: Dọn dẹp môi trường Lab](#9-phần-8-dọn-dẹp-môi-trường-lab)

---

## 1. Bản chất kiến trúc User & Group trong Linux

Hệ điều hành Linux quản lý quyền dựa trên các số định danh:
- **UID (User ID)**: Mã số định danh của người dùng.
  - `UID = 0`: Siêu người dùng `root` (toàn quyền trên hệ thống).
  - `UID 1 - 999` (hoặc `1 - 499` tùy distro): System User (dành cho các dịch vụ daemon như `nginx`, `mysql`, `systemd`).
  - `UID >= 1000`: Regular User (người dùng thật do quản trị viên tạo).
- **GID (Group ID)**: Mã số định danh của nhóm.

### Các file cơ sở dữ liệu hệ thống cốt lõi

| Đường dẫn | Mục đích | Quyền mặc định | Ai đọc được? |
| :--- | :--- | :--- | :--- |
| `/etc/passwd` | Chứa danh sách user và thông tin cơ bản | `rw-r--r--` (644) | Mọi người dùng |
| `/etc/shadow` | Chứa mật khẩu đã mã hóa (hash) và hạn dùng pass | `r--------` hoặc `rw-------` | Chỉ `root` |
| `/etc/group` | Chứa danh sách các group và thành viên | `rw-r--r--` (644) | Mọi người dùng |
| `/etc/gshadow` | Chứa mật khẩu và admin của group | `r--------` | Chỉ `root` |
| `/etc/skel/` | Thư mục mẫu chứa file cấu hình (`.bashrc`, `.profile`...) tự động sao chép sang home của user mới tạo | `rwxr-xr-x` | `root` |

### Giải mã 7 trường trong `/etc/passwd`

Ví dụ một dòng trong `/etc/passwd`:
```text
alice:x:1001:1001:Alice DevOps:/home/alice:/bin/bash
  1   2   3    4         5            6          7
```
1. `alice`: Username (tên đăng nhập).
2. `x`: Mật khẩu được lưu trong `/etc/shadow` (không lưu trực tiếp tại đây để bảo mật).
3. `1001`: User ID (UID).
4. `1001`: Primary Group ID (GID chính).
5. `Alice DevOps`: GECOS / Comment (họ tên, ghi chú, phòng ban).
6. `/home/alice`: Thư mục Home directory của user.
7. `/bin/bash`: Login shell (chương trình shell chạy khi user đăng nhập).

---

## 2. Phần 1: Xem và kiểm tra thông tin User

Trước khi tạo hay xóa, quản trị viên cần biết cách kiểm tra hệ thống đang có những user nào và ai đang hoạt động.

### 2.1. Xem danh sách tất cả User trên máy

**Cách 1: Xem toàn bộ thông tin từ `/etc/passwd`**
```bash
cat /etc/passwd
# hoặc dùng getent (hỗ trợ cả tài nguyên mạng LDAP/Active Directory nếu có)
getent passwd
```

**Cách 2: Chỉ lấy danh sách tên người dùng (Username)**
```bash
cut -d: -f1 /etc/passwd
# hoặc
compgen -u
```

**Cách 3: Lọc danh sách người dùng thực tế (Regular Users - UID >= 1000)**
Bỏ qua các system daemon/service user để chỉ xem những user do người tạo:
```bash
awk -F: '$3 >= 1000 && $3 < 65534 {printf "User: %-15s | UID: %-5s | Shell: %-15s | Home: %s\n", $1, $3, $7, $6}' /etc/passwd
```

### 2.2. Xem người dùng đang đăng nhập và lịch sử hoạt động

```bash
# Xem bạn đang là ai
whoami

# Xem ai đang đăng nhập vào máy, từ IP/terminal nào
who

# Xem chi tiết: ai đang đăng nhập + máy đang chạy lệnh gì + uptime hệ thống
w

# Xem danh sách tên user đang login (ngắn gọn)
users

# Xem lịch sử đăng nhập gần nhất vào hệ thống
last -n 10

# Xem lần đăng nhập cuối cùng của tất cả user
lastlog
```

### 2.3. Xem thông tin chi tiết của một User cụ thể

```bash
# Xem UID, GID và các group của một user
id alice

# Xem một dòng thông tin đầy đủ của user trong cơ sở dữ liệu
getent passwd alice

# Xem chính sách hết hạn mật khẩu của user
sudo chage -l alice
```

---

## 3. Phần 2: Quản trị User toàn diện

### 3.1. Phân biệt `useradd` và `adduser`

- **`useradd`**: Lệnh chuẩn cấp thấp (low-level), có sẵn trên mọi bản phân phối Linux (Ubuntu, Debian, RHEL, CentOS, Alpine...). Khi dùng `useradd`, bạn nên truyền rõ các cờ (flags) để tạo thư mục home và chọn shell.
- **`adduser`**: Script cấp cao (high-level) tương tác bằng Perl (chủ yếu trên Debian/Ubuntu), sẽ tự hỏi mật khẩu, tạo home và sao chép cấu hình mẫu.

### 3.2. Tạo User bằng `useradd` (Chuẩn chuyên nghiệp cho Script & Mọi Distro)

Cú pháp đầy đủ:
```bash
sudo useradd [options] <username>
```

Các tùy chọn quan trọng:
- `-m` (`--create-home`): Bắt buộc tạo thư mục home `/home/<username>`.
- `-s <shell>`: Chỉ định login shell (ví dụ `/bin/bash`). Nếu không chỉ định, trên Ubuntu có thể mặc định là `/bin/sh` rất khó dùng.
- `-c "<comment>"`: Đặt tên mô tả, họ tên đầy đủ hoặc chức danh.
- `-g <group>`: Chỉ định primary group (nhóm chính).
- `-G <group1,group2>`: Thêm vào các supplementary groups (nhóm phụ).
- `-u <UID>`: Chỉ định UID thủ công.
- `-e YYYY-MM-DD`: Đặt ngày hết hạn tài khoản (thích hợp cho nhân viên thử việc / thực tập sinh).

**Ví dụ thực tế:**
```bash
# Tạo user alice có thư mục home, shell bash, kèm họ tên
sudo useradd -m -s /bin/bash -c "Alice Nguyen - Dev Lead" alice

# Tạo user bob
sudo useradd -m -s /bin/bash -c "Bob Tran - Backend Dev" bob

# Tạo user charlie có ngày hết hạn tài khoản đến 2026-12-31
sudo useradd -m -s /bin/bash -e 2026-12-31 -c "Charlie Le - QA Tester" charlie
```

**Tạo System User (Dành cho service/bot không cần đăng nhập shell):**
```bash
# User không có thư mục home và shell nologin
sudo useradd -r -s /usr/sbin/nologin deploybot
```

### 3.3. Đặt và quản lý mật khẩu (`passwd`, `chage`)

```bash
# Đặt mật khẩu cho user
sudo passwd alice
sudo passwd bob
sudo passwd charlie

# Bắt buộc user phải tự đổi mật khẩu ở lần đăng nhập đầu tiên
sudo passwd -e bob
# hoặc
sudo chage -d 0 bob
```

### 3.4. Chỉnh sửa thông tin User (`usermod`)

Nếu đã lỡ tạo user thiếu tham số, bạn dùng `usermod` để cập nhật:

```bash
# Đổi login shell sang /bin/bash
sudo usermod -s /bin/bash alice

# Đổi ghi chú/họ tên
sudo usermod -c "Alice Nguyen - Engineering Director" alice

# Đổi tên tài khoản (username) từ oldname sang newname
sudo usermod -l alice_new alice_old

# Đổi thư mục home và tự động DI CHUYỂN toàn bộ file sang home mới (-m)
sudo usermod -d /home/alice_data -m alice
```

### 3.5. Khóa và Mở khóa tài khoản (Lock / Unlock)

Trong thực tế doanh nghiệp, khi nhân viên nghỉ việc hoặc tạm dừng công tác, **không nên xóa tài khoản ngay** mà cần **khóa** để kiểm tra dữ liệu và bảo mật.

**Khóa tài khoản:**
```bash
# Khóa mật khẩu của user (thêm dấu ! vào hash mật khẩu trong /etc/shadow)
sudo usermod -L alice
# hoặc
sudo passwd -l alice

# Khóa luôn shell để không cho đăng nhập bằng SSH key
sudo usermod -s /usr/sbin/nologin alice
```

*Kiểm tra xem user đã bị khóa chưa:*
```bash
sudo grep alice /etc/shadow
# Kết quả có dấu ! ở đầu trường password: alice:!$6$...:19800:...
```

**Mở khóa tài khoản:**
```bash
# Mở khóa mật khẩu
sudo usermod -U alice
# hoặc
sudo passwd -u alice

# Khôi phục shell
sudo usermod -s /bin/bash alice
```

### 3.6. Xóa User an toàn và triệt để (`userdel`)

Trước khi xóa user, cần đảm bảo user đó không còn tiến trình (process) nào đang chạy ngầm trên hệ thống!

**Bước 1: Kiểm tra và dừng toàn bộ tiến trình của user**
```bash
# Xem các process của user bob
pgrep -u bob

# Dừng toàn bộ tiến trình của bob
sudo pkill -u bob
# hoặc tắt hẳn bằng tín hiệu SIGKILL
sudo killall -9 -u bob
```

**Bước 2: Tiến hành xóa user**
```bash
# LỰA CHỌN A: Chỉ xóa user trong hệ thống, GIỮ LẠI thư mục /home/bob và mail spool
sudo userdel bob

# LỰA CHỌN B: Xóa TRIỆT ĐỂ cả user, thư mục /home/bob và toàn bộ file hòm thư
# Trên Ubuntu/Debian hoặc CentOS/RHEL:
sudo userdel -r bob
# Hoặc trên Debian/Ubuntu có thể dùng:
sudo deluser --remove-home bob
```

**Bước 3: Xử lý các file "mồ côi" (Orphaned Files)**
Nếu user đã bị xóa nhưng trước đó có tạo file ở `/tmp`, `/opt`, `/var`..., các file đó sẽ không còn ai sở hữu (hiển thị UID dạng số). Cách quét tìm:
```bash
# Tìm các file trên hệ thống không còn user sở hữu
sudo find / -nouser 2>/dev/null

# Chuyển quyền sở hữu các file mồ côi đó về cho root
sudo find /opt -nouser -exec chown root:root {} +
```

---

## 4. Phần 3: Quản trị Group toàn diện

Group cho phép gán cùng một bộ quyền hạn cho nhiều người dùng một cách nhất quán.

- **Primary Group (Nhóm chính)**: Nhóm mặc định khi user tạo file mới. Mỗi user chỉ có đúng 1 nhóm chính (lưu ở trường thứ 4 trong `/etc/passwd`).
- **Supplementary / Secondary Groups (Nhóm phụ)**: Các nhóm bổ sung mà user tham gia để chia sẻ quyền hạn với các bộ phận khác.

### 4.1. Xem thông tin Group

```bash
# Xem tất cả các group trên hệ thống
cat /etc/group
# hoặc
getent group

# Xem danh sách tên group
cut -d: -f1 /etc/group

# Xem các group mà một user đang tham gia
groups alice
id alice

# Xem danh sách user thuộc một group cụ thể (ví dụ dev)
getent group dev
```

### 4.2. Tạo Group mới (`groupadd`)

```bash
# Tạo group thông thường
sudo groupadd dev
sudo groupadd tester

# Tạo group với GID định trước (ví dụ 3000)
sudo groupadd -g 3000 sysadmin
```

### 4.3. Chỉnh sửa Group (`groupmod`)

```bash
# Đổi tên group từ dev thành developers
sudo groupmod -n developers dev

# Đổi GID của group
sudo groupmod -g 3005 developers
```

### 4.4. Thêm User vào Group (`usermod` vs `gpasswd`)

Có 2 cách thông dụng, bạn cần nắm rõ:

**Cách 1: Dùng `usermod -aG`**
```bash
sudo usermod -aG dev alice
sudo usermod -aG dev bob
sudo usermod -aG tester bob
sudo usermod -aG tester charlie
```
> [!WARNING]
> Cờ `-a` (append) cực kỳ quan trọng! Nếu bạn gõ nhầm `sudo usermod -G dev alice` (thiếu `-a`), hệ thống sẽ **xóa alice khỏi toàn bộ các nhóm phụ khác** (kể cả nhóm `sudo`, `docker`...) và chỉ giữ lại mỗi nhóm `dev`.

**Cách 2: Dùng `gpasswd -a` (Khuyên dùng - an toàn hơn, không sợ quên `-a`)**
```bash
sudo gpasswd -a alice dev
sudo gpasswd -a bob dev
sudo gpasswd -a bob tester
sudo gpasswd -a charlie tester
```

### 4.5. Xóa User ra khỏi một Group (`gpasswd -d`)

Khi một nhân sự chuyển dự án, bạn muốn gỡ họ khỏi nhóm mà không làm ảnh hưởng các nhóm khác:
```bash
# Cú pháp: sudo gpasswd -d <username> <groupname>
sudo gpasswd -d bob tester
```
Kiểm tra lại:
```bash
groups bob
# Kết quả: bob : bob dev (đã không còn group tester)
```

### 4.6. Xóa Group (`groupdel`)

```bash
sudo groupdel tester
```
> [!NOTE]
> Không thể xóa một group nếu group đó đang là **Primary Group** của bất kỳ user nào còn tồn tại trên hệ thống. Bạn phải đổi primary group của user đó hoặc xóa user trước.

---

## 5. Phần 4: Quản trị quyền Sudo chuyên sâu

Lệnh `sudo` (SuperUser DO) cho phép người dùng được ủy quyền thực thi lệnh dưới quyền `root` mà không cần biết mật khẩu của `root`.

### 5.1. Cấp quyền sudo qua nhóm quản trị mặc định

- Trên **Ubuntu/Debian**: Thành viên nhóm `sudo` mặc định có quyền `sudo`.
  ```bash
  sudo usermod -aG sudo alice
  ```
- Trên **CentOS/RHEL/Fedora**: Thành viên nhóm `wheel` mặc định có quyền `sudo`.
  ```bash
  sudo usermod -aG wheel alice
  ```

### 5.2. File cấu hình `/etc/sudoers` và quy tắc `visudo`

> [!CAUTION]
> Tuyệt đối KHÔNG BAO GIỜ mở file `/etc/sudoers` bằng `nano` hoặc `vim` thông thường! Nếu bạn gõ sai cú pháp và lưu lại, bạn có thể bị khóa quyền `sudo` toàn hệ thống vĩnh viễn. Luôn luôn dùng lệnh:
> ```bash
> sudo visudo
> ```
> `visudo` sẽ tự động kiểm tra cú pháp trước khi lưu. Nếu phát hiện lỗi cú pháp, nó sẽ chặn không cho lưu và cảnh báo bạn sửa.

### 5.3. Cấp quyền chuyên nghiệp bằng file drop-in trong `/etc/sudoers.d/`

Thay vì sửa trực tiếp vào file `/etc/sudoers` gốc, thực tiễn chuẩn của DevOps/Sysadmin là tạo một file riêng trong thư mục `/etc/sudoers.d/`.

Dùng `visudo -f` để tạo và kiểm tra cú pháp:
```bash
sudo visudo -f /etc/sudoers.d/devops-rules
```

**Cấu trúc một quy tắc Sudo:**
```text
User_hoặc_%Group   Host=(RunAsUser:RunAsGroup)   Commands
```

**Một số ví dụ thực tế cấu hình sudo:**

1. Cấp toàn quyền cho user `alice` (phải nhập mật khẩu của `alice`):
   ```text
   alice ALL=(ALL:ALL) ALL
   ```

2. Cấp toàn quyền nhưng KHÔNG cần hỏi mật khẩu (NOPASSWD):
   ```text
   alice ALL=(ALL) NOPASSWD: ALL
   ```

3. Cấp quyền hạn chế cho `bob` chỉ được restart service Nginx và xem log:
   ```text
   bob ALL=(root) /usr/bin/systemctl restart nginx, /usr/bin/journalctl
   ```

### 5.4. Kiểm tra quyền Sudo của User mà không cần chuyển sang User đó

Quản trị viên có thể kiểm tra trực tiếp quyền hạn của một tài khoản:
```bash
# Xem quyền sudo của alice
sudo -l -U alice

# Xem quyền sudo của bob
sudo -l -U bob
```

---

## 6. Phần 5: Kịch bản Lab thực hành tích hợp

Bây giờ, chúng ta sẽ liên kết tất cả kiến thức trên vào một kịch bản dự án thực tế hoàn chỉnh!

### 6.1. Mục tiêu kịch bản
- **3 User**:
  - `alice`: Trưởng nhóm phát triển (Dev Lead), có quyền `sudo`.
  - `bob`: Lập trình viên (Developer) kiêm kiểm thử (Tester), không có quyền `sudo`.
  - `charlie`: Chuyên viên kiểm thử (QA Tester), không có quyền `sudo`.
- **2 Group**:
  - `dev`: Dành cho nhóm phát triển.
  - `tester`: Dành cho nhóm kiểm thử.
- **Thư mục dự án**: `/opt/project`
  - Chỉ thành viên nhóm `dev` mới được phép truy cập, đọc, tạo và sửa file.
  - Người ngoài (`other`) và nhóm `tester` hoàn toàn bị chặn.
  - Mọi file và thư mục con mới tạo bên trong `/opt/project` phải **tự động thuộc về group `dev`** (nhờ cơ chế `setgid`).

---

### 6.2. Bước 1: Khởi tạo User và Group

```bash
# 1. Tạo 2 group
sudo groupadd dev
sudo groupadd tester

# 2. Tạo 3 user kèm home và shell bash
sudo useradd -m -s /bin/bash -c "Alice Dev Lead" alice
sudo useradd -m -s /bin/bash -c "Bob Developer" bob
sudo useradd -m -s /bin/bash -c "Charlie Tester" charlie

# 3. Đặt mật khẩu cho 3 user (ví dụ đặt: Password123!)
echo "alice:Password123!" | sudo chpasswd
echo "bob:Password123!" | sudo chpasswd
echo "charlie:Password123!" | sudo chpasswd

# 4. Gán user vào các group tương ứng
sudo gpasswd -a alice dev
sudo gpasswd -a bob dev
sudo gpasswd -a bob tester
sudo gpasswd -a charlie tester

# 5. Cấp quyền sudo cho alice
# (Ubuntu/Debian dùng group sudo, RHEL/CentOS dùng group wheel)
if grep -q "^sudo:" /etc/group; then
    sudo usermod -aG sudo alice
else
    sudo usermod -aG wheel alice
fi
```

**Kiểm tra lại cấu hình:**
```bash
id alice
id bob
id charlie
```
*Kết quả mong muốn:*
- `alice`: `groups=...,dev,sudo`
- `bob`: `groups=...,dev,tester`
- `charlie`: `groups=...,tester`

---

### 6.3. Bước 2: Tạo thư mục dự án và thiết lập quyền `setgid`

```bash
# Tạo thư mục dự án
sudo mkdir -p /opt/project

# Đổi sở hữu sang root và group dev
sudo chown root:dev /opt/project

# Thiết lập quyền 2770
sudo chmod 2770 /opt/project
```

#### Giải thích mã quyền `2770`:
- `2` (Chữ số đặc biệt): Bật cờ **`setgid`** (Set Group ID).
  - Khi `setgid` được bật trên một thư mục, bất kỳ file hoặc thư mục con nào được tạo ra bên trong thư mục này sẽ **tự động kế thừa Group của thư mục cha (`dev`)**, thay vì lấy primary group của người tạo file.
- `7` (Chữ số Owner - `root`): Đọc (`4`) + Ghi (`2`) + Thực thi/Truy cập (`1`) = `rwx`.
- `7` (Chữ số Group - `dev`): Đọc (`4`) + Ghi (`2`) + Thực thi/Truy cập (`1`) = `rwx`.
- `0` (Chữ số Others - người khác): Không có bất kỳ quyền nào = `---`.

Kiểm tra bằng lệnh:
```bash
ls -ld /opt/project
```
Kết quả hiển thị:
```text
drwxrws--- 2 root dev 4096 ... /opt/project
      ^
      Chữ 's' ở quyền nhóm đại diện cho setgid đã kích hoạt!
```

---

### 6.4. Bước 3: Kiểm thử phân quyền truy cập thực tế

#### Test 1: Đăng nhập bằng `alice` (Thuộc group `dev`)
```bash
su - alice
id
# Tạo file trong dự án
touch /opt/project/alice_app.py
echo "print('Hello Dev')" > /opt/project/alice_app.py
ls -l /opt/project/alice_app.py
exit
```
*Kết quả mong muốn:*
- File `/opt/project/alice_app.py` được tạo thành công.
- File có owner là `alice`, nhưng group tự động là `dev`:
  `-rw-r--r-- 1 alice dev ... alice_app.py`

#### Test 2: Đăng nhập bằng `bob` (Thuộc group `dev`)
```bash
su - bob
id
# Tạo thêm file và đọc file của alice
touch /opt/project/bob_module.py
cat /opt/project/alice_app.py
ls -la /opt/project
exit
```
*Kết quả mong muốn:*
- `bob` truy cập bình thường, đọc được file của `alice` và tạo được file mới thuộc group `dev`.

#### Test 3: Đăng nhập bằng `charlie` (Chỉ thuộc group `tester`, KHÔNG thuộc `dev`)
```bash
su - charlie
id
# Thử liệt kê file trong thư mục
ls /opt/project
# Thử ghi file vào thư mục
touch /opt/project/charlie_hack.txt
exit
```
*Kết quả mong muốn:*
```text
ls: cannot open directory '/opt/project': Permission denied
touch: cannot touch '/opt/project/charlie_hack.txt': Permission denied
```
`charlie` bị chặn hoàn toàn!

---

### 6.5. Bước 4: Kiểm thử quyền Sudo

#### Test Sudo với `alice`:
```bash
su - alice
sudo whoami
exit
```
*Kết quả:* Trả về `root` (thành công).

#### Test Sudo với `bob`:
```bash
su - bob
sudo whoami
exit
```
*Kết quả:*
```text
bob is not in the sudoers file. This incident will be reported.
```
`bob` không được phép dùng sudo!

---

### 6.6. Bước 5: Kiểm thử tính kế thừa của thư mục con

Khi `alice` hoặc `bob` tạo một thư mục con, thư mục con đó cũng phải tự động có `setgid` và thuộc group `dev`:

```bash
su - alice
mkdir /opt/project/backend
touch /opt/project/backend/server.py
ls -ld /opt/project/backend
ls -l /opt/project/backend/server.py
exit
```
*Kết quả:*
- Thư mục `/opt/project/backend` có quyền `drwxrwsr-x` thuộc sở hữu `alice:dev`.
- File `server.py` bên trong cũng thuộc group `dev`.

---

## 7. Phần 6: Khắc phục sự cố thường gặp (Troubleshooting)

### 7.1. Đã thêm user vào group nhưng vẫn bị "Permission Denied"
- **Nguyên nhân**: Session hiện tại của user được sinh ra trước khi bạn chạy `usermod`. Hệ thống Linux chỉ nạp thông tin group khi khởi tạo session đăng nhập mới.
- **Khắc phục**:
  - Đăng xuất (`exit` hoặc `logout`) rồi đăng nhập lại.
  - Hoặc làm mới session bằng lệnh:
    ```bash
    su - <username>
    # hoặc kích hoạt group trong shell hiện tại
    newgrp <groupname>
    ```

### 7.2. Lỗi "userdel: user ... is currently used by process ..."
- **Nguyên nhân**: User vẫn đang có ứng dụng, shell hoặc tiến trình SSH chạy nền.
- **Khắc phục**:
  ```bash
  sudo pkill -9 -u <username>
  sudo userdel -r <username>
  ```

### 7.3. Vô tình làm sai cú pháp sudoers dẫn đến mất quyền `sudo`
- **Cách xử lý**:
  Nếu bạn còn một phiên `root` đang mở, hãy gõ `visudo` để sửa ngay.
  Nếu bị khóa hoàn toàn, cần khởi động lại máy vào chế độ **Recovery Mode** (chọn root shell trong menu GRUB) và chạy:
  ```bash
  visudo -c   # Kiểm tra file nào bị lỗi cú pháp
  nano /etc/sudoers.d/<file-bi-loi>
  ```

### 7.4. Các thành viên trong cùng group không sửa được file của nhau
- **Nguyên nhân**: Giá trị `umask` mặc định của user thường là `0022`, khiến file mới tạo ra có quyền `644` (`rw-r--r--`, group chỉ có quyền đọc `r`, không có quyền ghi `w`).
- **Khắc phục**:
  - Khi cần cùng chỉnh sửa, user có thể chạy:
    ```bash
    chmod g+w /opt/project/<filename>
    ```
  - Hoặc cấu hình `umask 0002` trong file `~/.bashrc` của user để mọi file tạo ra mặc định có quyền `rw-rw-r--`.

---

## 8. Phần 7: Bảng tra cứu lệnh nhanh (Cheat Sheet)

### Quản lý User

| Thao tác | Câu lệnh |
| :--- | :--- |
| Xem danh sách tất cả user | `getent passwd` hoặc `cut -d: -f1 /etc/passwd` |
| Xem user thường (UID >= 1000) | `awk -F: '$3>=1000 && $3<65534 {print $1, $3, $7}' /etc/passwd` |
| Xem user đang đăng nhập | `who` hoặc `w` |
| Xem UID/GID của user | `id <user>` |
| Tạo user kèm Home và Bash | `sudo useradd -m -s /bin/bash -c "Tên" <user>` |
| Đặt mật khẩu cho user | `sudo passwd <user>` |
| Bắt buộc đổi mật khẩu lần sau | `sudo passwd -e <user>` |
| Đổi Shell của user | `sudo usermod -s /bin/bash <user>` |
| Khóa tài khoản | `sudo usermod -L <user>` |
| Mở khóa tài khoản | `sudo usermod -U <user>` |
| Dừng tiến trình của user | `sudo pkill -9 -u <user>` |
| Xóa user giữ lại Home | `sudo userdel <user>` |
| Xóa user kèm Home và Mail | `sudo userdel -r <user>` |
| Tìm file mồ côi (chủ bị xóa) | `sudo find / -nouser 2>/dev/null` |

### Quản lý Group & Sudo

| Thao tác | Câu lệnh |
| :--- | :--- |
| Xem danh sách group | `getent group` |
| Xem nhóm của 1 user | `groups <user>` |
| Tạo group | `sudo groupadd <groupname>` |
| Xóa group | `sudo groupdel <groupname>` |
| Thêm user vào group phụ | `sudo gpasswd -a <user> <group>` hoặc `sudo usermod -aG <group> <user>` |
| **Xóa user khỏi group** | `sudo gpasswd -d <user> <group>` |
| Cấp quyền Sudo (Ubuntu) | `sudo usermod -aG sudo <user>` |
| Cấp quyền Sudo (CentOS/RHEL) | `sudo usermod -aG wheel <user>` |
| Sửa cấu hình Sudoers an toàn | `sudo visudo` |
| Tạo file cấu hình Sudo riêng | `sudo visudo -f /etc/sudoers.d/<filename>` |
| Xem quyền Sudo của 1 user | `sudo -l -U <user>` |
| Gán quyền sở hữu thư mục | `sudo chown root:<group> <path>` |
| Bật Setgid chia sẻ nhóm | `sudo chmod 2770 <path>` |

---

## 9. Phần 8: Dọn dẹp môi trường Lab

Sau khi hoàn thành bài thực hành, bạn có thể hoàn trả hệ thống về trạng thái sạch sẽ ban đầu:

```bash
# 1. Đảm bảo thoát hết các phiên làm việc của các user lab
sudo pkill -9 -u alice 2>/dev/null
sudo pkill -9 -u bob 2>/dev/null
sudo pkill -9 -u charlie 2>/dev/null

# 2. Xóa sạch 3 user và thư mục home tương ứng
sudo userdel -r alice 2>/dev/null
sudo userdel -r bob 2>/dev/null
sudo userdel -r charlie 2>/dev/null

# 3. Xóa các group đã tạo
sudo groupdel dev 2>/dev/null
sudo groupdel tester 2>/dev/null

# 4. Xóa thư mục dự án lab
sudo rm -rf /opt/project

# 5. Kiểm tra lại hệ thống
id alice 2>&1
getent group dev 2>&1
ls -d /opt/project 2>&1
```

Nếu hệ thống báo `no such user`, `no such group` và `No such file or directory` nghĩa là bạn đã dọn dẹp môi trường lab thành công và sạch sẽ!

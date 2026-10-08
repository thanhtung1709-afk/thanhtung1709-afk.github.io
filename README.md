# PersistenceIsFutile
## Malicious SSH Authorized Key (Remote Access)
Linux server production gần như chắc chắn sử dụng SSH, nên đây là nơi đầu tiên cần kiểm tra.
![image](https://hackmd.io/_uploads/rJxkOnVsGe.png)
Trong challenge có `nobody@nothing` (abnormaly key) -> Attacker có thể SSH vào mà không cần password => SSH persistence / remote access backdoor. `sshd` tự động đọc `authorized_keys`
```
Attacker
   │
   │ SSH + private key
   ▼
sshd
   │
   ▼
root/user account
```
Xóa key lạ
![image](https://hackmd.io/_uploads/Hk_bah4ifl.png)
## SSH Key Persistence via cron.daily (Persistence)
chạy ls crontab 
![image](https://hackmd.io/_uploads/SJ6FwpVofe.png)
thấy 2 file access-up và pyssh khả nghi, cat pyssh
![image](https://hackmd.io/_uploads/ByjdD64sGx.png)
thấy nó gọi  /lib/python3/dist-packages/ssh_import_id_update, cat file này 
![image](https://hackmd.io/_uploads/rkhavTVsfg.png)
Sau khi decode 
![image](https://hackmd.io/_uploads/S1uQup4sze.png)
đoạn bash này sẽ thêm lại khóa SSH  vào tệp authorized_keys của người dùng root, rm 2 file đó, chạy /root/solveme
![image](https://hackmd.io/_uploads/B1Wc_pEofx.png)
##  Alias Reverse Shell in .bashrc (Remote Access)
Vì `.bashrc` được shell load khi user mở interactive shell.
![image](https://hackmd.io/_uploads/SJyECnNoGe.png)
```

User chạy cat
       │
       ▼
Bash thực thi alias
       │
       ▼
Network connection
       │
       ▼
Attacker

login
  ↓
.bashrc
  ↓
malicious alias loaded
  ↓
user executes command
  ↓
reverse shell
```
Remove the malicious alias line 
![image](https://hackmd.io/_uploads/H1erJpEsGl.png)
##  Bind Shell via alertd in root .bashrc (Remote Access)
Khi sudo su chay ps auxf thì thấy process chạy alertd bind shell port 4444
![image](https://hackmd.io/_uploads/S16EfaVjfx.png)
kill process
![image](https://hackmd.io/_uploads/Hy7PmpEjzl.png)
xóa line alertd trong /root/.bashrc
![image](https://hackmd.io/_uploads/S1rR7pEoGg.png)
rm command alertd
![image](https://hackmd.io/_uploads/S1gBEaViMe.png)
chay /root/solveme
![image](https://hackmd.io/_uploads/ryqD4TVjfg.png)
## Reverse Shell via MOTD (Remote Access)
Khi chạy ps auxf có bash thực thi file connectivity-check
![image](https://hackmd.io/_uploads/SJp3EaEjGg.png)
Hệ thống MOTD (Message of the Day) thực thi các tập lệnh trong thư mục `/etc/update-motd.d/` mỗi khi đăng nhập qua SSH. Một tập lệnh độc hại đã được thêm vào để khởi chạy một reverse shell duy trì kết nối, trong file connectivity-check có chứa
![image](https://hackmd.io/_uploads/r1fIrTEoMe.png)
rm các file liên quan, sau đó kill process 
![image](https://hackmd.io/_uploads/BkmQUT4jGg.png)
chạy /root/solveme
![image](https://hackmd.io/_uploads/Hk2SLpEsMe.png)
## DNS-based Cron Command Execution (Remote Access)
chạy `crontab -l -u user` để hiển thị danh sách các tác vụ tự động đang được lên lịch chạy cho tài khoản có tên là user
![image](https://hackmd.io/_uploads/S17vKTVoMg.png)
cronjob sẽ truy vấn bản ghi TXT từ máy chủ DNS của kẻ tấn công và thực thi bất kỳ lệnh nào được trả về, xóa hoàn toàn crontab của user bằng lệnh `crontab -u user -r`, chạy /root/solveme
![image](https://hackmd.io/_uploads/rJx6KpEszl.png)
## SUID Bash Creator via cron.daily (Persistence)
Kiểm tra các scheduled task, phát hiện /etc/cron.daily/access-up. Script này được cron thực thi định kỳ và tạo các bản sao của /bin/bash với tên ngẫu nhiên trong /bin hoặc /sbin, sau đó đặt SUID cho các bản sao. Vì owner của các bản sao là root và SUID được bật, người dùng thường có thể thực thi chúng với effective UID của root. Script cũng sử dụng touch -r /bin/bash để sao chép timestamp của Bash gốc, gây khó khăn cho việc phát hiện chỉ dựa trên mtime. Do đó cần loại bỏ cả script tạo SUID (access-up) lẫn các SUID payload đã được tạo ra.
![image](https://hackmd.io/_uploads/ry4NcpEiMl.png)
## SUID Binaries (Privilege Escalation)
Enumerate các file SUID bất thường và phát hiện các binary có tên ngẫu nhiên như /usr/bin/dlxcrw, /usr/bin/mgxttm, /usr/sbin/afdluk, /usr/sbin/ppppd và /home/user/.backdoor. Các file này là các bản sao/payload SUID được tạo bởi cơ chế access-up. Vì chúng chạy với quyền của owner root, chúng cung cấp một đường privilege escalation cho user thường. 
tìm các file có quyền SUID được sinh ra ngẫu nhiên do script trên với quyền SUID
![image](https://hackmd.io/_uploads/SkeXo6EoGg.png)
gồm 5 file:
- /usr/bin/dlxcrw
- /usr/bin/mgxttm
- /usr/sbin/afdluk
- /usr/sbin/ppppd
- /home/user/.backdoor
xóa 5 file này 
![image](https://hackmd.io/_uploads/ry7nhaVoGx.png)
![image](https://hackmd.io/_uploads/rko0nTNsMe.png)
## gnats User Modification (Privilege Escalation)
Kiểm tra `/etc/passwd` phát hiện account gnats có thuộc tính bất thường. Attacker không nhất thiết phải tạo account mới mà có thể sửa một service account tồn tại để tạo access/persistence. GID của gnats bị thay đổi từ 41 thành 0, tức primary group bị chuyển sang root group; đồng thời shell/credential của account có thể bị thay đổi. Đây là dấu hiệu account manipulation và cần khôi phục về trạng thái hợp lệ.
![image](https://hackmd.io/_uploads/ByGYTaEsfg.png)
dùng `grep -i "/bash"` để xem user nào có thể dùng shell
![image](https://hackmd.io/_uploads/rJV06aVjfe.png)
Primary group của gnats bị thay đổi thành GID 0 (root group), tạo ra quyền group-level bất thường và là một dấu hiệu compromise.
![image](https://hackmd.io/_uploads/BkezAaEsMx.png)

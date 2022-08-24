# 1. Check quyền Users hiện tại

`whoami /priv`

Nếu có quyền `SeImpersonatePrivilege` hoặc `SeAssignPrimaryTokenPrivilege` có thể sử dụng tool [PrintSpoofer](https://github.com/itm4n/PrintSpoofer) để leo thang đặc quyền.

Nếu là user "nt authority\local service" hoặc "nt authority\local network" sử dụng [FullPowers](https://github.com/itm4n/FullPowers) để leo thang đặc quyền

# 2. Các cách leo thang đặc quyền

https://tryhackme.com/room/windows10privesc


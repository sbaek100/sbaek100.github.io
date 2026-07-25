---
title: "[네트워크 보안] 04. TCP/UDP와 포트 - 어떤 프로그램과 대화하는가"
date: 2026-06-10 13:00:00 +0900
categories:
  - 1.응용강의
  - 네트워크보안
  - TCPUDP
tags:
  - TCP
  - UDP
  - Wireshark
  - 3wayhandshake
  - hping3
math: false
mermaid: true
---

> **학습목표**
> 1. TCP의 다섯 가지 특징과 헤더·플래그의 의미를 설명할 수 있다.
> 2. Wireshark로 3-way 핸드셰이크와 연결 종료 과정을 관찰할 수 있다.
> 3. hping3로 RST·FIN·PSH 플래그를 직접 주입해 서버 반응을 분석할 수 있다.
> 4. UDP의 특징과 TCP와의 차이, 그리고 UDP를 노리는 위협을 설명할 수 있다.
{: .prompt-info }

> **이번 실습 대상**: Metasploitable2 `192.168.0.100`의 DVWA(`http://192.168.0.100/dvwa/`, 로그인 `admin/password`). 공격자·분석자는 Kali `192.168.0.10`.
{: .prompt-info }

IP 주소가 “어느 컴퓨터인가”를 가리킨다면, 포트 번호는 “그 컴퓨터 안의 어느 프로그램인가”를 가리킵니다. 앞 장에서 만든 실습실 위에서, 이제 통신의 속을 들여다봅니다. 같은 웹 요청도 TCP는 “연결을 맺고” 주고받고, UDP는 “그냥 보냅니다”. 이 차이가 보안에서 왜 중요한지 Wireshark로 직접 확인합니다.

---

## 1. TCP란 — 신뢰할 수 없는 IP 위에 신뢰를 얹다

TCP를 이해하려면 그 아래에 있는 IP를 먼저 떠올려야 합니다. 인터넷에서 데이터를 실제로 나르는 IP는 “최선을 다하되 책임은 지지 않는(best-effort)” 방식입니다. 편지를 부치듯 목적지로 보내기는 하지만, 도중에 사라져도, 순서가 뒤바뀌어 도착해도, 같은 편지가 두 번 도착해도 IP는 알려 주지 않습니다. 이대로라면 웹 페이지나 파일이 조금씩 깨지거나 뒤섞여 도착할 것입니다. 바로 이 불안정함을 메우기 위해 등장한 것이 **TCP**입니다.

그래서 TCP는 “신뢰할 수 없는 IP 위에서 신뢰할 수 있는 양방향 바이트 스트림을 제공하는 프로토콜”로 정의됩니다. 받은 쪽이 “잘 받았다(ACK)”고 답하게 하고, 답이 없으면 다시 보내고, 조각마다 번호를 붙여 순서를 맞추고, 상대가 감당할 만큼만 속도를 조절합니다. 이런 절차의 대가로 TCP는 “보낸 데이터가 반드시, 순서대로, 온전히 도착한다”는 약속을 줍니다.

| 특징 | 설명 |
|---|---|
| 연결 지향 | 데이터 전송 전에 3-way 핸드셰이크로 연결을 맺습니다. |
| 신뢰성 | 확인 응답(ACK)과 재전송으로 손실 없이 전달합니다. |
| 순서 보장 | 시퀀스 번호로 조각을 원래 순서대로 재조립합니다. |
| 흐름 제어 | 윈도우 크기로 받는 쪽이 감당할 만큼만 보냅니다. |
| 혼잡 제어 | 네트워크 혼잡을 감지해 전송 속도를 조절합니다. |

## 2. TCP 헤더와 플래그

TCP 헤더에는 출발지·목적지 포트(각 16비트), 시퀀스 번호(32비트), 확인 응답 번호(32비트), 제어 플래그, 윈도우 크기 등이 담깁니다. 그중 연결의 상태를 좌우하는 것이 **제어 플래그**입니다.

제어 플래그는 TCP 헤더 안의 여러 ‘스위치(1비트 깃발)’로, 이 패킷이 어떤 의미인지를 상대에게 알리는 신호등입니다. TCP는 “연결하자”, “받았다”, “끊자” 같은 신호를 별도 메시지 대신 플래그 비트를 켜고 끄는 방식으로 아주 경제적으로 전달합니다. 그래서 아래 플래그만 읽을 줄 알면, Wireshark에서 패킷 하나하나가 ‘무슨 말을 하고 있는지’를 알아볼 수 있습니다.

| 플래그 | 의미 | 한마디로 |
|---|---|---|
| SYN | 연결 시작 요청 | "연결하자" |
| ACK | 확인 응답 | "받았어" |
| FIN | 정상 연결 종료 | "이제 끊자" |
| RST | 연결 강제 종료 | "당장 끊어" |
| PSH | 버퍼링 없이 즉시 전달 | "바로 처리해" |
| URG | 긴급 데이터 표시 | "급한 거야" |

![Wireshark 패킷 상세 — Transmission Control Protocol을 펼치면 각 플래그의 설정 상태가 보입니다](/assets/img/posts/2026-06-10-networkset-04-tcp-06.png)

## 3. 실습 — Wireshark로 3-way 핸드셰이크 관찰

TCP 연결은 `SYN → SYN·ACK → ACK` 세 단계로 시작됩니다. 왜 한 번에 연결하지 않고 세 번이나 인사를 주고받을까요? 두 컴퓨터가 데이터를 순서대로 맞추려면, 서로 “나는 몇 번부터 셀게”라는 시작 번호(시퀀스 번호)를 교환하고, 상대도 그 번호를 제대로 받았음을 확인해야 하기 때문입니다. 양방향 모두 확인이 끝나야 비로소 믿을 수 있는 통신이 시작됩니다. 이 핸드셰이크는 뒤에서 배울 SYN 스캔([05장](/posts/networkset-05-scan-recon/))과 SYN 플러딩([09장](/posts/networkset-09-dos-ddos/))의 원리이기도 하므로, 눈으로 직접 확인해 둡니다.

> **실습 4-1. 3-way 핸드셰이크 캡처하기**  
> **목표** Kali에서 Metasploitable2의 웹(80)에 접속할 때 오가는 3-way 핸드셰이크를 캡처한다. **대상** Kali(192.168.0.10) → Metasploitable2(192.168.0.100) DVWA.
{: .prompt-tip }

**1단계 — Wireshark 실행과 캡처 필터** (Kali)

```bash
sudo wireshark
# 캡처 필터(두 VM 사이의 TCP만):
tcp && host 192.168.0.100
```

> **왜?** Wireshark는 인터페이스를 지나는 모든 패킷을 잡습니다. 필터로 두 VM 사이의 TCP만 남겨야 핸드셰이크가 눈에 잘 띕니다. `eth0` 인터페이스를 선택해 캡처를 시작합니다.

![Wireshark 메인 화면 — 캡처할 인터페이스(eth0) 선택](/assets/img/posts/2026-06-10-networkset-04-tcp-01.png)

**2단계 — 접속으로 핸드셰이크 유발** (Kali 브라우저)

```text
http://192.168.0.100/dvwa
```

> **왜?** 웹 접속을 시도하면 그 즉시 TCP 3-way 핸드셰이크가 일어납니다. 관찰할 트래픽을 만들어 내는 단계입니다.

![DVWA 로그인 페이지 — 접속하면 TCP 연결이 발생](/assets/img/posts/2026-06-10-networkset-04-tcp-02.png)

**3단계 — 세 단계 확인** (Wireshark 패킷 목록)

```text
[SYN]       192.168.0.10  → 192.168.0.100   (Flags 0x002)
[SYN, ACK]  192.168.0.100 → 192.168.0.10    (Flags 0x012)
[ACK]       192.168.0.10  → 192.168.0.100   (Flags 0x010)
```

> **왜?** Info 열의 `[SYN]`·`[SYN, ACK]`·`[ACK]` 표시로 세 단계를 확인합니다. 각 패킷을 클릭해 Flags 상세와 시퀀스 번호가 어떻게 이어지는지 살펴봅니다.

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-06-10-networkset-04-tcp-03.png" alt="첫 SYN 패킷" style="max-width:48%">
<img src="/assets/img/posts/2026-06-10-networkset-04-tcp-04.png" alt="3-way handshake 세 패킷" style="max-width:48%">
</div>

> **참고 — 연결 종료는 4-way** 연결을 끊을 때는 네 단계(FIN → ACK → FIN → ACK)를 거칩니다. 필터 `tcp.flags.fin == 1`로 종료 패킷만 모아 보면 양쪽이 각자 "끊자"(FIN)와 "알았어"(ACK)를 주고받는 과정을 볼 수 있습니다.
{: .prompt-info }

## 4. 실습 — hping3로 플래그 직접 주입

hping3는 원하는 플래그를 가진 TCP 패킷을 직접 만들어 보내는 도구입니다. 평소 운영체제는 정해진 절차대로만 패킷을 만들어, 사용자가 “RST 플래그만 켠 패킷을 하나 보내라”처럼 비정상적인 요청을 할 방법이 없습니다. hping3는 이 한계를 넘어 플래그·포트·출발지 주소·데이터 크기까지 원하는 대로 조립한 패킷을 쏘게 해 줍니다. 공격자는 이 능력으로 방화벽을 떠보고, 분석가는 “이런 패킷이 오면 서버가 어떻게 반응하는지”를 안전하게 실험합니다.

| 옵션 | 의미 |
|---|---|
| `-S` / `-R` / `-F` / `-P` | SYN / RST / FIN / PSH 플래그 |
| `-p 80` / `-s 45678` | 대상 포트 / 출발지 포트 지정 |
| `-d 100` | 데이터(페이로드) 크기(바이트) |

> **실습 4-2. RST·FIN·PSH 플래그 주입 관찰**  
> **목표** hping3로 RST·FIN·PSH 패킷을 보내고 각 플래그와 서버 반응을 확인한다. **대상** Kali → Metasploitable2(192.168.0.100) 80번 포트.
{: .prompt-tip }

**RST 주입** — "당장 끊어" 강제 종료 신호

```bash
sudo hping3 -R -p 80 -s 45678 192.168.0.100
```

> **왜?** 정상 연결이라면 서버가 즉시 연결을 폐기합니다. 필터 `tcp.flags.reset == 1`로 RST 패킷(Flags 0x004)을 확인합니다.

![Wireshark에 잡힌 RST 패킷(빨강) — 연결이 즉시 종료됨](/assets/img/posts/2026-06-10-networkset-04-tcp-05.png)

**FIN 주입** — "정상적으로 끊자"

```bash
sudo hping3 -F -p 80 -s 45678 192.168.0.100
```

> **왜?** RST가 즉시 끊는 것과 달리 FIN은 종료 절차를 시작합니다. 둘의 서버 반응 차이를 비교합니다.

**PSH 주입** — "버퍼에 쌓지 말고 바로 처리해"(데이터 포함)

```bash
sudo hping3 -P -p 80 -d 100 192.168.0.100
```

> **왜?** PSH는 실제 데이터와 함께 쓰입니다. 필터 `tcp.flags.push == 1`로 PSH·ACK(0x018)와 페이로드(Len>0)를 확인합니다. (일부 자료에 PSH 명령이 `-R`로 잘못 적힌 경우가 있으니, PSH는 반드시 `-P`를 씁니다.)

![PSH+ACK 패킷 상세 — 실제 데이터를 포함](/assets/img/posts/2026-06-10-networkset-04-tcp-07.png)

### 연결 상태와 그래프로 보기

리눅스에서 연결 상태를, Wireshark의 `Statistics > TCP Stream Graphs`로 시퀀스·윈도우 변화를 시각화하면 전송 속도·지연·재전송·흐름 제어가 그래프로 드러납니다.

```bash
watch -n 1 "netstat -ant | grep 192.168.0.100"   # ESTABLISHED, TIME_WAIT 등 관찰
```

![Wireshark Statistics 메뉴 — 대화·프로토콜 계층·그래프 도구](/assets/img/posts/2026-06-10-networkset-04-tcplab-01.png)

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-06-10-networkset-04-tcplab-02.png" alt="Sequence Numbers(Stevens) 그래프" style="max-width:48%">
<img src="/assets/img/posts/2026-06-10-networkset-04-tcplab-03.png" alt="Window Scaling 그래프" style="max-width:48%">
</div>

## 5. TCP를 노리는 공격과 방어

역설적이게도, TCP를 믿을 수 있게 만들어 주는 바로 그 연결 구조가 공격의 표적이 됩니다. 세 공격 모두 TCP의 정상 동작을 흉내 내기 때문에, 방어의 핵심도 ‘위조·남용을 어렵게 만드는 것’ — 암호화 채널(SSL/TLS), SYN 쿠키, 타임아웃 축소에 있습니다.

| 공격 | 원리 | 방어 | 관련 차시 |
|---|---|---|---|
| SYN Flooding | 응답 없는 SYN을 대량 전송해 백로그 큐 고갈 | SYN 쿠키, 타임아웃 축소 | [09장](/posts/networkset-09-dos-ddos/) |
| RST 공격 | 위조 RST로 정상 연결 강제 종료 | 암호화 채널(SSL/TLS) | [07장](/posts/networkset-07-spoofing-mitm/) |
| 세션 하이재킹 | 시퀀스 번호를 예측해 세션 탈취 | 암호화, 세션 타임아웃 축소 | [07장](/posts/networkset-07-spoofing-mitm/) |

![TCP 상태 전이 흐름도 — 연결 수립부터 종료까지](/assets/img/posts/2026-06-10-networkset-04-tcp-08.png)

---

## 6. UDP란 — 속도와 단순함을 택하다

TCP가 신뢰성을 위해 여러 절차를 거친다면, UDP는 속도와 단순함을 택합니다. 이렇게 허술해 보이는 UDP는 왜 존재할까요? TCP의 신뢰성은 공짜가 아니기 때문입니다. 인터넷 전화나 실시간 영상처럼 “조금 끊겨도 좋으니 무조건 빨라야 하는” 통신에서는, 잃어버린 조각을 다시 보내느라 지연되는 것이 오히려 방해가 됩니다. 늦게 도착한 목소리 조각은 이미 쓸모가 없기 때문입니다. DNS처럼 “질문 한 번, 대답 한 번”으로 끝나는 짧은 통신에서도 연결을 맺고 끊는 절차가 배보다 배꼽이 큽니다.

| 특징 | 설명 |
|---|---|
| 비연결성 | 핸드셰이크 없이 바로 전송합니다. |
| 신뢰성 없음 | 확인 응답·재전송이 없습니다(손실될 수 있음). |
| 순서 보장 없음 | 도착 순서가 뒤바뀔 수 있습니다. |
| 가벼움 | 헤더가 8바이트로 작습니다(TCP는 20바이트~). |
| 빠름 | 절차가 없어 지연이 적습니다. |

UDP 헤더는 출발지 포트·목적지 포트·길이·체크섬 네 필드뿐입니다. TCP처럼 SYN·ACK 같은 플래그가 없다는 점이 핵심입니다. 이 선택의 차이는 보안에도 그대로 이어집니다 — 연결 상태를 추적하는 TCP와 달리, 상태도 확인 절차도 없는 UDP는 출발지 위조와 증폭 공격에 훨씬 취약합니다.

| 구분 | TCP | UDP |
|---|---|---|
| 연결 | 3-way 핸드셰이크 | 없음 |
| 신뢰성 | 있음(ACK·재전송) | 없음 |
| 헤더 크기 | 20바이트 이상 | 8바이트 |
| 용도 | 웹·메일·파일전송 | DNS·스트리밍·VoIP |

## 7. 실습 — Wireshark로 UDP 통신 관찰

가장 흔한 UDP 서비스인 DNS를 통해 UDP의 요청-응답을 관찰합니다.

> **실습 4-3. DNS 질의로 보는 UDP 통신**  
> **목표** DNS 질의를 보내고 Wireshark로 UDP 요청-응답 한 쌍을 관찰한다. **대상** Kali → Metasploitable2(192.168.0.100) DNS(53).
{: .prompt-tip }

```bash
# 필터: udp && ip.dst_host == 192.168.0.100
dig @192.168.0.100 example.com
```

> **왜?** `@192.168.0.100`은 그 주소를 DNS 서버로 지정한다는 뜻입니다. UDP는 연결이 없으므로 질의와 응답을 **Transaction ID**로 짝짓습니다. 응답 패킷은 출발지·목적지 포트가 요청과 반대로 뒤집혀 있고, 두 패킷의 시간차가 곧 조회 시간입니다.

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-01.png" alt="UDP 캡처 필터 설정" style="max-width:32%">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-02.png" alt="nmap -sU -p53 결과" style="max-width:32%">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-03.png" alt="Wireshark UDP 캡처" style="max-width:32%">
</div>

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-04.png" alt="UDP 패킷 상세(포트·길이·체크섬, 플래그 없음)" style="max-width:32%">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-07.png" alt="DNS 질의 패킷 상세(Transaction ID)" style="max-width:32%">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-06.png" alt="dig 실행 결과(Answer 섹션)" style="max-width:32%">
</div>

**UDP 기반 파일 전송(TFTP)** — TFTP(69/udp)는 초기 요청은 69번, 실제 전송은 임의 포트로 전환하며 블록 단위 ACK로 신뢰성을 애플리케이션 계층에서 구현합니다.

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-08.png" alt="TFTP 패킷 흐름" style="max-width:32%">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-09.png" alt="TFTP Write Request 상세" style="max-width:32%">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-10.png" alt="여러 UDP 서비스 패킷" style="max-width:32%">
</div>

## 8. UDP를 노리는 위협

플래그도 연결 상태도 없는 UDP는 출발지 위조와 증폭에 취약합니다. TCP라면 3-way 핸드셰이크로 “정말 네가 보낸 게 맞는지”를 확인하지만, UDP는 이런 확인 절차가 없어 봉투의 보내는 사람 주소를 아무렇게나 적어 보내도 그대로 전달됩니다. 공격자는 출발지 주소를 ‘피해자’로 위조해 여러 서버에 작은 질의를 보내, 서버들의 커다란 응답이 모두 피해자에게 쏟아지게 만듭니다 — 이것이 반사·증폭 공격(DRDoS)의 원리입니다([09장](/posts/networkset-09-dos-ddos/)).

> **실습 4-4. UDP 포트 스캔과 스푸핑 관찰**  
> **목표** UDP 포트 스캔의 응답 방식과, 출발지를 위조한 스푸핑 패킷을 관찰한다. **대상** Kali → Metasploitable2(192.168.0.100).
{: .prompt-tip }

```bash
sudo nmap -sU -p 1-100 192.168.0.100        # UDP 스캔
sudo hping3 --udp -a 10.0.0.1 -s 53 -p 53 192.168.0.100   # 출발지 IP 위조
```

> **왜?** UDP 스캔은 응답으로 포트 상태를 짐작합니다. 닫힌 포트는 “ICMP 포트 도달 불가(Type 3, Code 3)”를 돌려주고 열린 포트는 대개 응답이 없습니다(필터 `icmp.type == 3 and icmp.code == 3`). `-a`로 출발지 IP를 속이면 Wireshark에 출발지가 `10.0.0.1`로 찍히는데, UDP는 연결 확인이 없어 이런 위조가 쉽고 이것이 DRDoS의 토대가 됩니다.

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-05.png" alt="닫힌 UDP 포트의 ICMP Port Unreachable 응답" style="max-width:32%">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-11.png" alt="ICMP 오류로 포트 상태 유추" style="max-width:32%">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-12.png" alt="출발지가 위조된 UDP 패킷" style="max-width:32%">
</div>

| UDP 취약점 | 설명 |
|---|---|
| 인증 부재 | 출발지 위조(스푸핑)가 쉬움 |
| 증폭 공격 | 작은 요청 → 큰 응답(DNS/NTP) → DDoS |
| 스캔 탐지 어려움 | 명확한 응답 패턴 없음 |

응답 시간·통계도 분석에 활용합니다.

<div style="display:flex;gap:8px;flex-wrap:wrap">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-13.png" alt="dig 응답 시간 측정" style="max-width:48%">
<img src="/assets/img/posts/2026-06-10-networkset-04-udp-14.png" alt="Wireshark DNS Query/Response 통계" style="max-width:48%">
</div>

> **윤리·법적 유의** hping3로 만드는 RST·SYN·UDP 위조 패킷은 실제 공격에도 쓰이는 기법입니다. 반드시 본인 소유의 격리된 실습실에서, 원리와 방어를 이해하기 위한 목적으로만 사용합니다.
{: .prompt-danger }

---

## 방어 — 불필요한 포트를 닫는다

방어의 첫걸음은 열린 포트를 확인하고, 필요 없는 서비스를 중지하며, 방화벽으로 필요한 포트만 허용하는 것입니다. UDP 기반 서비스에는 DTLS 같은 애플리케이션 계층 인증·암호화, 응답 크기 제한, 속도 제한을 함께 적용합니다.

```bash
ss -tulpen                             # 열린 포트 확인
sudo systemctl disable --now vsftpd    # 불필요 서비스 중지
sudo ufw default deny incoming && sudo ufw allow 22/tcp && sudo ufw allow 80/tcp && sudo ufw enable
```

> UDP 포트 스캔의 상세(핑·Idle·조각화 스캔)와 방어는 [05. 포트 스캐닝](/posts/networkset-05-scan-recon/)에서 다룹니다.
{: .prompt-tip }

---

## 기출 연결

| 기출 키워드 | 기억할 내용 |
|---|---|
| TCP | 연결 지향, 신뢰성, 3-way handshake |
| UDP | 비연결, 빠름, 8바이트 헤더, 신뢰성 낮음 |
| TCP 플래그 | SYN, ACK, FIN, RST, PSH, URG |
| Port / Socket | 16비트 포트 / IP+Port 조합 |
| ICMP Port Unreachable | 닫힌 UDP 포트 응답(Type 3/Code 3) |
| UDP 증폭 | 작은 요청, 큰 응답 → DDoS |

시험에서는 “UDP는 신뢰성 있는 전송을 제공한다”처럼 TCP와 UDP의 특징을 바꿔 묻는 경우가 많습니다.

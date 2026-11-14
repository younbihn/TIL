# 사설 CA로 서버 인증서 발급 후 내용 확인 (맥, ~/lab-tls)

## 측정 조건
- 일시: 2026-11-14 14:21 KST 무렵 확인. 인증서 발급은 그 직전(발급 시각 아래 참조)
- 위치: 맥의 `~/lab-tls` (저장소 밖. 개인키가 있어 저장소에 두지 않는다)
- 명령: `openssl x509 -in srv.crt -noout -subject -issuer -dates -ext subjectAltName,keyUsage,extendedKeyUsage`
- 인증서 발급 명령은 자료 17절의 ①~④ 순서(루트 CA 생성, 서버 키와 CSR 생성, 확장 설정 파일 `ext.cnf`, CA로 서명)를 따른 것으로 보이나 실행 출력은 받지 못했다. `openssl verify` 결과와 openssl 버전은 확인하지 못함(미확인)

## 결과
```
subject=O=Lab, CN=lab.example.test
issuer=O=Lab CA, CN=Lab Root CA
notBefore=Nov 14 05:20:27 2026 GMT
notAfter=Dec 14 05:20:27 2026 GMT
X509v3 Subject Alternative Name:
    DNS:lab.example.test, IP Address:192.168.64.6
X509v3 Key Usage:
    Digital Signature
X509v3 Extended Key Usage:
    TLS Web Server Authentication
```

## 명령 조각
| 조각 | 의미 |
|---|---|
| `openssl x509 -in srv.crt` | 인증서 파일을 읽는다 |
| `-noout` | 인증서 본체(`BEGIN CERTIFICATE ...`)를 다시 출력하지 않는다 |
| `-subject` `-issuer` | 인증서의 주체(누구의 것인가)와 발급자(누가 서명했나) |
| `-dates` | 유효 기간(`notBefore`, `notAfter`) |
| `-ext 확장이름,...` | 지정한 확장 항목만 출력: 이름 목록(SAN), 키 용도, 확장 키 용도 |

## 읽은 것
- **발급자와 주체가 다르다**: subject 는 `lab.example.test`, issuer 는 `Lab Root CA`. 서버 인증서를 별도의 CA 가 서명했다는 뜻이다(자체 서명이 아니다)
- **유효 기간 30일**: `notBefore` 05:20:27 GMT 는 한국 시각으로 14:20:27 이고 `notAfter` 는 12월 14일 14:20:27 KST. 만료일 확인은 운영에서 점검 항목으로 둘 만한 값이다
- **키 용도 `Digital Signature` 와 확장 키 용도 `TLS Web Server Authentication`**: 서버 인증용으로만 쓰도록 제한된 인증서다. CA 처럼 다른 인증서를 서명하는 용도(`keyCertSign`)는 없다
- **SAN 의 이름은 `lab.example.test`, IP 는 `192.168.64.6`**: 접속 이름이 이 목록에 있어야 이름 검증이 통과한다(CN 은 이름 검증에 쓰지 않는 추세)

## 확인할 점: SAN 의 IP 가 옛 주소다
- lb01 의 실제 주소는 `192.168.64.7` 인데 인증서에는 `192.168.64.6` 이 들어 있다. 자료 17절 명령이 옛 주소를 쓰고 있기 때문으로 보인다
- 이름(`lab.example.test`)으로 접속하면 DNS 항목이 맞아 영향이 없을 것이다. 하지만 `https://192.168.64.7/` 처럼 IP 로 접속하면 IP 항목이 맞지 않아 이름 검증이 실패할 것이다(예상, 이 환경에서 시험하지 않음)
- 이 인증서로 접속하는 시험은 아직 하지 않았다(미확인). 필요하면 SAN 의 IP 를 `.7` 로 바꿔 다시 서명한다(같은 CA 키와 CSR 을 재사용할 수 있다)

## 보안 메모
- `ca.key`, `srv.key` 는 개인키라 저장소에 올리지 않는다(`~/lab-tls` 는 저장소 밖). 서버 키는 `chmod 600` 으로 권한을 제한한다. 권한 확인은 하지 않음(미확인)
- 이 CA 는 시험용이라 실서비스 신뢰 저장소에 설치하지 않는다

## 미확인
- 발급 명령의 실제 출력, `openssl verify -CAfile ca.crt srv.crt` 결과, openssl 버전
- 개인키 파일 권한
- 이 인증서를 쓰는 HTTPS 서버의 접속 시험(`openssl s_client`, `netprobe.py --cacert`)
- SAN 의 IP 를 현재 주소로 바꿔 다시 발급했는지

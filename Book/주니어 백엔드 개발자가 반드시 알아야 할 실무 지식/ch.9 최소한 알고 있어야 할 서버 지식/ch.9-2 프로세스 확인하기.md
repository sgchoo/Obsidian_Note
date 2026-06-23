[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# Linux 프로세스 관리 정리

## 1. 프로세스 확인

### 정적 확인 (스냅샷)

```bash
# 모든 프로세스 확인 (BSD 스타일)
ps aux

# 모든 프로세스 확인 (System V 스타일)
ps -eaf

# 특정 프로세스 필터링
ps aux | grep java
ps -eaf | grep tomcat
```

|컬럼|설명|
|---|---|
|PID|프로세스 ID|
|%CPU|CPU 사용률|
|%MEM|메모리 사용률|
|STAT|프로세스 상태 (S=Sleep, R=Running, Z=Zombie)|
|COMMAND|실행 명령어|

### 동적 확인 (실시간)

```bash
top       # 기본 실시간 모니터링
htop      # 개선된 UI, 컬러 지원 (별도 설치 필요)
```

---

## 2. 프로세스 종료

```bash
kill [옵션] PID
```

|옵션|시그널|동작|사용 시기|
|---|---|---|---|
|`-15` / `-TERM` / `-s SIGTERM`|SIGTERM|**정상 종료** 요청, 프로세스가 자체 정리 후 종료|**기본값, 우선 시도**|
|`-9` / `-KILL` / `-s SIGKILL`|SIGKILL|**강제 종료**, OS가 즉시 제거|SIGTERM으로 안 될 때만|

> **⚠️ SIGKILL 주의점**  
> 프로세스가 열린 파일, DB 커넥션, 트랜잭션 롤백 등 정리 작업을 수행하지 못한다.  
> 데이터 손상이나 좀비 리소스가 남을 수 있으므로 최후 수단으로만 사용한다.

```bash
# 예시
kill -15 1234      # PID 1234에 정상 종료 요청
kill -9 1234       # PID 1234 강제 종료

# 프로세스명으로 종료 (PID 조회 불필요)
pkill -15 java
pkill -9 tomcat
```

---

## 3. 백그라운드 프로세스

### 포그라운드 vs 백그라운드

|구분|특징|예시|
|---|---|---|
|포그라운드|터미널 점유, 터미널 종료 시 같이 종료|`vi`, `top`, `tail -f`|
|백그라운드 (`&`)|터미널 비점유, 터미널 종료 시 같이 종료|`./app.sh &`|
|`nohup` + `&`|터미널 종료 후에도 계속 실행|`nohup ./app.sh &`|

```bash
# 백그라운드 실행
./server.sh &

# 터미널 종료 후에도 유지 (nohup)
nohup ./server.sh &

# 출력을 파일로 리다이렉트 (nohup.out 대신 지정)
nohup ./server.sh > /var/log/app.log 2>&1 &
#                                    ^^^^^
#                          stderr도 stdout으로 합쳐서 저장
```

### 실무 패턴

```bash
# 실행 후 PID 바로 확인
nohup java -jar app.jar > app.log 2>&1 &
echo $!   # 방금 실행한 프로세스의 PID 출력

# 나중에 PID로 종료
ps aux | grep app.jar
kill -15 <PID>
```

> **Spring Boot / Tomcat 운영 시**  
> `nohup` + `&` 조합이 가장 기본적인 방식이며, 실제 운영에서는 **systemd** 서비스로 등록하거나 **Docker** 컨테이너로 관리하는 것이 프로세스 재시작, 로그 관리 면에서 더 안정적이다.
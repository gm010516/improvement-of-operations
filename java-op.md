# 운영 history.md

## 목차


## <div id='1'/> 1. os commanding IDS 보안 이벤트 발생 
### <div id='1-1'/> 1.1. IDS 보안 이벤트 발생
```
◆ 등록시간
2025-05-10 05:28:24 
◆ 공격지IP
66.249.66.33

◆ 공격유형
OS Commanding

◆ 공격유형 설명
- 웹 애플리케이션의 취약점을 통해 운영체제 명령을 실행하는 공격입니다.
- 취약점을 통해 시스템 명령을 실행시켜 시스템 권한을 획득할 수 있고 이로 인해 2차 피해가 발생 할 수 있습니다.

◆ 요청사항
- 시스템 명령을 실행 할 수 있는 취약점을 제거합니다.
- 화이트리스트를 적용하여 필요한 문자만 실행할 수 있도록 제한합니다.
```

> 소스코드 레벨에서 발생한 OS Command 취약점 으로 확인

- 진행 사항
    1) 소스코드 전수조사
    2) xssfilter 수정

### <div id='1-2'/> 1.2. OS Commanding 취약점 조치 관련 참고 자료
http://yesxyz.kr/os-command-injection-vs-webshell-attack/

https://wiki.wikisecurity.net/guide:java_%EA%B0%9C%EB%B0%9C_%EB%B3%B4%EC%95%88_%EA%B0%80%EC%9D%B4%EB%93%9C#%EC%9A%B4%EC%98%81%EC%B2%B4%EC%A0%9C_%EB%AA%85%EB%A0%B9%EC%96%B4_%EC%82%BD%EC%9E%85_improper_neutralization_of_special_elements_used_in_an_os_command_os_command_injection

### <div id='1-3'/> 1.3. OS Commanding 취약점 조치 사항
- 로그 분석<br>
모든 app 로그 분석 (05.09 ~ 05.11) : 공격 로그 없음 <br>
Cluster 모든 vm 로그 분석 (05.09 ~ 05.11) : 공격 로그 없음 <br>(다른이슈 : 메모리 부족 로그 확인)
--> java heap outofmemory 에러<br>
80포트로 운영되고 있는 타 app (ex.jenkins) 이 이슈일 가능성도 있음<br>

- CSP 긴급 조치<br>
LoadBalancer : ACL 생성 및 공격자 IP 차단 (66.249.66.33)<br>
Server : ACG 설정 변경 (0.0.0.0/0 삭제)<br>
DB Server : ACG 설정 변경 (0.0.0.0/0 삭제 및 서버자원만 접근 가능)<br>

- 추가 작업<br>
	- user(front)에서 들어올수 있는 검색어 필터 강화
	- api(back)-interceptor 적용(기존 코드 interceptor 누락)
	- Request Param 필터 가능하도록 소스코드 생성

### <div id='1-3-1'/> 1.3.1. 취약점 조치 관련 소스 코드 수정
- ls -al 와 같은 명령어들에 대한 필터 처리 필요(interceptor 사용)
- 기존 필터 강화

> interceptor란 : https://gngsn.tistory.com/153

--> user에서 검색어 유입으로 들어오는 검색어 유입 긴급 조치 [완료]

- api로 바로 값을 넘기는 부분에 대해서도 조치 [완료]
  - api쪽에는 interceptor 설정 자체가 되어있지 않아서 해당 부분부터 조치함
  - post값으로 넘어오는 값에 대한 필터 기능 추가(기존 코드는 get만 검사) 
  
- xssfilter 코드 추가 및 변화로 인해 xssfilter 감지로 에러가 발생하였음을 알려주는 모달창 추가 작업 [완료]

### <div id='1-3-1'/> 1.3.2. 취약점 조치 테스트
- xss 테스트
	- api 단에 유입.
```json
{
    "error": "XSS 공격 시도로 의심되는 입력이 감지되었습니다.",
    "field": "query"
}
```

## <div id='2'/> 2. 교육 신청 반려 내용 작성시 오류 수정
### <div id='2-1'/> 2.1. 오류 분석 및 수정 사항
- 입력 값이 제한(256자)를 초과하여 생긴 오류
	- db수정
```sql
use `###`;

select * from `###`;
alter table `###` modify column reason_for_Participation varchar(1024);
```
## <div id='3'/> 3. 파일 다운로드 서비스 기능 개선 사례 (OutOfMemoryError 발생)
### <div id='3-1'/> 3.1. 오류 내용
- 포털 내부 파일 다운로드 및 S3 연동 서비스 개선
- 대용량 파일 다운로드시 Java OutOfMemoryError 발생
- 오류 발생시 pod는 다운되지않으나 정상 접속이 불가능하며 upstream 에러가 보여짐 

```log
2025-05-28 05:37:36,330 java-sdk-http-connection-reaper ERROR An exception occurred processing Appender STDOUT org.apache.logging.log4j.core.appender.AppenderLoggingException: java.lang.OutOfMemoryError: Java heap space
	at org.apache.logging.log4j.core.config.AppenderControl.tryCallAppender(AppenderControl.java:165)
	at org.apache.logging.log4j.core.config.AppenderControl.callAppender0(AppenderControl.java:134)
	at org.apache.logging.log4j.core.config.AppenderControl.callAppenderPreventRecursion(AppenderControl.java:125)
	at org.apache.logging.log4j.core.config.AppenderControl.callAppender(AppenderControl.java:89)
	at org.apache.logging.log4j.core.config.LoggerConfig.callAppenders(LoggerConfig.java:542)
	at org.apache.logging.log4j.core.config.LoggerConfig.processLogEvent(LoggerConfig.java:500)
	at org.apache.logging.log4j.core.config.LoggerConfig.log(LoggerConfig.java:483)
	at org.apache.logging.log4j.core.config.LoggerConfig.log(LoggerConfig.java:388)
	at org.apache.logging.log4j.core.config.AwaitCompletionReliabilityStrategy.log(AwaitCompletionReliabilityStrategy.java:63)
	at org.apache.logging.log4j.core.Logger.logMessage(Logger.java:153)
	at org.apache.logging.slf4j.Log4jLogger.log(Log4jLogger.java:376)
	at org.apache.commons.logging.impl.SLF4JLocationAwareLog.debug(SLF4JLocationAwareLog.java:151)
	at com.amazonaws.http.IdleConnectionReaper.run(IdleConnectionReaper.java:194)
Caused by: java.lang.OutOfMemoryError: Java heap space
```
- Pod를 재시작하여 일시적으로 해결하였으나, 발생주기가 짧아져 근본적인 코드 수정이 필요함
- 문제 대상 : S3파일 다운로드 및 기존 FTP/HTTP 다운로드 서비스

- 원인 분석 
  - 기존 코드 : 파일 전체를 메모리에 로딩한 다음 클라이언트에게 전송 -> 대용량 파일 처리시 힙 메모리 부족 발생
  
### <div id='3-2'/> 3.2. 개선/조치 내용
1. S3 파일 다운로드 코드 개선
	- 파일 전체를 메모리에 올리지 않고 InputStream을 통해 스트리밍 전송
	- 브라우저 호환을 고려한 파일명 인코딩 적용
2. 기존 TFP/HTTP 다운로드 코드 개선
	- InputStream을 통한 스트리밍 처리 적용
	- 파일명 브라우저 호환처리 개선
	- 8KB 단위 버퍼로 메모리 사용 최소화

### <div id='3-3'/> 3.3. 결과
- 대용량 파일 다운로드시 OutOfMemoryError 미발생
- Pod 재시작없이 안정적으로 서비스 운영 가능
- S3 및 FTP/HTTP파일 다운로드 모두 스트리밍 방식 적용으로 메모리 사용 최소화
- 사용자 파일 다운로드 개선 및 시스템 안정성 향상

- 해당 오류 발생일 : 2025/05/28
    - 기존 오류 발생주기 한달 -> 05/26, 05/27, 05/28 (지속적으로 발생)
    - 파일다운로드 소스코드 수정이후 발생 X



## <div id='4'/> 4.포털 장애 - redis 저장 실패로 인한 접속 불가
### <div id='3-1'/> 3.1. 오류 내용
- 포털 장애 발생 (2025.08.19 14:55 분경 확인)
- user log - 레디스(redis) 캐시 사용 중 에러 RedisCommandExecutionException
- redis 서버 접근 불가 (네이버 콘솔로 접근시 - failed to write entry ignoring read only file system)
```sh
2025-08-19 06:04:17.423 [DEBUG] [http-nio-8080-exec-65465] DispatcherServlet - Error rendering view [org.springframework.web.servlet.view.JstlView: name '/error/error404'; URL [/WEB-INF/views//error/error404.jsp]]
org.springframework.data.redis.RedisSystemException: Error in execution; nested exception is io.lettuce.core.RedisCommandExecutionException: MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commands that may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.
	at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:54) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:52) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:41) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.PassThroughExceptionTranslationStrategy.translate(PassThroughExceptionTranslationStrategy.java:44) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.FallbackExceptionTranslationStrategy.translate(FallbackExceptionTranslationStrategy.java:42) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.connection.lettuce.LettuceConnection.convertLettuceAccessException(LettuceConnection.java:277) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.connection.lettuce.LettuceConnection.await(LettuceConnection.java:1085) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.connection.lettuce.LettuceConnection.lambda$doInvoke$4(LettuceConnection.java:938) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.connection.lettuce.LettuceInvoker$Synchronizer.invoke(LettuceInvoker.java:665) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.connection.lettuce.LettuceInvoker.just(LettuceInvoker.java:109) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.connection.lettuce.LettuceHashCommands.hMSet(LettuceHashCommands.java:257) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.connection.DefaultedRedisConnection.hMSet(DefaultedRedisConnection.java:1435) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.core.DefaultHashOperations.lambda$putAll$12(DefaultHashOperations.java:213) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:224) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:191) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.core.AbstractOperations.execute(AbstractOperations.java:97) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.core.DefaultHashOperations.putAll(DefaultHashOperations.java:212) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.data.redis.core.DefaultBoundHashOperations.putAll(DefaultBoundHashOperations.java:187) ~[spring-data-redis-2.7.12.jar:2.7.12]
	at org.springframework.session.data.redis.RedisIndexedSessionRepository$RedisSession.saveDelta(RedisIndexedSessionRepository.java:811) ~[spring-session-data-redis-2.7.1.jar:2.7.1]
	at org.springframework.session.data.redis.RedisIndexedSessionRepository$RedisSession.save(RedisIndexedSessionRepository.java:799) ~[spring-session-data-redis-2.7.1.jar:2.7.1]
	at org.springframework.session.data.redis.RedisIndexedSessionRepository$RedisSession.access$000(RedisIndexedSessionRepository.java:686) ~[spring-session-data-redis-2.7.1.jar:2.7.1]
	at org.springframework.session.data.redis.RedisIndexedSessionRepository.save(RedisIndexedSessionRepository.java:415) ~[spring-session-data-redis-2.7.1.jar:2.7.1]
	at org.springframework.session.data.redis.RedisIndexedSessionRepository.save(RedisIndexedSessionRepository.java:251) ~[spring-session-data-redis-2.7.1.jar:2.7.1]
	at org.springframework.session.web.http.SessionRepositoryFilter$SessionRepositoryRequestWrapper.commitSession(SessionRepositoryFilter.java:226) ~[spring-session-core-2.7.1.jar:2.7.1]
	at org.springframework.session.web.http.SessionRepositoryFilter$SessionRepositoryRequestWrapper.access$100(SessionRepositoryFilter.java:193) ~[spring-session-core-2.7.1.jar:2.7.1]
	at org.springframework.session.web.http.SessionRepositoryFilter.doFilterInternal(SessionRepositoryFilter.java:145) ~[spring-session-core-2.7.1.jar:2.7.1]
	at org.springframework.session.web.http.SessionRepositoryFilter.doFilterNestedErrorDispatch(SessionRepositoryFilter.java:152) ~[spring-session-core-2.7.1.jar:2.7.1]
	at org.springframework.session.web.http.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:72) ~[spring-session-core-2.7.1.jar:2.7.1]
```


### <div id='3-2'/> 3.2. 발생 원인 파악
파일시스템이 읽기 전용으로 전환되어 Redis가 데이터를 기록하지 못한 상황
Redis는 디스크에 데이터를 저장하는 방식(영속성) 때문에, 디스크가 읽기 전용 상태가 되면 다음 문제가 발생하게됨
- EXT4는 디스크에 오류가 발생하면 안전을 위해 파일시스템을 자동으로 read-only로 마운트함
  - 원인 : 하드웨어 I/O 오류, 디스크 손상, 갑작스러운 전원 차단, Xen 하이퍼바이저 I/O 문제 등...
  - /dev/xvda1 : OS 및 Redis 데이터가 있는 파티션
    - Redis는 데이터를 디스크에 AOF(Append Only File) 또는 RDB 스냅샷 형태로 기록하는데, 읽기 전용 상태에서는 쓰기 불가 → 데이터 저장 실패 -> 결과적으로 클라이언트에서 RedisCommandExecutionException 발생
    - Redis는 메모리 기반이지만, persistence 옵션(appendonly yes, save ...)이 켜져 있는 경우 디스크 쓰기 실패 시 오류 발생
		- 일부 명령이 실패 → 포털사이트 캐시 연동 실패 → 서비스 장애
```sh
$ sudo journalctl -k -b -1 | grep -i "error"
Aug 19 15:39:12 kpaas-redis-storage kernel: RAS: Correctable Errors collector initialized.
Aug 19 15:39:12 kpaas-redis-storage kernel: EXT4-fs (xvda1): re-mounted. Opts: errors=remount-ro
Aug 19 15:39:17 kpaas-redis-storage kernel: EXT4-fs (xvdb1): mounted filesystem with ordered data mode. Opts: errors=remount-ro

$ sudo journalctl -k | grep -i "ext4"
Aug 19 15:45:34 kpaas-redis-storage kernel: EXT4-fs (xvda1): mounted filesystem with ordered data mode. Opts: (null)
Aug 19 15:45:34 kpaas-redis-storage kernel: EXT4-fs (xvda1): re-mounted. Opts: errors=remount-ro
Aug 19 15:45:35 kpaas-redis-storage kernel: EXT4-fs (xvdb1): mounted filesystem with ordered data mode. Opts: errors=remount-ro

$ sudo journalctl -k | grep -i "read-only"
Aug 19 15:45:34 kpaas-redis-storage kernel: Write protecting the kernel read-only data: 26624k

$ sudo journalctl -k | grep -i "i/o"
Aug 19 15:45:34 kpaas-redis-storage kernel: Xen Platform PCI: I/O protocol version 1
Aug 19 15:45:34 kpaas-redis-storage kernel: APIC: Switch to symmetric I/O mode setup
Aug 19 15:45:34 kpaas-redis-storage kernel: 00:06: ttyS0 at I/O 0x3f8 (irq = 4, base_baud = 115200) is a 16550A
```

### <div id='3-3'/> 3.3. 조치 내용
- 원격 접속 불가로 naver cloud 콘솔 창에서 접근
- GRUB모드 접근 (부팅 과정에성 shift / esc - > recovery mode 선택) 
> 참고 : https://mslee89.tistory.com/5
- clean(Try to make free space) 실행 -> redis 서버 정상 작동

clean역할
- clean(Try to make free space)
  - GRUB 모드에서 fsck 또는 clean 수행
    - 읽기 전용 상태로 마운트된 EXT4 파일시스템을 검사
	- 오류 블록/손상된 inode 등을 복구
    - 파일시스템 정상 상태로 재마운트 → Redis가 다시 디스크에 기록 가능

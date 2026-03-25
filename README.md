# OSORI Backend Portfolio

## 프로젝트 소개

OSORI는 사용자의 소비내역을 기록하고 관리할 수 있는 가계부 서비스입니다.

수입과 지출을 기록할 수 있으며, 매달 반복적으로 발생하는 고정지출을 관리할 수 있습니다.

또한 개인/그룹 챌린지 기능을 통해 사용자가 자신의 소비 습관을 점검하고 개선할 수 있도록 하였습니다. 

본 프로젝트는 팀 프로젝트로 진행되었으며

저는 Backend 영역에서 사용자계정 관리와 고정지출 기능 구현을 담당했습니다. 

특히 로그인 기능에서는 단순 인증처리에 그치지 않고 다음과 같은 기능을 함께 구현했습니다.

- 로그인 5회 실패 시 계정 잠금 처리
- 장기 미사용 계정 휴면 처리

이를 통해 사용자 계정 상태(정상, 휴면, 탈퇴)를 함께 고려하는 로그인 기능을 구현했습니다. 

### 프로젝트 링크 

- 팀 프로젝트 Repository
https://github.com/KH-FinalProject-OSoRi/OSoRi_Repository_Back

- 서비스
http://13.239.33.140/

## 담당 역할 

### 사용자 계정 및 인증 기능
- 회원 가입 및 로그인 기능구현
- 소셜 로그인 기능구현
- JWT 기반 인증 처리
- 로그인 실패횟수 기반 계정잠금 기능구현
- 휴면계정 처리 기능구현
- 회원 정보 수정

로그인 기능에서는 아이디와 비밀번호 확인뿐 아니라

- 로그인 실패 횟수
- 계정 잠금 상태
- 휴면 계정 상태
- 탈퇴 회원 여부 

까지 함께 확인하여 로그인 가능 여부를 판단하도록 구성했습니다. 
  

### 소비 관리 기능
- 고정지출 등록 / 수정 / 삭제 기능 구현
- 고정지출 데이터 조회 및 관리로직 구현
- MyBatis 기반 데이터 접근 로직 작성
- Oracle DB 연동 및 SQL 작성

월세, 통신비, 구독 서비스처럼 반복적으로 발생하는 지출을
사용자가 쉽게 관리할 수 있도록 고정지출 기능을 구현했습니다. 

## 기술 스택

### Backend
- Java 17
- Spring Boot
- MyBatis
- Oracle DB

### Authentication
- JWT
- BCrypt 

### Tools
- Git
- GitHub
- STS

## 시스템 아키텍처  

Client(React) -> Spring Boot Server -> Oracle Database 

- React : 사용자 인터페이스
- Spring Boot : 인증 및 비즈니스 로직 처리
- Oracle DB : 사용자 / 지출 데이터 저장
- JWT : 로그인 이후 사용자 인증 처리

로그인 이후에는 JWT 토큰을 이용하여 사용자를 식별하도록 구성했습니다. 

## 주요 기능 

### 로그인 및 사용자 인증

사용자의 아이디와 비밀번호를 검증하여 로그인 인증을 수행하고
인증 성공 시 JWT 토큰을 발급하도록 구현했습니다. 

- BCrypt 기반 비밀번호 암호화
- 로그인 성공 시 JWT 토큰 발급
- 로그인 성공 시 마지막 로그인 시점 기록
- 계정 상태(Y: 정상, H: 휴면, N: 탈퇴)에 따른 로그인 처리

비밀번호 확인 외에도 계정상태를 함께 확인하여 로그인 가능 여부를 판단하도록 구성했습니다.

안전한 인증을 위해 서명된 JWT 토큰을 발급하고, 만료 기간을 설정하여 보안을 강화했습니다.
```java
@Component 
public class JwtUtil {

	@Value("${jwt.secret:mySecretkeybackupTokenkey123}")
	private String secret; 
	
	//보안을 위해 토큰만료기간을 두어 탈취되었을때도 무한정 사용할 수 없도록 하기 위함
	@Value("${jwt.expiration:1800000}")
	private long expiration;

    //사용자 로그인시 jwt 토큰을 생성하는 메소드
	//매개변수 : loginId : 토큰에 포함할 사용자 식별자데이터
	//반환값 : 생성된 JWT 토큰 반환열
	public String generateToken(String loginId) {
		Date now = new Date(); // 현재 시간을 토큰 발급시간으로 지정하기
		Date expiryDate = new Date(now.getTime()+expiration); // 현재시간 + 만료시간
		
		//JWT 토큰 빌더를 이용해서 사용
		return Jwts.builder()
					.setSubject(loginId) 
					.setIssuedAt(now) 
					.setExpiration(expiryDate) 
					.signWith(getSignKey()) //signWith : 생성된 암호화키로 토크에 디지털 서명(위조 방지) 
					.compact(); // 최종적으로 JWT 문자열형태로 압축하여 반환한다. 
	}
```

### 로그인 처리 흐름

1. 로그인 요청
2. 사용자 조회 및 마지막 로그인 날짜 갱신, 휴면 여부 판단
3. 비밀번호 검증
4. 계정 잠금 여부 확인
5. 계정 상태(정상,휴면,탈퇴) 확인
6. 로그인 실패 시 실패 횟수 증가 및 5회 이상이면 잠금 처리
7. 로그인 성공 시 실패 횟수 초기화 및 JWT 토큰 발급 

### 로그인 실패 기반 계정 잠금

무차별 로그인 시도를 방지하기 위해 로그인 실패횟수 기반 계정잠금 정책을 구현했습니다.

- 로그인 실패 시 실패 횟수 증가
- 로그인 실패 5회 누적 시 계정 잠금
- 계정 잠금 시간 10분 설정
- 잠금 상태에서는 로그인 차단
- 10분이 지나면 로그인 가능 

반복적인 로그인 시도를 제한하여 사용자 계정을 보호할 수 있도록 했습니다. 

### 소셜 로그인

사용자의 로그인 편의를 위해 소셜 로그인 기능을 구현했습니다.

- OAuth 인증을 통해 사용자 정보 확인
- 최초 소셜 로그인 시 회원가입 진행
- 일반계정의 이메일과 소셜 이메일이 일치하지 않을 경우 계정 충돌 방지를 위해 연동 제한

소셜 로그인 연동을 해제하더라도
일반 로그인은 계속 사용할 수 있도록 구성했습니다.

### 휴면 계정 처리

장기간 로그인 하지 않은 계정을 구분하여 관리할 수 있도록 휴면계정 처리기능을 구현했습니다.

로그인 시점에 마지막 로그인 날짜를 함께 확인하여 휴면 여부를 판단하도록 구성했습니다.

처리 방식

- 현재 로그인 시점과 마지막 로그인 시점의 차이가 30일 이상이면 휴면 계정으로 변경
- 30일이 지나지 않은 경우 마지막 로그인 날짜를 현재 날짜로 갱신
- 처음 가입한 회원은 마지막 로그인 날짜가 없으므로 현재 날짜로 저장

이를 통해 로그인시 마지막 로그인 날짜 갱신과 휴면 여부 판단이 함께 이루어지도록 구현했습니다. 

### 고정 지출 관리 및 자동화 시스템

월세, 통신비, 구독 서비스처럼 매달 반복적으로 발생하는 지출을 효율적으로 관리하고 자동으로 가계부에 반영되도록 구현했습니다.

- 고정 지출 CRUD 구현 : 사용자가 반복 지출 내역을 한 번 등록하면 지속적으로 관리(조회/수정/삭제)할 수 있도록 기능을 구현했습니다.

- Spring Scheduler 기반 자동화 : 사용자가 등록한 고정 지출 내역이 매월 지정된 결제일에 개인 가계부 및 캘린더에 자동 등록되도록 @Scheduled를
활용한 배치 프로세스를 구축했습니다.

- MERGE INTO 쿼리를 통한 무결성 보장 : 스케줄러 동작 시 서버 재시작이나 로직 중복 실행 등의 예외 상황이 발생하더라도, 동일한 지출 내역이 중복으로 삽입되지 않도록 Oracle DB의 MERGE INTO 구문을 활용하여 데이터 무결성을 보장했습니다.

```java
@Component
@RequiredArgsConstructor
public class FixedTransSchedulerConfig {

  private final TransServiceImpl transService;

  @Scheduled(cron = "0 * * * * *") // 매분 0초마다 실행
  public void runDailyFixedTransToMyTrans() {
    transService.mergeFixedToMyTrans();
  }
}

```

## 트러블 슈팅

### 로그인 실패 횟수와 계정 잠금 처리

처음에는 로그인 실패가 5회 누적되면 계정을 잠그고, 잠금 시간(10분)이 지나면 다시 로그인할 수 있도록 구현했습니다.

하지만 테스트 과정에서 정상 로그인에 성공했음에도 이전 실패횟수가 그대로 남아있어 추후 다시 로그인 실패시 실패횟수 5회 누적으로 계정이 잠길 수 있다는 것을 확인하였습니다.

이 경우 로그인에 성공한다면 계정이 잠긴 상태가 아니므로 잠금 해제 시간을 고려할 필요는 없고, 로그인 실패 횟수만 0으로 초기화하면 됩니다.

따라서 로그인 처리 로직은 다음과 같이 정리했습니다.

해결 방안 : 비밀번호가 맞는 경우 계정잠금 여부를 확인한 뒤 로그인 실패 횟수를 초기화

```java
@Override
	public boolean compareLockUntil2(Timestamp lockUntil, String loginId) {
		
		// 1. 잠금 시간 자체가 없으면 바로 로그인 가능
	    if (lockUntil == null) {
	        int result = dao.resetLoginLock2(sqlSession, loginId);
	        
	        if(result > 0) { // LOGIN_COUNT를 0으로 리셋 했다면 
	        	return true; 
	        } 
	    	
	        return false;
	    	
	    }
```

```xml
<update id="resetLoginLock2" parameterType="String">
  		UPDATE USERS
  		SET LOGIN_COUNT = 0
  		WHERE LOGIN_ID = #{loginId}
  	</update>
```



이후 아래 상황을 기준으로 테스트를 진행했습니다.

- 정상 로그인 성공 후 로그인 실패 횟수가 초기화되는지 여부 

테스트 결과, 로그인 실패 횟수와 계정 잠금 시간이 의도한 대로 정상 처리되는 것을 확인했습니다.

![계정 잠금 처리](https://github.com/user-attachments/assets/db6a9964-52da-49b8-a19f-72e053ae0557)


## 휴면 계정 처리

휴면 계정 기능을 구현하면서 가장 까다로웠던 부분은
회원 상태 변경과 마지막 로그인 날짜를 함께 처리하는 일이었습니다.

로그인 시점마다 다음 상황을 모두 고려해야 했습니다.

- 현재 로그인 시점과 마지막 로그인 날짜 간격이 30일 이상인 경우
- 30일이 지나지 않은 경우
- 처음 가입했으므로 마지막 로그인 날짜가 기록되어 있지 않은 경우

특히 처음 가입한 회원은 LAST_LOGIN(최근에 마지막으로 로그인 한 시점)값이 없기 때문에

이 경우를 별도로 처리하지 않으면 휴면 여부를 정확하게 판단할 수 없었습니다.

이를 해결하기 위해 SQL에서 다음 문장을 추가했습니다.

WHEN LAST_LOGIN IS NULL THEN SYSDATE

또한 마지막 로그인 날짜 차이에 따라 휴면 여부와 날짜 갱신이 함께 이루어지도록 수정했습니다.

```xml
<update id="updateDate" parameterType="User">
  		UPDATE USERS
  		SET 
  			STATUS 	= 	CASE
  							WHEN SYSDATE - LAST_LOGIN &gt;= 30 THEN 'H'
  							ELSE STATUS
  					 	END,
  			LAST_LOGIN = CASE
  			    			WHEN SYSDATE - LAST_LOGIN &lt; 30 THEN SYSDATE
  			    			WHEN LAST_LOGIN IS NULL THEN SYSDATE
  			    			ELSE LAST_LOGIN
  			    		 END
  		WHERE LOGIN_ID = #{loginId}    		 			 	
  	</update>

```

그 결과 로그인 시 마지막 로그인 날짜 갱신과 휴면 여부 확인이 한 번에 처리되도록 개선했습니다. 

![휴면 계정 처리](https://github.com/user-attachments/assets/701e6aaa-236b-4410-ace1-44f8804b36a0)


## 일반 로그인과 소셜 로그인 처리

일반 로그인과 소셜 로그인을 함께 지원하면서 가장 까다로웠던 부분은
로그인 방식이 달라도 사용자 상태는 같은 기준으로 처리되어야 한다는 점이었습니다.

예를 들어, 일반 로그인에서 실패가 누적되어 계정이 잠긴 경우에는
소셜 로그인을 해도 로그인이 차단되어야 했고,
이미 탈퇴한 회원은 일반 로그인과 소셜 로그인 모두 사용할 수 없도록 처리해야 했습니다.

이 문제를 해결하기 위해 소셜 로그인 처리 과정에서도 기존 회원 정보를 다시 조회한 뒤

- 계정 잠금 여부 확인
- 탈퇴 여부 확인
- 기존 회원 정보와의 연결 여부 확인

을 함께 수행하도록 구성했습니다.

```java
      // 3. DB 가입 확인 및 처리 (이메일 기준)
	    User user = dao.findLoginIdByEmail(sqlSession, email); // 회원 조회
	    
	    Map<String, Object> result = new HashMap<>();
	    
	    if (user == null) { // 신규 회원이면 

	    		result.put("isNewMember", true); 
	    		result.put("email",email);
	    		result.put("nickName", nickName);
	    		result.put("providerUserId", providerUserId); 
	    		result.put("loginType","KAKAO");
	    		
	    		return result;
	    } else {
	    	
	    		if(user.getLockUntil() != null) { // 카카오로 로그인을 했어도 잠금 시간이 설정 되어 있는지 확인해야 한다.
	    			boolean canLogin = compareLockUntil(user.getLockUntil(), user.getLoginId());
	    			
	    			if(!canLogin) { // false 일 때만 메시지 띄우기 
	    				result.put("message", "잠금 모드가 해제 되는 시간은 " + user.getLockUntil() + "입니다.");
	    				return result; 
	    			}
	    		}
```

```java
int rowUpdate = dao.updateDate(sqlSession,user); // 업데이트 된 행이 있는지 판별
	    
	    if(rowUpdate > 0) { // 마지막 로그인 시점이 갱신 되었는지를 판별 

	    	
	    	user = dao.findLoginIdByEmail(sqlSession, email); // 업데이트 된 유저 객체 한번 더 호출
	    	
	    	// 4. 전용 JWT 발행
		    String token = jwtUtil.generateToken(user.getLoginId());
		    user.setPassword(null);
		    result.put("token", token);
		    result.put("user", user);
		    
	        if("H".equals(((User)result.get("user")).getStatus())) {
        			result.put("message", "휴면 회원 입니다. 프로필 설정 페이지에서 휴면 해제 후, 서비스 이용 가능합니다.");
	        } else if ("N".equals(((User)result.get("user")).getStatus())) {
	        		result.put("message", "탈퇴한 회원입니다."); 
	        }
		   
		    return result; // 날짜가 갱신이 되면 로그인 성공 처리 

	    	
	    }
```

그 결과 로그인 방식이 달라도 하나의 계정 상태를 기준으로 로그인 가능 여부를 판단할 수 있도록 정리했습니다. 

## 스케줄러 설정 오류와 데이터 중복 방어 

기능을 빠르게 테스트하기 위해 '1분 주기'로 설정해 두었던 스케줄러가 실제 운영 서버에 그대로 배포되면서 하루에 1440번이나
과다 실행되는 문제가 발생했습니다. 

스케줄러 제어에는 문제가 있었지만, 다행히 동일한 지출 내역이 중복으로 쌓이는 치명적인 데이터 오류는 발생하지 않았습니다. 
기능을 처음 설계할 때 스케줄러가 중복으로 실행될 수 있는 예외 상황을 미리 고려하여, Oracle DB의 MERGE INTO 구문으로 동일한 데이터는 무시하도록 방어 로직을 작성해 두었기 때문입니다. 

```xml
<insert id="mergeFixedToMyTrans">
  MERGE INTO MYTRANS m
  USING (
    SELECT FIXED_ID, USER_ID, NAME AS TITLE, TRUNC(SYSDATE) AS TRANS_DATE, AMOUNT, CATEGORY
    FROM FIXEDTRANS
    WHERE TO_NUMBER(TO_CHAR(TRUNC(SYSDATE), 'DD')) = LEAST(PAY_DAY, TO_NUMBER(TO_CHAR(LAST_DAY(TRUNC(SYSDATE)), 'DD')))
  ) s
  ON (
	/* 오늘 날짜로 등록된 동일한 고정 지출이 있다면 INSERT 무시 */ 
    m.FIXED_ID = s.FIXED_ID
    AND TRUNC(m.TRANS_DATE) = s.TRANS_DATE
    AND m.USER_ID = s.USER_ID
  )
  WHEN NOT MATCHED THEN
	/* 매칭되는 데이터가 없을 경우에만 새로운 고정 지출 내역 추가 */ 
    INSERT (TRAN_ID, TITLE, TRANS_DATE, ORIGINAL_AMOUNT, IS_SHARED, CATEGORY, TYPE, MEMO, USER_ID, FIXED_ID)
    VALUES (SEQ_MYTRANS.NEXTVAL, s.TITLE, s.TRANS_DATE, s.AMOUNT, 'N', s.CATEGORY, 'OUT', '고정지출 자동등록', s.USER_ID, s.FIXED_ID)
</insert>
```

예상치 못한 설정 오류에서도 데이터의 정확성을 안전하게 지켜낼 수 있었습니다.

## 프로젝트를 통해 배운 점

이번 프로젝트를 진행하며 단순히 눈에 보이는 기능을 완성하는 것을 넘어, 실제 서비스의 운영과 안정성을 미리 고민하는 백엔드 설계의 중요성을 깊이 체감했습니다. 특히 일반 로그인과 소셜 로그인처럼 인증 진입점이 다르더라도 계정 잠금, 휴면, 탈퇴와 같은 사용자 상태는 하나의 통일된 기준으로 제어해야 비즈니스 로직의 일관성과 보안을 유지할 수 있다는 점을 배웠습니다. 또한, 테스트용 스케줄러 설정이 운영 서버에 그대로 배포되어 과다 실행되었던 상황을 겪으며 로컬 환경과 운영 환경 설정의 철저한 분리와 배포 전 점검의 필요성을 뼈저리게 느꼈습니다. 다행히 사전에 작성해 둔 데이터베이스 단의 MERGE INTO 쿼리 덕분에 중복 데이터가 쌓이는 것을 막을 수 있었고, 이를 통해 애플리케이션 코드에 예상치 못한 문제가 생기더라도 DB 단에서 한 번 더 막아주는 설계가 서비스 장애를 예방하는 안전장치라는 것을 깨달았습니다. 

  

  


   







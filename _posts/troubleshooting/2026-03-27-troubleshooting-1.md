---
title: 대시보드 상태 체크 오류 장애 분석 — Interceptor 응답 처리 문제와 WireMock을 이용한 검증
description: 상태 체크 실패가 정상으로 표시되던 장애를 분석하면서 Interceptor 응답 처리 문제를 확인하고 WireMock으로 테스트 환경을 구성한 과정 정리
date: 2026-03-27T16:00:00+09:00
updated: 2026-03-27T16:00:00+09:00
categories: [Backend, TroubleShooting, Test]
tags: [Spring, Interceptor, WireMock, Mock, Gateway, Retrofit, Legacy]
---

<h2> 0. 글을 쓰게 된 이유 </h2>

운영 중이던 서비스에서 로그 상 `Exception`이 발생했다는 알림이 감지되었지만,
대시보드에는 정상 상태로 표시되는 현상이 발생했다.

원인을 확인해 보니 서버 응답 처리 로직의 오류로 인해
실패 응답이 정상 응답으로 저장되고 있었고,
그 결과 실제로는 장애가 발생했음에도 불구하고
대시보드에서는 정상으로 표시되고 있었다.

해당 문제를 수정하기 위해서는
외부 서버 응답을 포함한 전체 흐름을 다시 테스트해야 했지만,
운영 환경 구조상 외부 서버 응답을 직접 제어하며 테스트하기가 어려웠다.

동일한 상황을 시험 환경에서 재현하기 위해
`Mock` 서버를 구성하여 테스트를 진행하기로 했고,
이 과정에서 `WireMock을` 사용하게 되었다.

이번 글에서는 `WireMock`을 사용하게 된 배경과
실제로 적용하면서 정리한 내용들을 기록해 보려고 한다.


<h2> 1. 장애 발견 </h2>

운영 중이던 서비스에서 모니터링 시스템을 통해
특정 서비스의 상태 체크에 실패했다는 알람을 자정일 기준으로 약 10시간동안 3~5분 주기로 받기 시작했다.

로그를 확인해 보니 외부 서버와 통신하는 과정에서
`SSLHandshakeException`이 발생하고 있었고,
이로 인해 상태 체크 요청이 정상적으로 처리되지 않고 있었다.

해당 서비스는 외부 서버의 응답을 받아
대시보드에 상태를 표시하는 구조였기 때문에,
응답 처리 과정에 문제가 있을 가능성이 있다고 판단하여
관련 로직을 확인하게 되었다.


<h2> 2. 문제 상황 파악 </h2>

상태 체크 실패 알람이 발생하여
실제 서비스 운영에 영향이 있는지 먼저 확인했다.

하지만 예상과 달리 서비스 자체에는 문제가 없었고,
헬스체크 요청을 수행할 때만 오류가 발생하고 있었다.

로그 상으로는 외부 서버와 통신하는 과정에서
`SSL` 관련 오류가 발생하고 있었기 때문에
외부 → 내부 구간에서 인증서 문제나 통신 설정 문제가 있는 것이 아닌지 의심했다.

다만 해당 구간은 우리 쪽에서 직접 수정할 수 있는 부분이 아니었기 때문에
외부 측에 문의를 남겨두고 내부 로직을 다시 확인하기로 했다.

대시보드를 확인해 보니
서비스 상태 체크 결과는 체크 `API`를 통해 전달되어야 하는데,
실패가 발생한 경우에도 `Success`로 저장되고 있는 현상이 확인되었다.

응답을 받아 처리하는 과정에 문제가 있다고 판단했고,
요청 흐름을 따라가며

게이트웨이 서버 → 중앙 처리 서버 → 대시보드 서버

순서로 코드를 확인해 보기로 했다.


<h2> 3. 원인 분석 </h2>

첫 번째로는 게이트 웨이 서버를 확인했다.
게이트 웨이 서버는 spring boot, spring gateway, spring webflux 등의 기술로 구성되었고.

중앙 처리로 보낼때 어떤식으로 코드를 보내고 있는지 에러 로그를 기준으로 찾아봤고 분석중 아래와 같은 코드를 확인할 수 있었다.



<h3> 3.1 gateway </h3>

외부 서버 호출을 담당하는 게이트웨이 서버의 코드를 먼저 확인했다.

- Controller

```java
    @PostMapping(path = "...", produces = "application/json")
    public Mono<String> callRestExternal(@RequestBody Mono<String> request) {
        return request.map(body -> {
                        String result = externalService.requestExternal(json.getString("url"),
                        json.getJSONObject("headers")
                                .toString().trim().replace("\\",""),
                        json.getString("method"),
                        isEmptyData
                                ? null
                                : json.getJSONObject("payload").toString().trim().replace("\\","")
                );
                return result;
        });
    }
```

- ExternalService

```java
        try{
           ...
        } catch (IOException e) {
            log.error("Failed to read external response body - {}, {}", url, e.getClass().getSimpleName());
            return "{ \"code\": \"" + ReturnStatusCode.FAIL.getCode() + "\""
                    + ", \"message\": \"" + ReturnStatusCode.FAIL.getApiMessage() + "\" }";
        } finally {
            if (reader != null) {
                reader.close();
            }
        }
```

코드를 확인해 보니 외부 서버에 요청을 보내는 과정에서
`IOException`이 발생하면 해당 에러를 그대로 전달하지 않고
에러 정보를 `JSON` 형태로 만들어 반환하도록 되어 있었다.
`SSLHandshakeException`은 `IOException`을 상속받고 있어 해당 분기에서 처리가 되어 응답이 내려지게 된다.

즉,
외부 요청 실패
`SSLHandshakeException` 발생
게이트웨이에서 에러 로그 기록
실패 정보를 `JSON`으로 변환
`HTTP Status` 는 그대로 200 유지

와 같은 흐름으로 동작하고 있었다.

이 구조에서는 외부 요청이 실패하더라도
`HTTP` 응답 코드는 `200`으로 전달되기 때문에
응답을 받는 중앙 처리 서버에서는

`HttpStatus` 값이 아니라
`ResponseBody` 안에 들어 있는 `code`, `message` 값을 기준으로
외부 서버의 상태를 판단해야 하는 구조였다.


<h3> 3.2 중앙 처리 서버 </h3>

게이트웨이 서버에서 실패 정보를 명확하게 반환하고 있다는 점을 확인한 뒤,
이 응답을 받아 처리하는 중앙 처리 서버에서는 어떤 방식으로 상태를 판단하고 있는지 확인했다.

중앙 처리 서버는 레거시 `Spring 기반으로 구성되어 있었고,
외부 호출은 `Retrofit`을 사용하고 있었다.
또한 상태 체크 로직은 스케줄러를 통해 주기적으로 수행되고 있었다.

- Schedule

```java
    @Scheduled(fixedDelay = 30000)
    @Async("HealthCheckExecutor")
    void healthCheck() {
    	if ("true".equalsIgnoreCase(enableHealthCheck)) {
	        try {
	            healthCheckService.doHealthCheck();
	        } catch (Exception e) {
	            throw new RuntimeException(e);
	        }
    	}
    }
```

`doHealthCheck()` 내부에서는
상태 체크 대상 외부 서버 목록을 `DB`에서 조회한 뒤,
대상별로 게이트웨이 요청을 생성하여 순차적으로 처리하고 있었다.

- Service


>   요청 준비 부분

```java
import retrofit2.Call;
for(int i = 0; checkList.size(); i++){
    ExternalApiService service =
    	    requestBuilder.newDmzExternalRequest(dmzExternalBaseUrl, 
    	            new ExtJsonPostReqInterceptor(
    	            		check.getSysCd(), 
    	    				intService, 
    						tokenMgmtMapper,
    	            		healthCheckMapper),
    	            check.getDelayTime() * 2);
    Map<String,String> headers = new HashMap<>();
    headers.put("Content-Type", "application/json;encoding=utf-8");	            
    call = service.requestDmzExternal(headers, delegateBody.toString());
}
```

이 부분은 `checkList`를 순회하면서
게이트웨이로 요청을 보내기 위한 `Retrofit`의 `Call` 객체를 생성하는 과정이다.
실제 외부 요청은 이후 `call.execute()`가 실행되는 시점에 발생한다.


>   실제 요청 ~ 응답 부분


```java
    Response<String> response = null;
    try {
        response = call.execute();
        LocalDateTime resTime = LocalDateTime.now();	    			
        int useSeconds = (int)ChronoUnit.SECONDS.between(reqTime, resTime);
        boolean isDelayed = useSeconds >= checkList.get(i).getDelayTime();
        if (response.isSuccessful() && response.body() != null) {
            JSONObject resJson = new JSONObject(response.body());
                commonHealthCheckService.setHealthCheckResult(healthCheckLogDto, 
                                        String.valueOf(resJson.getString("code")), resTime, isDelayed);	                		
        } 
        else {
            throw new EmptyBodyResponseException("No Response Body");
        }
    } 

    public void setHealthCheckResult(HealthCheckLogDto logDto, String resultCode, 
    								 LocalDateTime resDt, boolean isDelayed) {
    	
        logDto.setResDt(Timestamp.valueOf(resDt));
        boolean isSuccess = "0000".equals(resultCode) || "200".equals(resultCode);
        HealthCheckStatusCode result;
        if (isSuccess && isDelayed) {
        	result = HealthCheckStatusCode.DELAYED;
        } else if (isSuccess && ! isDelayed) {
        	result = HealthCheckStatusCode.SUCCESS;
        } else {
        	result = HealthCheckStatusCode.ERROR;
        	logDto.setErrorReason("Code " + resultCode);
        }
        
        logDto.setStatus(result.name());
    }

```

코드만 놓고 보면 처리 방식 자체는 크게 이상해 보이지 않았다.

중앙 처리 서버는

- `call.execute()`로 게이트웨이에 요청을 보낸다.
- `response.isSuccessful()`과 `response.body()`를 확인한다.
- 응답 바디에서 `code` 값을 꺼낸다.
- `setHealthCheckResult()`에서 `code` 값을 기준으로 성공/실패를 판정한다.

즉, 게이트웨이에서 전달한 응답 바디 안의 `code` 값만 올바르게 들어온다면
정상적으로 `SUCCESS`, `DELAYED`, `ERROR` 중 하나로 처리되어야 하는 구조였다.

특히 `setHealthCheckResult()`를 보면
성공으로 판정되는 조건은 "0000" 또는 "200" 두 가지뿐이다.

그렇다면 900과 같은 실패 코드가 들어온 경우에는
정상적으로 `ERROR`로 저장되어야 한다.

하지만 실제 `DB`에는 실패 상황임에도 불구하고 `SUCCESS`로 저장되고 있었다.

즉, 문제는 단순히 게이트웨이 응답 형식에 있는 것이 아니라,
중앙 처리 서버 내부에서 응답 값을 해석하거나 전달하는 과정 어딘가에
추가적인 결함이 있을 가능성이 높다고 판단했다.

원인이 될 만한 지점을 다시 생각해 보니  
요청과 응답 흐름에 영향을 줄 수 있는 구간은 크게 `Filter` 또는 `Interceptor` 정도로 좁혀볼 수 있었다.

외부 서버 호출은 스프링 애플리케이션 내부에서 실행되고 있었기 때문에  
요청 전후에 동작하는 공통 처리 로직이 개입했을 가능성이 있다고 판단했다.

특히 응답 코드가 정상적으로 전달되지 않고  
다른 값으로 처리되고 있는 상황이었기 때문에  
응답을 가로채거나 변환할 수 있는 지점을 우선적으로 확인하기로 했다.

게이트 웨이로 보내는 요청이 스프링 범위 내에서 동작하는 구조를 고려했을 때  
`Filter`보다는 `Interceptor`에서 응답을 가공하고 있을 가능성이 더 높다고 판단했고,  
먼저 `Interceptor` 관련 코드를 중심으로 분석을 진행하기로 했다.

그러던중 `Call` 객체를 만들때 `Interceptor`를 설정해서 넣어주는 부분을 확인해서 해당 코드를 확인해봤다.


- ExtJsonPostReqInterceptor

`ExtJsonPostReqInterceptor` 코드를 확인해 보니  
문제를 일으킬 만한 지점이 바로 드러났다.

```java
public class ExtJsonPostReqInterceptor implements Interceptor {
	

	@Override
	public Response intercept(Chain chain) throws IOException {
        
        Response response = chain.proceed(originalRequest
		.newBuilder()
		.url(originUrl)
		.headers(originHeaders.newBuilder()
				.set("Content-Length", String.valueOf(rawBody.length))
				.set("Content-Type", "application/json")
				.set("Cache-Control", "no-cache")
				.build())
		.method(originMethod, RequestBody.create(originMediaType, rawBody))
		.build());
		
		Headers firstHeaders = response.headers();
		ResponseBody firstBody = response.body();
		byte[] firstBodyBytes = firstBody.bytes();
		int firstResHttpCd = response.code();

        JSONObject delegateMsgJson = new JSONObject();
        switch (firstResHttpCd) {
            case 200: {
                delegateMsgJson.put("code", ReturnStatusCode.SUCCESS.getCode());
                delegateMsgJson.put("message", ReturnStatusCode.SUCCESS.getApiMessage());
            } break;
                    
            case 404: {
                delegateMsgJson.put("code", ReturnStatusCode.NO_RESPONSE.getCode());
                delegateMsgJson.put("message", ReturnStatusCode.NO_RESPONSE.getApiMessage());
            } break;
                    
            default : {
                delegateMsgJson.put("code", ReturnStatusCode.FAIL.getCode());
                delegateMsgJson.put("message", ReturnStatusCode.FAIL.getApiMessage());
            }
        }
        final String delegateMsg = delegateMsgJson.toString();
		final byte[] delegateRaw = delegateMsg.getBytes();
			
		return response.newBuilder()
				.body(ResponseBody.create(originMediaType, delegateRaw))
				.headers(response.headers().newBuilder()
						.set("Content-Length", String.valueOf(rawBody.length))
						.build())
				.code(HttpStatus.OK.value())
				.build();
    }
}
```

많은 코드가 생략되어 있지만
문제의 핵심을 이해하기에는 충분하다 생각한다.

이 코드의 가장 큰 문제는
실제 응답 바디 안에 들어 있는 업무상 실패 여부를 보지 않고,
오직 `response.code()` 값만 기준으로 다시 결과 코드를 생성하고 있다는 점이다.

즉, 분기 기준은 아래 한 줄이다.

```
int firstResHttpCd = response.code();
```

`response`에 담긴 객체는 `.code()`를 호출하면 `HttpStatus`값을 반환한다.


```java
public final class Response implements Closeable {
  final Request request;
  final Protocol protocol;
  final int code;
  final String message;
  final @Nullable Handshake handshake;
  final Headers headers;
  final @Nullable ResponseBody body;
  final @Nullable Response networkResponse;
  final @Nullable Response cacheResponse;
  final @Nullable Response priorResponse;
  final long sentRequestAtMillis;
  final long receivedResponseAtMillis;
  final @Nullable Exchange exchange;
  
  /** Returns the HTTP status code. */
  public int code() {
    return code;
  }

}
```

결론적으로 문제의 원인은  
외부 서버의 실제 응답 상태를 그대로 전달받는 구조가 아니라,  
게이트웨이 서버의 HTTP 응답 상태값을 기준으로 결과를 다시 생성하고  
최종 `response` 값을 덮어쓰고 있었기 때문이었다.

이 구조에서는 외부 서버에서 실패가 발생하더라도  
게이트웨이까지의 통신이 정상적으로 이루어지면  
항상 `200 OK` 기준으로 처리될 수 있고,  
그 결과 중앙 처리 서버에서는 실패 상황임에도 불구하고  
정상 응답으로 판단하게 되는 문제가 발생하고 있었다.


<h2> 4. 해결 방안 검토 </h2>

원인을 확인하고 나니 해결 방법 자체는 비교적 단순했다.

기존 코드에서는 `HttpStatus` 값만 기준으로 분기를 하고 있었기 때문에,  
외부 서버에서 전달된 실제 응답 코드가 제대로 반영되지 않고 있었다.

따라서 HTTP 상태 코드 분기 이후에  
응답 바디 안에 들어 있는 `code` 값을 기준으로  
한 번 더 분기하도록 로직을 추가하면  
정상적으로 성공 / 실패를 구분할 수 있을 것이라 판단했다.

문제는 수정 자체보다  
이 변경 사항을 어떻게 검증할 것인가였다.

실제 장애 상황은 외부 서버와의 통신 과정에서 발생했기 때문에  
같은 상황을 다시 만들기 위해서는  
외부 서버의 응답을 직접 제어할 수 있는 환경이 필요했다.

이때 이전에 조사해 두었던 `WireMock`을 이용하면  
게이트웨이 응답을 원하는 형태로 모킹할 수 있고,  
`Interceptor` 내부에서 분기가 정상적으로 동작하는지  
시험 환경에서 재현하여 확인할 수 있을 것이라 판단했다.


<h2> 5. 적용 계획 수립 </h2>

수정한 로직을 검증하기 위해 테스트 환경을 구성해야 했지만,  
프로젝트 구조상 애플리케이션 내부에 테스트 코드를 내재화하여 실행하기는 어려운 상황이었다.

따라서 `WireMock`을 애플리케이션 내부에 포함시키는 방식 대신  
`standalone` 방식으로 `Mock` 서버를 따로 띄워  
외부 서버를 대신하도록 구성하는 방법을 선택했다.

테스트를 진행하기 위해서는  
상태 체크 대상 서버 목록을 DB에서 조회하는 구조를 고려해야 했다.

중앙 처리 서버는 DB에 등록된 외부 서버 목록을 기준으로  
게이트웨이를 통해 요청을 보내고 있었기 때문에,  
테스트 환경에서는 `WireMock` 서버의 URL을 외부 서버로 인식하도록  
DB에 테스트용 데이터를 추가하는 방식으로 구성했다.

즉,

- `WireMock` 서버 실행
- `WireMock`에 테스트용 응답 정의
- `DB`에 `WireMock URL` 추가
- 서버 재배포
- 스케줄러 실행
- 대시보드 상태 확인

순서로 테스트를 진행하기로 했다.


<h2> 6. 느낀 점 </h2>

사실 장애 내용 자체만 놓고 보면  
기술적으로 매우 어려운 유형의 장애는 아니었다고 생각한다.

이전에 스프링 기반 프로젝트를 오래 다뤄왔고,  
현재 투입된 시스템의 전체 아키텍처를 어느 정도 이해하고 있는 상태였기 때문에  
요청 흐름을 하나씩 따라가며 원인을 찾는 과정 자체는 크게 어렵지 않았다.

아마 테스트 환경이 자유로운 프로젝트였다면  
비슷한 유형의 문제는 1시간 내외로 원인을 찾고 수정할 수 있었을 것 같다.

하지만 이번 프로젝트는 구성 요소마다 사용하는 기술이 서로 달랐고  
환경 자체가 장애 분석을 어렵게 만드는 구조였다.

- 일부는 `Spring` 레거시
- 일부는 `Spring Boot`
- `HTTP Client`도 `Retrofit` / `WebFlux` 혼용
- 테스트 코드 없는 구조
- 컨트롤러 / 서비스에 로직 집중
- 폐쇄망 환경

특히 테스트 코드를 고려하지 않고 작성된 구조가 많아서  
문제를 재현하고 검증하는 과정이 생각보다 오래 걸렸다.

폐쇄망 프로젝트라면  
이런 상황에서도 기능 검증이나 장애 재현을 할 수 있도록  
`Mock` 서버나 테스트 환경을 쉽게 구성할 수 있는 구조가 필요하다고 느꼈다.

이번 작업에서는 기능 단위로 `WireMock`을 이용해 테스트를 진행할 수 있었지만,  
여러 서비스가 동시에 호출되거나  
게이트웨이를 거쳐 복잡한 요청이 오가는 구조에서는  
어떤 방식으로 테스트 환경을 구성해야 할지에 대한 고민도 남았다.

또 하나 느낀 점은  
기존 코드 구조가 필요 이상으로 복잡하게 작성되어 있는 부분이 많았다는 것이다.

컨트롤러에는 많은 로직이 들어가 있는데  
서비스는 지나치게 얇고,  
실제 동작은 여러 컴포넌트로 나뉘어 있어  
전체 흐름을 이해하는 데 시간이 많이 소요되었다.

이번 장애를 겪으면서  
기능 구현뿐 아니라

- 테스트 가능한 구조
- 일관된 기술 스택
- 명확한 책임 분리
- 재현 가능한 환경

이 얼마나 중요한지 다시 한번 느끼게 되었다.

- wiremock ? : [https://angrypig123.github.io/posts/wiremock/](https://angrypig123.github.io/posts/wiremock/)
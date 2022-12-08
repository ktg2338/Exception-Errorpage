# Exception-Errorpage
예외처리를 서블릿의 필터와 스프링의 인터셉터를 사용하여 구현해보고 부트를 이용하여 오류페이지를 간편하게 만들어본다.

서블릿은 다음 2가지 방식으로 예외 처리를 지원한다. <br/>
Exception (예외)<br/>
response.sendError(HTTP 상태 코드, 오류 메시지)<br/>

Exception(예외)<br/>
자바 직접 실행<br/>
자바의 메인 메서드를 직접 실행하는 경우 main 이라는 이름의 쓰레드가 실행된다.<br/>
실행 도중에 예외를 잡지 못하고 처음 실행한 main() 메서드를 넘어서 예외가 던져지면, 예외 정보를
남기고 해당 쓰레드는 종료된다.<br/>
웹 애플리케이션<br/>
웹 애플리케이션은 사용자 요청별로 별도의 쓰레드가 할당되고, 서블릿 컨테이너 안에서 실행된다.<br/>
애플리케이션에서 예외가 발생했는데, 어디선가 try ~ catch로 예외를 잡아서 처리하면 아무런 문제가
없다. 그런데 만약에 애플리케이션에서 예외를 잡지 못하고, 서블릿 밖으로 까지 예외가 전달되면 어떻게
동작할까?<br/>
<br/>
WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)<br/>
결국 톰캣 같은 WAS 까지 예외가 전달된다<br/>
<br/>
 tomcat이 기본으로 제공하는 오류 화면을 볼 수 있다.<br/>
HTTP Status 500 – Internal Server Error<br/>
웹 브라우저에서 개발자 모드로 확인해보면 HTTP 상태 코드가 500으로 보인다.<br/>
Exception 의 경우 서버 내부에서 처리할 수 없는 오류가 발생한 것으로 생각해서 HTTP 상태 코드 500을
반환한다.<br/>
이번에는 아무사이트나 호출해보자.<br/>
http://localhost:8080/no-page<br/>
HTTP Status 404 – Not Found<br/>
톰캣이 기본으로 제공하는 404 오류 화면을 볼 수 있다.<br/>
<br/>
response.sendError(HTTP 상태 코드, 오류 메시지)<br/>
오류가 발생했을 때 HttpServletResponse 가 제공하는 sendError 라는 메서드를 사용해도 된다. <br/>
이것을 호출한다고 당장 예외가 발생하는 것은 아니지만, 서블릿 컨테이너에게 오류가 발생했다는 점을
전달할 수 있다.<br/>
이 메서드를 사용하면 HTTP 상태 코드와 오류 메시지도 추가할 수 있다.<br/>
response.sendError(HTTP 상태 코드, 오류 메시지)<br/>
<br/>
sendError 흐름<br/>
WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러<br/>
(response.sendError())<br/>
response.sendError() 를 호출하면 response 내부에는 오류가 발생했다는 상태를 저장해둔다.<br/>
그리고 서블릿 컨테이너는 고객에게 응답 전에 response 에 sendError() 가 호출되었는지 확인한다. <br/>
그리고 호출되었다면 설정한 오류 코드에 맞추어 기본 오류 페이지를 보여준다.<br/>
<br/>

서블릿 예외 처리 - 오류 화면 제공<br/>
서블릿 컨테이너가 제공하는 기본 예외 처리 화면은 고객 친화적이지 않다. 서블릿이 제공하는 오류 화면<br/>
기능을 사용해보자<br/>
![image](https://user-images.githubusercontent.com/69129562/206109504-7fe9d4eb-d894-4835-9c18-fad01db0532b.png)<br/>
response.sendError(404) : errorPage404 호출<br/>
response.sendError(500) : errorPage500 호출<br/>
RuntimeException 또는 그 자식 타입의 예외: errorPageEx 호출<br/>
500 예외가 서버 내부에서 발생한 오류라는 뜻을 포함하고 있기 때문에 여기서는 예외가 발생한 경우도
500 오류 화면으로 처리했다.<br/>
오류 페이지는 예외를 다룰 때 해당 예외와 그 자식 타입의 오류를 함께 처리한다. 예를 들어서 위의 경우
RuntimeException 은 물론이고 RuntimeException 의 자식도 함께 처리한다.<br/>
오류가 발생했을 때 처리할 수 있는 컨트롤러가 필요하다. 예를 들어서 RuntimeException 예외가
발생하면 errorPageEx 에서 지정한 /error-page/500 이 호출된다.<br/>
<br/>
서블릿은 Exception (예외)가 발생해서 서블릿 밖으로 전달되거나 또는 response.sendError() 가 호출
되었을 때 설정된 오류 페이지를 찾는다<br/>
WAS는 해당 예외를 처리하는 오류 페이지 정보를 확인한다.<br/>
new ErrorPage(RuntimeException.class, "/error-page/500")<br/>
예를 들어서 RuntimeException 예외가 WAS까지 전달되면, WAS는 오류 페이지 정보를 확인한다. <br/>
확인해보니 RuntimeException 의 오류 페이지로 /error-page/500 이 지정되어 있다. WAS는 오류
페이지를 출력하기 위해 /error-page/500 를 다시 요청한다.<br/>
<br/>
예외 발생과 오류 페이지 요청 흐름<br/>
1. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)<br/>
2. WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/errorpage/500) -> View<br/>
<br/>
중요한 점은 웹 브라우저(클라이언트)는 서버 내부에서 이런 일이 일어나는지 전혀 모른다는 점이다. 오직
서버 내부에서 오류 페이지를 찾기 위해 추가적인 호출을 한다<br/>
<br/>
WAS는 오류 페이지를 단순히 다시 요청만 하는 것이 아니라, 오류 정보를 request 의 attribute 에
추가해서 넘겨준다.<br/>
필요하면 오류 페이지에서 이렇게 전달된 오류 정보를 사용할 수 있다.<br/>
<br/>
오류가 발생하면 오류 페이지를 출력하기 위해 WAS 내부에서 다시 한번 호출이 발생한다. 이때 필터, 
서블릿, 인터셉터도 모두 다시 호출된다. 그런데 로그인 인증 체크 같은 경우를 생각해보면, 이미 한번
필터나, 인터셉터에서 로그인 체크를 완료했다. 따라서 서버 내부에서 오류 페이지를 호출한다고 해서 해당
필터나 인터셉트가 한번 더 호출되는 것은 매우 비효율적이다.<br/>
결국 클라이언트로 부터 발생한 정상 요청인지, 아니면 오류 페이지를 출력하기 위한 내부 요청인지 구분할
수 있어야 한다. 서블릿은 이런 문제를 해결하기 위해 DispatcherType 이라는 추가 정보를 제공한다.<br/>

![image](https://user-images.githubusercontent.com/69129562/206113198-8717e450-5109-4b9b-9e0e-607e53b6f3c9.png)
<br/>
DispatcherType<br/>
REQUEST : 클라이언트 요청<br/>
ERROR : 오류 요청<br/>
FORWARD : MVC에서 배웠던 서블릿에서 다른 서블릿이나 JSP를 호출할 때<br/>
RequestDispatcher.forward(request, response);<br/>
INCLUDE : 서블릿에서 다른 서블릿이나 JSP의 결과를 포함할 때<br/>
RequestDispatcher.include(request, response);<br/>
ASYNC : 서블릿 비동기 호출<br/>
<br/>
/error-ex 오류 요청<br/>
필터는 DispatchType 으로 중복 호출 제거 ( dispatchType=REQUEST )<br/>
인터셉터는 경로 정보로 중복 호출 제거( excludePathPatterns("/error-page/**") )<br/>
1. WAS(/error-ex, dispatchType=REQUEST) -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러<br/>
2. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)<br/>
3. WAS 오류 페이지 확인<br/>
4. WAS(/error-page/500, dispatchType=ERROR) -> 필터(x) -> 서블릿 -> 인터셉터(x) -> 
컨트롤러(/error-page/500) -> View<br/>
<br/>
지금까지 예외 처리 페이지를 만들기 위해서 다음과 같은 복잡한 과정을 거쳤다.<br/>
WebServerCustomizer 를 만들고<br/>
예외 종류에 따라서 ErrorPage 를 추가하고<br/>
예외 처리용 컨트롤러 ErrorPageController 를 만듬<br/>
<br/>
스프링 부트는 이런 과정을 모두 기본으로 제공한다.<br/>
ErrorPage 를 자동으로 등록한다. 이때 /error 라는 경로로 기본 오류 페이지를 설정한다.<br/>
new ErrorPage("/error") , 상태코드와 예외를 설정하지 않으면 기본 오류 페이지로 사용된다.<br/>
서블릿 밖으로 예외가 발생하거나, response.sendError(...) 가 호출되면 모든 오류는 /error 를
호출하게 된다. <br/>
BasicErrorController 라는 스프링 컨트롤러를 자동으로 등록한다.<br/>
ErrorPage 에서 등록한 /error 를 매핑해서 처리하는 컨트롤러다<br/>
<br/>
개발자는 오류 페이지만 등록<br/>
BasicErrorController 는 기본적인 로직이 모두 개발되어 있다.<br/>
개발자는 오류 페이지 화면만 BasicErrorController 가 제공하는 룰과 우선순위에 따라서 등록하면
된다. 정적 HTML이면 정적 리소스, 뷰 템플릿을 사용해서 동적으로 오류 화면을 만들고 싶으면 뷰 템플릿
경로에 오류 페이지 파일을 만들어서 넣어두기만 하면 된다.<br/>
<br/>
뷰 선택 우선순위<br/>
BasicErrorController 의 처리 순서<br/>
1. 뷰 템플릿<br/>
resources/templates/error/500.html<br/>
resources/templates/error/5xx.html<br/>
2. 정적 리소스( static , public )<br/>
resources/static/error/400.html<br/>
resources/static/error/404.html<br/>
resources/static/error/4xx.html<br/>
3. 적용 대상이 없을 때 뷰 이름( error )<br/>
resources/templates/error.html<br/>
해당 경로 위치에 HTTP 상태 코드 이름의 뷰 파일을 넣어두면 된다.<br/>
##필터와 인터셉터에 예외처리 (코드 참고)<br/>
<br/>
API 예외 처리는 어떻게 해야할까?<br/>
HTML 페이지의 경우 지금까지 설명했던 것 처럼 4xx, 5xx와 같은 오류 페이지만 있으면 대부분의 문제를
해결할 수 있다.<br/>
그런데 API의 경우에는 생각할 내용이 더 많다. 오류 페이지는 단순히 고객에게 오류 화면을 보여주고
끝이지만, API는 각 오류 상황에 맞는 오류 응답 스펙을 정하고, JSON으로 데이터를 내려주어야 한다.<br/>

![image](https://user-images.githubusercontent.com/69129562/206125967-38866e43-f554-4ee4-b44b-39d573599924.png)<br/>
produces = MediaType.APPLICATION_JSON_VALUE 의 뜻은 클라이언트가 요청하는 HTTP Header의
Accept 의 값이 application/json 일 때 해당 메서드가 호출된다는 것이다.<br/> 결국 클라어인트가 받고
싶은 미디어타입이 json이면 이 컨트롤러의 메서드가 호출된다.<br/>
응답 데이터를 위해서 Map 을 만들고 status , message 키에 값을 할당했다. Jackson 라이브러리는
Map 을 JSON 구조로 변환할 수 있다.<br/>
ResponseEntity 를 사용해서 응답하기 때문에 메시지 컨버터가 동작하면서 클라이언트에 JSON이
반환된다<br/>
<br/>
{<br/>
 "message": "잘못된 사용자",<br/>
 "status": 500<br/>
}<br/>
에러를 발생시키면 이렇게 api의 스펙이 뜬다.<br/>
<br/>
API 예외 처리도 스프링 부트가 제공하는 기본 오류 방식을 사용할 수 있다.<br/>
스프링 부트가 제공하는 BasicErrorController 코드를 보자.<br/>
![image](https://user-images.githubusercontent.com/69129562/206130264-05d1ea8c-6218-4db9-9f59-b79999c441ad.png)<br/>
/error 동일한 경로를 처리하는 errorHtml() , error() 두 메서드를 확인할 수 있다.<br/>
errorHtml() : produces = MediaType.TEXT_HTML_VALUE : 클라이언트 요청의 Accept 해더 값이
text/html 인 경우에는 errorHtml() 을 호출해서 view를 제공한다.<br/>
error() : 그외 경우에 호출되고 ResponseEntity 로 HTTP Body에 JSON 데이터를 반환한다.<br/>
스프링 부트의 예외 처리<br/>
앞서 학습했듯이 스프링 부트의 기본 설정은 오류 발생시 /error 를 오류 페이지로 요청한다.<br/>
BasicErrorController 는 이 경로를 기본으로 받는다. ( server.error.path 로 수정 가능, 기본 경로 /
error )<br/>
<br/>
예외가 발생해서 서블릿을 넘어 WAS까지 예외가 전달되면 HTTP 상태코드가 500으로 처리된다.<br/>
예를 들어서 IllegalArgumentException 을 처리하지 못해서 컨트롤러 밖으로 넘어가는 일이 발생하면
HTTP 상태코드를 400으로 처리하고 싶다. 어떻게 해야할까<br/>
HandlerExceptionResolver<br/>
스프링 MVC는 컨트롤러(핸들러) 밖으로 예외가 던져진 경우 예외를 해결하고, 동작을 새로 정의할 수 있는
방법을 제공한다. 컨트롤러 밖으로 던져진 예외를 해결하고, 동작 방식을 변경하고 싶으면
HandlerExceptionResolver 를 사용하면 된다. 줄여서 ExceptionResolver 라 한다<br/>
![image](https://user-images.githubusercontent.com/69129562/206132843-1dc80060-2375-4407-8563-7d726d569e40.png)<br/>
![image](https://user-images.githubusercontent.com/69129562/206132918-f6db9b2c-76bc-45d6-8e1f-3d5dbcaee165.png)<br/>
참고: ExceptionResolver 로 예외를 해결해도 postHandle() 은 호출되지 않는다<br/>
<br/>
![image](https://user-images.githubusercontent.com/69129562/206135122-5503d252-5b2f-407a-ab82-167f2bf920ee.png)<br/>
ExceptionResolver 가 ModelAndView 를 반환하는 이유는 마치 try, catch를 하듯이, Exception 을
처리해서 정상 흐름 처럼 변경하는 것이 목적이다. 이름 그대로 Exception 을 Resolver(해결)하는 것이
목적이다.<br/>
여기서는 IllegalArgumentException 이 발생하면 response.sendError(400) 를 호출해서 HTTP 
상태 코드를 400으로 지정하고, 빈 ModelAndView 를 반환한다.<br/>
반환 값에 따른 동작 방식<br/>
HandlerExceptionResolver 의 반환 값에 따른 DispatcherServlet 의 동작 방식은 다음과 같다.<br/>
빈 ModelAndView: new ModelAndView() 처럼 빈 ModelAndView 를 반환하면 뷰를 렌더링 하지
않고, 정상 흐름으로 서블릿이 리턴된다.<br/>
ModelAndView 지정: ModelAndView 에 View , Model 등의 정보를 지정해서 반환하면 뷰를 렌더링
한다.<br/>
null: null 을 반환하면, 다음 ExceptionResolver 를 찾아서 실행한다. 만약 처리할 수 있는
ExceptionResolver 가 없으면 예외 처리가 안되고, 기존에 발생한 예외를 서블릿 밖으로 던진다.<br/>
ExceptionResolver 활용<br/>
예외 상태 코드 변환<br/>
예외를 response.sendError(xxx) 호출로 변경해서 서블릿에서 상태 코드에 따른 오류를
처리하도록 위임<br/>
이후 WAS는 서블릿 오류 페이지를 찾아서 내부 호출, 예를 들어서 스프링 부트가 기본으로 설정한 /
error 가 호출됨<br/>
뷰 템플릿 처리<br/>
ModelAndView 에 값을 채워서 예외에 따른 새로운 오류 화면 뷰 렌더링 해서 고객에게 제공<br/>
API 응답 처리<br/>
response.getWriter().println("hello"); 처럼 HTTP 응답 바디에 직접 데이터를 넣어주는
것도 가능하다. 여기에 JSON 으로 응답하면 API 응답 처리를 할 수 있다.<br/>
<br/>
WebMvcConfigurer 를 통해 등록<br/>
@Override<br/>
public void extendHandlerExceptionResolvers(List<HandlerExceptionResolver><br/>
resolvers) {<br/>
 resolvers.add(new MyHandlerExceptionResolver());<br/>
}<br/>
<br/>
예외가 발생하면 WAS까지 예외가 던져지고, WAS에서 오류 페이지 정보를 찾아서 다시 /error 를
호출하는 과정은 생각해보면 너무 복잡하다. ExceptionResolver 를 활용하면 예외가 발생했을 때 이런
복잡한 과정 없이 여기에서 문제를 깔끔하게 해결할 수 있다<br/>
![image](https://user-images.githubusercontent.com/69129562/206137935-6730666b-b17c-4a6f-a2b8-e00ed9cc81c5.png)<br/>
ExceptionResolver 를 사용하면 컨트롤러에서 예외가 발생해도 ExceptionResolver 에서 예외를
처리해버린다.<br/>
따라서 예외가 발생해도 서블릿 컨테이너까지 예외가 전달되지 않고, 스프링 MVC에서 예외 처리는 끝이
난다.<br/>
결과적으로 WAS 입장에서는 정상 처리가 된 것이다. 이렇게 예외를 이곳에서 모두 처리할 수 있다는 것이
핵심이다.<br/>
서블릿 컨테이너까지 예외가 올라가면 복잡하고 지저분하게 추가 프로세스가 실행된다. 반면에
ExceptionResolver 를 사용하면 예외처리가 상당히 깔끔해진다.<br/>
<br/>
스프링 부트가 기본으로 제공하는 ExceptionResolver 는 다음과 같다.<br/>
HandlerExceptionResolverComposite 에 다음 순서로 등록<br/>
1. ExceptionHandlerExceptionResolver<br/>
2. ResponseStatusExceptionResolver<br/>
3. DefaultHandlerExceptionResolver 우선 순위가 가장 낮다.<br/>
<br/>
ExceptionHandlerExceptionResolver<br/>
@ExceptionHandler 을 처리한다. API 예외 처리는 대부분 이 기능으로 해결한다. 조금 뒤에 자세히
설명한다.<br/>
<br/>
ResponseStatusExceptionResolver<br/>
HTTP 상태 코드를 지정해준다.<br/>
예) @ResponseStatus(value = HttpStatus.NOT_FOUND)<br/>
<br/>
DefaultHandlerExceptionResolver<br/>
스프링 내부 기본 예외를 처리한다.<br/>
<br/>
ResponseStatusExceptionResolver 는 예외에 따라서 HTTP 상태 코드를 지정해주는 역할을 한다.<br/>
<br/>
@ResponseStatus 가 달려있는 예외<br/>
@ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "잘못된 요청 오류")<br/>
public class BadRequestException extends RuntimeException {<br/>
}<br/>
BadRequestException 예외가 컨트롤러 밖으로 넘어가면 ResponseStatusExceptionResolver 예외가
해당 애노테이션을 확인해서 오류 코드를 HttpStatus.BAD_REQUEST (400)으로 변경하고, 메시지도
담는다.<br/>
ResponseStatusExceptionResolver 코드를 확인해보면 결국 response.sendError(statusCode, 
resolvedReason) 를 호출하는 것을 확인할 수 있다.<br/>
sendError(400) 를 호출했기 때문에 WAS에서 다시 오류 페이지( /error )를 내부 요청한다<br/>
<br/>
ResponseStatusException 예외<br/>
@ResponseStatus 는 개발자가 직접 변경할 수 없는 예외에는 적용할 수 없다. (애노테이션을 직접 넣어야
하는데, 내가 코드를 수정할 수 없는 라이브러리의 예외 코드 같은 곳에는 적용할 수 없다.)
추가로 애노테이션을 사용하기 때문에 조건에 따라 동적으로 변경하는 것도 어렵다. 이때는
ResponseStatusException 예외를 사용하면 된다.<br/>
@GetMapping("/api/response-status-ex2")<br/>
public String responseStatusEx2() {<br/>
 throw new ResponseStatusException(HttpStatus.NOT_FOUND, "error.bad", new<br/>
IllegalArgumentException());<br/>
}<br/>
<br/>
이번에는 DefaultHandlerExceptionResolver 를 살펴보자.<br/>
DefaultHandlerExceptionResolver 는 스프링 내부에서 발생하는 스프링 예외를 해결한다.<br/>
대표적으로 파라미터 바인딩 시점에 타입이 맞지 않으면 내부에서 TypeMismatchException 이
발생하는데, 이 경우 예외가 발생했기 때문에 그냥 두면 서블릿 컨테이너까지 오류가 올라가고, 결과적으로
500 오류가 발생한다.<br/>
그런데 파라미터 바인딩은 대부분 클라이언트가 HTTP 요청 정보를 잘못 호출해서 발생하는 문제이다. <br/>
HTTP 에서는 이런 경우 HTTP 상태 코드 400을 사용하도록 되어 있다.<br/>
DefaultHandlerExceptionResolver 는 이것을 500 오류가 아니라 HTTP 상태 코드 400 오류로
변경한다.<br/>
스프링 내부 오류를 어떻게 처리할지 수 많은 내용이 정의되어 있다.<br/>
코드 확인<br/>
DefaultHandlerExceptionResolver.handleTypeMismatch 를 보면 다음과 같은 코드를 확인할 수
있다.<br/>
response.sendError(HttpServletResponse.SC_BAD_REQUEST) (400)<br/>
결국 response.sendError() 를 통해서 문제를 해결한다.<br/>
sendError(400) 를 호출했기 때문에 WAS에서 다시 오류 페이지( /error )를 내부 요청한다.<br/>
ex) <br/>
@GetMapping("/api/default-handler-ex")<br/>
public String defaultException(@RequestParam Integer data) {<br/>
 return "ok";<br/>
}<br/>
Integer data 에 문자를 입력하면 내부에서 TypeMismatchException 이 발생한다.<br/>
실행<br/>
http://localhost:8080/api/default-handler-ex?data=hello&message=<br/>
<br/>
지금까지 HTTP 상태 코드를 변경하고, 스프링 내부 예외의 상태코드를 변경하는 기능도 알아보았다. <br/>
그런데 HandlerExceptionResolver 를 직접 사용하기는 복잡하다. API 오류 응답의 경우 response 에
직접 데이터를 넣어야 해서 매우 불편하고 번거롭다. ModelAndView 를 반환해야 하는 것도 API에는 잘
맞지 않는다.<br/>
스프링은 이 문제를 해결하기 위해 @ExceptionHandler 라는 매우 혁신적인 예외 처리 기능을 제공한다. <br/>
그것이 아직 소개하지 않은 ExceptionHandlerExceptionResolver 이다.<br/>
<br/>
API 예외 처리 - @ExceptionHandler 22<br/>
HTML 화면 오류 vs API 오류<br/>
웹 브라우저에 HTML 화면을 제공할 때는 오류가 발생하면 BasicErrorController 를 사용하는게
편하다.<br/>
이때는 단순히 5xx, 4xx 관련된 오류 화면을 보여주면 된다. BasicErrorController 는 이런 메커니즘을
모두 구현해두었다.<br/>
그런데 API는 각 시스템 마다 응답의 모양도 다르고, 스펙도 모두 다르다. 예외 상황에 단순히 오류 화면을
보여주는 것이 아니라, 예외에 따라서 각각 다른 데이터를 출력해야 할 수도 있다. 그리고 같은 예외라고
해도 어떤 컨트롤러에서 발생했는가에 따라서 다른 예외 응답을 내려주어야 할 수 있다. 한마디로 매우
세밀한 제어가 필요하다.<br/>
앞서 이야기했지만, 예를 들어서 상품 API와 주문 API는 오류가 발생했을 때 응답의 모양이 완전히 다를 수
있다.<br/>
결국 지금까지 살펴본 BasicErrorController 를 사용하거나 HandlerExceptionResolver 를 직접
구현하는 방식으로 API 예외를 다루기는 쉽지 않다.<br/>
API 예외처리의 어려운 점<br/>
HandlerExceptionResolver 를 떠올려 보면 ModelAndView 를 반환해야 했다. 이것은 API 응답에는
필요하지 않다.<br/>
API 응답을 위해서 HttpServletResponse 에 직접 응답 데이터를 넣어주었다. 이것은 매우 불편하다.<br/>
스프링 컨트롤러에 비유하면 마치 과거 서블릿을 사용하던 시절로 돌아간 것 같다.<br/>
특정 컨트롤러에서만 발생하는 예외를 별도로 처리하기 어렵다. 예를 들어서 회원을 처리하는 컨트롤러에서
발생하는 RuntimeException 예외와 상품을 관리하는 컨트롤러에서 발생하는 동일한
RuntimeException 예외를 서로 다른 방식으로 처리하고 싶다면 어떻게 해야할까?<br/>
<br/>
@ExceptionHandler<br/>
스프링은 API 예외 처리 문제를 해결하기 위해 @ExceptionHandler 라는 애노테이션을 사용하는 매우
편리한 예외 처리 기능을 제공하는데, 이것이 바로 ExceptionHandlerExceptionResolver 이다.<br/>
스프링은 ExceptionHandlerExceptionResolver 를 기본으로 제공하고, 기본으로 제공하는
ExceptionResolver 중에 우선순위도 가장 높다. 실무에서 API 예외 처리는 대부분 이 기능을 사용한다.<br/>
<br/>
![image](https://user-images.githubusercontent.com/69129562/206461218-45dae5e3-7999-490a-887b-1a61d87ad3ce.png)
@ExceptionHandler 예외 처리 방법<br/>
@ExceptionHandler 애노테이션을 선언하고, 해당 컨트롤러에서 처리하고 싶은 예외를 지정해주면 된다.<br/>
해당 컨트롤러에서 예외가 발생하면 이 메서드가 호출된다. 참고로 지정한 예외 또는 그 예외의 자식
클래스는 모두 잡을 수 있다.<br/>
<br/>
우선순위<br/>
스프링의 우선순위는 항상 자세한 것이 우선권을 가진다. 예를 들어서 부모, 자식 클래스가 있고 다음과 같이
예외가 처리된다.<br/>
@ExceptionHandler(부모예외.class)<br/>
public String 부모예외처리()(부모예외 e) {}<br/>
@ExceptionHandler(자식예외.class)<br/>
public String 자식예외처리()(자식예외 e) {}<br/>
@ExceptionHandler 에 지정한 부모 클래스는 자식 클래스까지 처리할 수 있다. 따라서 자식예외 가
발생하면 부모예외처리() , 자식예외처리() 둘다 호출 대상이 된다. 그런데 둘 중 더 자세한 것이 우선권을
가지므로 자식예외처리() 가 호출된다. 물론 부모예외 가 호출되면 부모예외처리() 만 호출 대상이 되므로
부모예외처리() 가 호출된다.<br/>
<br/>
다양한 예외<br/>
다음과 같이 다양한 예외를 한번에 처리할 수 있다.<br/>
@ExceptionHandler({AException.class, BException.class})<br/>
public String ex(Exception e) {<br/>
 log.info("exception e", e);<br/>
}<br/>
예외 생략<br/>
@ExceptionHandler 에 예외를 생략할 수 있다. 생략하면 메서드 파라미터의 예외가 지정된다.<br/>
@ExceptionHandler<br/>
public ResponseEntity<ErrorResult> userExHandle(UserException e) {}<br/>
<br/>
IllegalArgumentException 처리<br/>
@ResponseStatus(HttpStatus.BAD_REQUEST)<br/>
@ExceptionHandler(IllegalArgumentException.class)<br/>
public ErrorResult illegalExHandle(IllegalArgumentException e) {<br/>
 log.error("[exceptionHandle] ex", e);<br/>
 return new ErrorResult("BAD", e.getMessage());<br/>
}<br/>
실행 흐름<br/>
컨트롤러를 호출한 결과 IllegalArgumentException 예외가 컨트롤러 밖으로 던져진다.<br/>
예외가 발생했으로 ExceptionResolver 가 작동한다. 가장 우선순위가 높은
ExceptionHandlerExceptionResolver 가 실행된다.<br/>
ExceptionHandlerExceptionResolver 는 해당 컨트롤러에 IllegalArgumentException 을 처리할
수 있는 @ExceptionHandler 가 있는지 확인한다.<br/>
illegalExHandle() 를 실행한다. @RestController 이므로 illegalExHandle() 에도
@ResponseBody 가 적용된다. 따라서 HTTP 컨버터가 사용되고, 응답이 다음과 같은 JSON으로 반환된다.<br/>
@ResponseStatus(HttpStatus.BAD_REQUEST) 를 지정했으므로 HTTP 상태 코드 400으로 응답한다<br/>
결과<br/>
{<br/>
 "code": "BAD",<br/>
 "message": "잘못된 입력 값"<br/>
}<br/>
<br/>
@ControllerAdvice<br/>
@ControllerAdvice 는 대상으로 지정한 여러 컨트롤러에 @ExceptionHandler , @InitBinder 기능을
부여해주는 역할을 한다.<br/>
@ControllerAdvice 에 대상을 지정하지 않으면 모든 컨트롤러에 적용된다. (글로벌 적용)<br/>
@RestControllerAdvice 는 @ControllerAdvice 와 같고, @ResponseBody 가 추가되어 있다.<br/>
@Controller , @RestController 의 차이와 같다.<br/>
![image](https://user-images.githubusercontent.com/69129562/206464130-cf16080f-cbd0-4445-8591-1202c855848d.png)


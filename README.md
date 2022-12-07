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



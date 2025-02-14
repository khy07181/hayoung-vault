---
title: spring boot mvc
aliases: []
classification: resource
tags:
  - springBoot
  - java
  - framework
  - web
  - mvc
created: 2022-09-13 15:55
updated: 2025-01-18T21:34
---

# Spring Boot MVC

## 소개

- 간단한 컨트롤러와 테스트 코드를 작성해보면 아래의 테스트에서 아무런 설정파일을 작성하지 않았지만 스프링 MVC 기능을 사용할 수 있다.
- 이것은 스프링 부트가 제공해주는 기본 설정 덕분이다.
	* spring-boot-autoconfigure -> spring.factories -> WebMvcAutoConfiguration 클래스

```java
@RunWith(SpringRunner.class)
@WebMvcTest(UserControllerTest.class)   //@WebMvcTest 애노테이션을 사용하면 MockMvc를 주입받아서 사용할 수 있다.
public class UserControllerTest {

    @Autowired
    MockMvc mockMvc;

    @Test
    public void hello() throws Exception {
        mockMvc.perform(get("/hello"))
                .andExpect(status().isOk())
                .andExpect(content().string("hello"));

    }
}
```

```java
@RestController
public class UserController {

    @GetMapping("/hello")
    public String hello() {
        return "hello";
    }
}
```

### 스프링 부트가 제공해주는 Web MVC 기능의 확장, 재정의

- 기본적으로 제공해주는 기능에서 추가적인 설정이 필요한 경우 아래 코드와 같은 설정 파일을 만들면 된다.
- 인터페이스 WebMvcConfigurer 제공하는 여러 콜백 메소드를 사용해 커스터마이징을 하면 된다.
- 이 때, 확장을 하고싶은 경우 `@EnableWebMvc`를 붙이면 안된다.
- 붙이면 스프링 부트가 제공하는 모든 웹 MVC 기능들이 사라지고 직접 웹 MVC 관련 설정을 해야한다.
- 반대로 `@EnableWebMvc`를 붙여 재정의 할 수 있다.

```java
@Configuration
// @EnableWebMvc 사용 하면 안된다.
public class WebConfig implements WebMvcConfigurer {
    ...
}
```

## [HttpMessageConverters](https://docs.spring.io/spring/docs/5.0.7.RELEASE/spring-framework-reference/web.html#mvc-config-message-converters)

- 스프링 프레임 워크에서 제공하는 인터페이스로 스프링 MVC의 일부분.
- HTTP 요청 본문을 객체로 변경하거나, 객체를 HTTP 응답 본문으로 변경할 때 사용.
- 어떤 요청을 받았는지, 응답을 보내는지에 따라서 메세지컨버터가 달라진다.
	* 요청이 JSON 이고, 본문도 JSON이면 JSON MessageConverter가 사용된다.
- 주로 @RequestBody, @ResponseBody와 같이 사용한다.
	* @RestController가 붙어있으면 @ResponseBody를 생략 가능하다.

```java
@RunWith(SpringRunner.class)
@WebMvcTest(UserController.class)
public class UserControllerTest {

    @Autowired
    MockMvc mockMvc;

    @Test
    public void createUser_JSON() throws Exception {
        String userJson = "{\"username\":\"hayoung\", \"password\":\"123\"}";
        //요청을 만드는 단계
        mockMvc.perform(post("/users/create")
                .contentType(MediaType.APPLICATION_JSON_UTF8)
                .accept(MediaType.APPLICATION_JSON_UTF8)
                .content(userJson))
                //응답 확인단계
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$.username", is(equalTo("hayoung"))))
                    .andExpect(jsonPath("$.password", is(equalTo("123"))));

    }
}
```

```java
@RestController
public class UserController {

    @GetMapping("/hello")
    public String hello(){
        return "hello";
    }

    @PostMapping("/users/create")
    public User create(@RequestBody User user){
        return user;
    }
}
```

```java
public class User {
    private Long id;
    private String username;
    private String password;

    ... getter and setter ...
}
```

## ViewResolver

- 스프링부트에 등록 되어있는 스프링 웹 MVC의 ContentNegotiatingViewResolver가 어떤 contentType일 때 어떤 응답을 보내고, accept header 요청에 의해서 해당 요청에 맞는 응답을 보내는 작업을 알아서 해준다.
- ContentNegotiatingViewResolver는 ViewResolver 중의 하나로 들어오는 요청의 accept header에 따라 응답이 달라진다.
	* accept header : 브라우저 또는 클라이언트가 어떠한 타입의 응답을 원한다고 서버에게 알려주는 것.
	* 어떠한 요청이 들어오면 응답을 만들어 낼 수 있는 모든 View를 찾아내고, View의 타입을 Accept Header랑 비교를 해서 최종적으로 선택을 하고 리턴한다.
	* 판단하기 가장 좋은 정보는 Accept Header이지만 경우에 따라 Accept Header를 제공하지 않는 경우도 있다.
	* Accept Header가 없는 경우 format이라는 parameter를 사용하여 /path?format=pdf와 같은 형식으로 알 수 있다.
- 요청은 JSON으로 보내고 응답은 XML로 받으려면 다음과 같이 코드를 작성하면 된다.

```java
@Test
public void createUser_XML() throws Exception {
    String userJson = "{\"username\":\"hayoung\", \"password\":\"123\"}";
    mockMvc.perform(post("/users/create")
            .contentType(MediaType.APPLICATION_JSON_UTF8)
            .accept(MediaType.APPLICATION_XML)  // accept header를 XML로 변경
            .content(userJson))
                .andExpect(status().isOk())
                .andExpect(xpath("/User/username").string("hayoung")) // jsonPath가 아니라 xpath로 확인
                .andExpect(xpath("/User/password").string("123"));

}
```

- 테스트를 돌리면 406 HttpMediaTypeNotAcceptableException 에러가 발생한다.

### 해결 방법

- HttpMessageConverter는 HttpMessageConvertersAutoConfiguration으로 인해서 적용된다.
- JacksonHttpMessageConvertersConfiguration 클래스에는 MappingJackson2XmlHttpMessageConverterConfiguration이라는 XML을 converting 해주는 converter가 있고
- MappingJackson2XmlHttpMessageConverterConfiguration 클래스에는 `@ConditionalOnClass(XmlMapper.class)`으로 XmlMapper클래스가 classpath에 존재해야만 등록되도록 정의되어 있다.
- 406 에러가 발생한 이유는 XML로 변환할 수 있는 컨버터가 등록되어있지 않기 때문이다.
- XML 메세지 converter 추가하기
	* pom.xml에 의존성을 추가하면 된다.

```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
    <version>2.9.8</version>
</dependency>
```

## 정적 리소스 지원

- 정적 리소스는 동적으로 생성하지 않은 것, 즉 웹브라우저나 클라이언트에서 요청이 들어왔을 때 해당하는 리소스가 이미 만들어져 있고 보내주기만 하면 되는 것이다.
- 기본 리소스 맵핑
	* classpath:/static
	* classpath:/public
	* classpath:/resources/
	* classpath:/META-INF/resources
	* 기본 리소스들은 "/**"에 맵핑에 되서 제공이 된다.
	* ex) "/hello.html"이라는 요청이 들어오면 /static/hello.html이라는 파일이 있으면 요청한 쪽으로 보내준다. (나머지도 마찬가지)
- Last-Modified 헤더를 보고 변경 사항이 없으면 304 응답을 보낸다. (리소스를 다시 보내지 않기 때문에 응답이 빠르다)
- ResourceHttpRequestHandler가 처리한다.

### 맵핑 설정 변경

- 기본적으로 root부터 맵핑이 되어 있는데 맵핑을 변경하고 싶다면 application.properties에 spring.mvc.static-path-pattern값을 주면 된다.
- ex) `spring.mvc.static-path-pattern=/static/**`
	* 이렇게 주면 root부터 요청할 수 없고 전부 static으로 요청해야한다.

### 커스터마이징

- spring.mvc.static-locations로 리소스 찾을 위치를 변경 가능하지만 기본 리소스 위치를 안쓰게 되서 추천되지 않는다.
- 추천되는 방법 : WebMvcConfigurer의 addResourceHandlers로 커스터마이징이 가능하다.
	* ResourceHandler를 추가하는 것으로 기존의 스프링 부트가 제공하는 4가지 ResourceHandler를 유지하면서 원하는 Handler만 추가할 수 있다.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/m/**")    // m으로 시작하는 요청이 오면
            .addResourceLocations("classpath:/m/")  // classpathth 기준으로 m디렉토리 밑에서 제공하겠다. 반드시 /로 끝나야 한다.
            .setCachePeriod(20);    // 초 단위
    }
}
```

## 웹 JAR

- 스프링 부트는 웹 JAR에 대한 기본 맵핑을 제공하기 때문에 클라이언트에서 사용하는 자바스크립트 라이브러리(jQuery, BootStrap 등등)를 웹JAR 형태로 dependency를 추가해서 사용할 수 있다.
- [MVN 중앙 저장소](https://mvnrepository.com/)에도 올라와 있기 때문에 검색해서 pom.xml에 의존성 추가하면 사용 가능하다.
- 예제 코드(jQuery)

```xml
<dependency>
    <groupId>org.webjars.bower</groupId>
    <artifactId>jquery</artifactId>
    <version>3.5.1</version>
</dependency>
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
Hello Static Resource

<script src="/webjars/jquery/3.5.1/dist/jquery.min.js"></script>
<script>
    ${function() {
        alert("ready!");
    })
</script>
</body>
</html>
```

- 버젼을 생략할 수 있는 기능이 있다.
	* 버전을 올릴 때 마다 코드 수정을 안해도 된다.
	* webjars-locator-core 의존성을 추가하면 된다.

```xml
<dependency>
    <groupId>org.webjars</groupId>
    <artifactId>webjars-locator-core</artifactId>
    <version>0.45</version>
</dependency>
```

* 내부적인 동작은 springframework의 resource chaining에 의해서 이루어진다. (필요할 때 공부)

## index 페이지와 파비콘

- index 페이지(welcome 페이지) : 애플리케이션 root를 요청했을 때 보여주는 페이지.
	* 리소스를 제공해주는 기본 위치(static, public, resources, META-INF/resources) 중 아무 곳이나 index.html 파일을 두면 된다.
	* index.html을 찾아 보고 제공
	* index.템플릿을 찾아 보고 제공
	* 둘 다 없으면 에러 페이지
- fabicon : title 옆에 있는 아이콘
	* 파비콘도 마찬가지로 기본 위치에 놓으면 된다.
	* 파비콘이 바뀌지 않으면 직접 파비콘 경로로 요청을 하고 새로고침 후 브라우저를 껐다 키면 바뀐다.
	* [파비콘 만들기 사이트](https://favicon.io/)

## Thymeleaf

- 비교적 최근에 만들어진 템플릿 엔진.
- 템플릿 엔진은 주로 뷰를 만드는데 사용한다.(뷰를 만드는데만 사용하는 것은 아니다. 뷰 뿐만 아니라 code generation이나 e-mail 템플릿 등에도 사용할 수 있다)
	* **Thymeleaf**
	* FreeMarker
	* Groovy
	* Mustache
- 뷰를 만드는데 템플릿 엔진을 쓰는 이유 : 기본적인 템플릿은 같은데 값들이 경우에 따라 달라지는 동적인 컨텐츠를 표현해야 하기 위해 사용한다.

### [JSP를 권장하지 않는 이유](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-jsp-limitations)

- 스프링 부트가 지향하는 바와 대립이 있다.
- JAR 패키징 할 때는 동작하지 않고, WAR 패키징을 해야한다.
- Undertow는 JSP를 지원하지 않는다.

### 스프링 부트에서 Thymeleaf 사용하기

- spring-boot-starter-thymeleaf 의존성 추가
- 기본적으로 자동 설정이 적용되면 동적으로 생성되는 모든 뷰들은 src/main/resources/templates에서 찾게 된다.
- 예시 코드

```java
@RunWith(SpringRunner.class)
@WebMvcTest(SampleController.class)
public class SampleControllerTest {

    @Autowired
    MockMvc mockMvc;

    @Test
    public void hello() throws Exception {
        // 요청 "/hello"
        // 응답
        // - 모델 name : hayoung
        // - 뷰 이름 : hayoung
        mockMvc.perform(get("/hello"))
                .andExpect(status().isOk())
                .andDo(print())
                .andExpect(view().name("hello"))
                .andExpect(model().attribute("name", is("hayoung")));
    }
}
```

```java
@Controller
public class SampleController {

    @GetMapping("/hello")
    public String hello(Model model) {
        model.addAttribute("name", "hayoung");
        return "hello";
    }
}
```

```html
<!-- src/main/resources/templates 폴더에 생성 -->
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Title</title>
</head>
<body>
    <h1 th:text="${name}">Name</h1>
</body>
</html>
```

## HtmlUnit

- HTML 템플릿 뷰 테스트를 보다 전문적으로 사용 가능하다.
- htmlunit 의존성을 추가하면 사용할 수 있다.
	* test scope이므로 test할 때만 사용된다.

```xml
<dependency>
  <groupId>​org.seleniumhq.selenium​</groupId>
  <artifactId>​htmlunit-driver​</artifactId>
  <scope>​test​</scope>
</dependency>
<dependency>
  <groupId>​net.sourceforge.htmlunit​</groupId>
  <artifactId>​htmlunit​</artifactId>
  <scope>​test​</scope>
</dependency>
```

- webClient로 요청 페이지, 태그, 엘리먼트 등을 가져와서 단위 테스트 한다.
- Form submit, 특정 브라우저인척 하기, html 문서의 엘리먼트를 가져와서 값들을 비교하거나 확인 해볼 수 있다.
- 예시 코드

```java
@RunWith(SpringRunner.class)
@WebMvcTest(SampleController.class)
public class SampleControllerTest {

    @Autowired
    WebClient webClient;    // MockMvc가 아니라 WebClient를 주입받을 수 있다. 

    @Test
    public void hello() throws Exception {
        HtmlPage page = webClient.getPage("/hello"); // Page라는 인터페이스로 나온느데 HtmlPage로 바꾸면 다양한 메소드 사용 가능
        HtmlHeading1 h1 = page.getFirstByXPath("//h1"); //h1 제일 앞에있는 것 하나만 가져오기(Object타입을 HtmlHeading1으로 바꾸기)
        assertThat(h1.getTextContent()).isEqualToIgnoringCase("hayoung"); // 대소문자인것을 무시하고 문자열이 같은지 비교
    }
}
```

## ExceptionHandler

- 스프링 부트가 제공하는 기본 예외 처리기
	* BasicErrorController
	- HTML과 JSON 응답 지원
- 커스터마이징 방법
	* ErrorController 타입의 클래스를 구현하고 bean으로 등록하면 BasicErrorController 대신 동작한다.

### 스프링 웹 애노테이션 기반의 MVC에서 ExceptionHandling

```java
@Controller
public class SampleController {

    @GetMapping("/hello")
    public String hello() {
        throw new SampleException();
    }

    // AppError: App에서 만든 커스텀한 에러정보를 담고 있는 클래스가 있다면
    @ExceptionHandler(SampleException.class)
    //메서드 파라메터로 해당하는 Exception 정보를 받아 올 수 있음
    public @ResponseBody AppError sampleError(SampleException e) {
        AppError appError = new AppError();
        appError.setMessage("error.app.key");
        appError.setReason("IDK IDK IDK");
        return appError;
    }
}
```

```java
public class SampleException extends RuntimeException {
}
```

```java
public class AppError {

    private String message;

    private String reason;

    ... getter and setter ...
}
```

- @ExceptionHandler는 Controller 안에서만 사용하고 전역적으로 사용하고 싶다면 클래스를 따로 만들어 @ControllerAdvice 애노테이션을 붙이고 그 안에 ExceptionHandler를 정의하면 여러 Controller에서 발생하는 SampleException을 처리하는 Handler가 동작하게 된다.

### [커스텀 에러 페이지 만들기](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-error-handling-custom-error-pages)

- 상태 코드 값에 따라 에러 페이지 보여주기
- src/main/resources/static|template/error/
	* 파일 이름이 상태 코드의 값과 같거나 x를 사용해 비슷한 html을 만들면 된다.
	* 404.html
	* 5xx.html
	* 좀 더 동적인 뷰로 다양하게 커스터마이징 하고싶으면 ErrorViewResolver를 구현하면 된다.

## Spring HATEOAS

- Spring HATEOAS는 HATEOAS를 구현하기 위해 편리한 기능을 제공하는 라이브러리
	* HATEOAS(Hypermedia As The Engine Of Application State)
		- 서버: 현재 리소스와 **연관된 링크 정보**​를 클라이언트에게 제공한다.
		- 클라이언트: **연관된 링크 정보​**를 바탕으로 리소스에 접근한다.
		- 연관된 링크 정보
			* Rel​ation(Rel)
			* H​ypertext Ref​erence(Href)
- 즉, 관련된 링크 정보들을 리소스에 추가해서 리턴하는식으로 구현하고 클라이언트도 그것을 사용하는 방식
- 사용하려면 spring-boot-starter-hateoas 의존성 추가

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

- HATEOAS에 대해 자세히 알고 싶으면 다음 Reference를 순서대로 보면 된다.
	* [https://spring.io/understanding/HATEOAS](https://spring.io/understanding/HATEOAS)
	* [https://spring.io/guides/gs/rest-hateoas/](https://spring.io/guides/gs/rest-hateoas/)
	* [https://docs.spring.io/spring-hateoas/docs/current/reference/html/](https://docs.spring.io/spring-hateoas/docs/current/reference/html/)

### HATEOAS가 자동설정으로 제공하는 기능

- ObjectMapper
	* spring.jackson.*(커스터마이징)
	* Jackson2ObjectMapperBuilder
	* 제공하는 리소스를 JSON으로 변환할 때 사용하는 인터페이스로 자주 사용된다.
- LinkDiscovers
	* 클라이언트 쪽에서 링크 정보를 Rel 이름으로 찾을때 사용할 수 있는 XPath 확장 클래스
	* 사용할 일이 많지 않다.

### HATEOAS 구현 테스트

- 테스트 코드 작성

```java
@RunWith(SpringRunner.class)
@WebMvcTest(SampleController.class)
public class SampleControllerTest {

    @Autowired
    MockMvc mockMvc;

    @Test
    public void hello() throws Exception {
        mockMvc.perform(get("/hello"))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$._links.self").exists());
    }
}
```

- Hello 클래스 작성

```java
public class Hello {

    private String prefix;
    private String name;

    public String getPrefix() {
        return prefix;
    }

    public void setPrefix(String prefix) {
        this.prefix = prefix;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return prefix + " " + name;
    }
}
```

- Controller 작성

```java
@RestController
public class SampleController {

    @GetMapping("/hello")
    public Resource<Hello> hello(){
        Hello hello = new Hello();
        hello.setPrefix("Hey,");
        hello.setName("Hayoung");

        // 링크 정보를 추가
        Resource<Hello> helloResource = new Resource<>(hello);
        //SampleController 클래스에 존재하는 hello라는 메서드에 대한 링크를 따서 self라는 릴레이션을 만들어서 추가
        helloResource.add(linkTo(methodOn(SampleController.class).hello()).withSelfRel());

        return helloResource;
    }
}
```

## CORS

- [Reference](https://docs.spring.io/spring/docs/5.0.7.RELEASE/spring-framework-reference/web.html#mvc-cors)

### SOP와 CORS

- SOP(Single-Origin-Policy)
	* 같은 Origin에만 요청을 보낼 수 있다는 Policy
- CORS(Cross-Origin Resource Sharing)
	* 서로 다른 Origin끼리 리소스를 share할 수 있는 방법을 제공하는 Policy
	* SOP를 우회하기 위한 표준 기술
- Origin이란?
	* URI 스키마(http, https)
	* hostname(hayoung.me, localhost)
	* 포트(8080, 18080)
	* 세 가지를 조합한 것이 Origin이다.
- 둘 다 웹 브라우저가 지원하는 표준 기술이다.
- 기본적으로는 SOP가 적용되어 있기 때문에 Origin이 다르면 호출할 수 없다.
	* 예를 들어 REST API가 localhost 8080을 통해 띄워져 있는 상태에서 18080이 띄운 웹브라우저에서 8080의 리소스를 가져오지 못한다.
	* 그것을 우회하기 위한 기법이 CORS이다.

### 스프링 MVC에서의 @CrossOrigin

- 스프링 부트에서는 @CrossOrigin에 관한 빈 설정들을 자동으로 해주기 때문에 그냥 사용하면 된다.
	* 스프링 MVC에서 쓰려면 여러가지 bean설정을 해줘야 한다.
- 사용방법
	* @Controller나 @RequestMapping에 추가하거나
	* WebMvcConfigurer를 사용해서 글로벌 설정
- Server쪽 예시 코드

```java
@SpringBootApplication
@RestController
public class Application {

    //@CrossOrigin 애노테이션을사용하여 허용할 Origin들을 설정
    @CrossOrigin(origins = "http://localhost:18080")
    @GetMapping("/hello")
    public String hello() {
        return "hello";
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

- 글로벌 설정
	* Origin 설정을 모든 메소드에 할 순 없으니 @CrossOrigin을 Controller에 붙이거나 여러 Controller에 걸쳐 설정을 해야한다면 WebMvcConfigurer를 구현한 Web관련된 설정파일을 따로 만든다.
	* 이렇게 만들면 스프링 부트가 제공하는 스프링 MVC 기능을 다 사용하면서 추가로 확장하는 것이다.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/hello") // hello를 registry에 등록 "/**" 하면 전부다 허용
                .allowedOrigins("http://localhost:18080");
    }
}
```

- 요청 Client쪽 예시 코드(jQuery 의존성을 추가해서 ajax로 CORS 동작 확인)
	* application.properties에서 `server.port=18080`으로 포트 변경

```xml
<dependency>
    <groupId>org.webjars.bower</groupId>
    <artifactId>jquery</artifactId>
    <version>3.5.1</version>
</dependency>
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
  <h1>CORS Client</h1>
</body>
<script src="/webjars/jquery/3.5.1/dist/jquery.min.js"></script>
<script>
  $(function () {
    $.ajax("http://localhost:8080/hello")
      .done(function (msg) {
        alert(msg);
      })
      .fail(function () {
        alert("fail");
      })
  })
</script>
</html>
```

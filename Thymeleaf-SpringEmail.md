# Thymeleaf를 사용하여 Spring에서 이메일 보내기.

> * Sending email in Spring with Thymeleaf 의 번역 문서
>   * https://www.thymeleaf.org/doc/articles/springmail.html 

José Miguel Samper `<jmiguelsamper AT users.sourceforge.net>` 작성

이 문서에서는 여러 종류의 이메일 메시지를 작성하기 위해 Thymeleaf 템플릿을 사용하는 방법을 보여드리고, 이를 Spring의 이메일 유틸리티와 통합하여 간단하지만 강력한 이메일 시스템을 구성하는 방법을 보여드리겠습니다.

이 문서와 해당 예제 앱은 Spring 프레임워크를 사용하지만, Spring을 사용하지 않는 애플리케이션에서 이메일 템플릿을 처리하는 데에도 Thymeleaf를 사용할 수 있습니다. 또한 예제 애플리케이션은 웹 애플리케이션이지만 Thymeleaf로 이메일을 보내기 위해 앱이 웹을 사용할 필요는 없습니다.



## 1. 전제 조건

이 문서에서는 Thymeleaf와 Spring 4에 모두 익숙하다고 가정합니다. 자세한 내용은 [스프링 문서에서 이메일 챕터](http://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/html/mail.html)를 참조하세요. 자세한 내용은 스프링 메일에 대해 자세히 설명하지 않습니다.



## 2. 예제 애플리케이션

이 글의 모든 코드는 작동 중인 예제 애플리케이션에서 가져온 것입니다. 해당 애플리케이션의 [GitHub 리포지토리](https://github.com/thymeleaf/thymeleaf/tree/3.1-master/examples/spring6/thymeleaf-examples-spring6-springmail)에서 소스를 보거나 다운로드할 수 있습니다. 이 애플리케이션을 다운로드하여 실행하고 소스 코드를 살펴볼 것을 적극 권장합니다(단, `src/main/resources/configuration.properties`에서 SMTP 사용자 이름과 비밀번호(그리고 GMail을 사용하지 않는 경우 SMTP 서버)를 구성해야 함에 유의하세요).



## 3. 스프링으로 이메일 보내기

먼저 다음 코드와 같이 Spring 구성에서 **Mail Sender** 개체를 구성해야 합니다(특정 구성 요구 사항은 다를 수 있음).

```java
@Configuration
@PropertySource("classpath:mail/emailconfig.properties")
public class SpringMailConfig implements ApplicationContextAware, EnvironmentAware {

    private static final String JAVA_MAIL_FILE = "classpath:mail/javamail.properties";

    private ApplicationContext applicationContext;
    private Environment environment;

    ...

    @Bean
    public JavaMailSender mailSender() throws IOException {

        final JavaMailSenderImpl mailSender = new JavaMailSenderImpl();

        // Basic mail sender configuration, based on emailconfig.properties
        mailSender.setHost(this.environment.getProperty(HOST));
        mailSender.setPort(Integer.parseInt(this.environment.getProperty(PORT)));
        mailSender.setProtocol(this.environment.getProperty(PROTOCOL));
        mailSender.setUsername(this.environment.getProperty(USERNAME));
        mailSender.setPassword(this.environment.getProperty(PASSWORD));

        // JavaMail-specific mail sender configuration, based on javamail.properties
        final Properties javaMailProperties = new Properties();
        javaMailProperties.load(this.applicationContext.getResource(JAVA_MAIL_FILE).getInputStream());
        mailSender.setJavaMailProperties(javaMailProperties);

        return mailSender;

    }

    ...

}
```

이전의 코드는 클래스 경로의 특성 파일 `mail/emailconfig.properties` 및 `mail/javamail.properties`에서 구성을 가져오고 있습니다.

Spring은 이메일 메시지 생성을 쉽게 하기 위해 `MimeMessageHelper`라는 클래스를 제공합니다. `mailSender`와 함께 사용하는 방법을 알아보겠습니다.

```java
final MimeMessage mimeMessage = this.mailSender.createMimeMessage();
final MimeMessageHelper message = new MimeMessageHelper(mimeMessage, "UTF-8");
message.setFrom("sender@example.com");
message.setTo("recipient@example.com");
message.setSubject("This is the message subject");
message.setText("This is the message body");
this.mailSender.send(mimeMessage);
```



## 4. Thymeleaf 이메일 템플릿

이메일 템플릿을 처리하기 위해 Thymeleaf를 사용하면 몇 가지 흥미로운 기능을 사용할 수 있습니다.

* Spring EL의 **표현식**.
* 흐름 제어: **반복**, **조건**, …
* **유틸리티 기능**: 날짜/숫자 서식 지정, 목록 처리, 배열…
* 애플리케이션의 Spring 국제화 인프라와 통합된 간편한 **i18n**.
* **자연스러운 템플릿**: 이메일 템플릿은 UI 디자이너가 작성한 정적 프로토타입일 수 있습니다.
* 기타 등등…



또한 Thymeleaf가 서블릿 API에 대한 필수 종속성이 없다는 사실을 감안할 때 웹 애플리케이션에서 이메일을 생성하거나 보낼 필요가 전혀 없습니다. 여기에 설명된 기술은 웹 UI가 없는 독립 실행형 애플리케이션에서 거의 또는 전혀 변경하지 않고 사용할 수 있습니다.



### 4.1 우리의 목표

우리의 예제 애플리케이션은 다섯 가지 유형의 이메일을 보낼 것입니다.

* 텍스트(비 HTML) 이메일.
* 간단한 HTML(국제화된 인사말 포함).
* 첨부 파일이 있는 HTML 텍스트입니다.
* 인라인 이미지가 있는 HTML 텍스트.
* 사용자가 편집한 HTML 텍스트입니다.



### 4.2 스프링 구성

템플릿을 처리하기 위해 Spring Email 구성에서 이메일 처리를 위해 특별히 구성된 **TemplateEngine**을 구성합니다.

```java
@Configuration
@PropertySource("classpath:mail/emailconfig.properties")
public class SpringMailConfig implements ApplicationContextAware, EnvironmentAware {

    ...

    @Bean
    public ResourceBundleMessageSource emailMessageSource() {
        final ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
        messageSource.setBasename("mail/MailMessages");
        return messageSource;
    }

    ...

    @Bean
    public TemplateEngine emailTemplateEngine() {
        final SpringTemplateEngine templateEngine = new SpringTemplateEngine();
        // Resolver for TEXT emails
        templateEngine.addTemplateResolver(textTemplateResolver());
        // Resolver for HTML emails (except the editable one)
        templateEngine.addTemplateResolver(htmlTemplateResolver());
        // Resolver for HTML editable emails (which will be treated as a String)
        templateEngine.addTemplateResolver(stringTemplateResolver());
        // Message source, internationalization specific to emails
        templateEngine.setTemplateEngineMessageSource(emailMessageSource());
        return templateEngine;
    }

    private ITemplateResolver textTemplateResolver() {
        final ClassLoaderTemplateResolver templateResolver = new ClassLoaderTemplateResolver();
        templateResolver.setOrder(Integer.valueOf(1));
        templateResolver.setResolvablePatterns(Collections.singleton("text/*"));
        templateResolver.setPrefix("/mail/");
        templateResolver.setSuffix(".txt");
        templateResolver.setTemplateMode(TemplateMode.TEXT);
        templateResolver.setCharacterEncoding(EMAIL_TEMPLATE_ENCODING);
        templateResolver.setCacheable(false);
        return templateResolver;
    }

    private ITemplateResolver htmlTemplateResolver() {
        final ClassLoaderTemplateResolver templateResolver = new ClassLoaderTemplateResolver();
        templateResolver.setOrder(Integer.valueOf(2));
        templateResolver.setResolvablePatterns(Collections.singleton("html/*"));
        templateResolver.setPrefix("/mail/");
        templateResolver.setSuffix(".html");
        templateResolver.setTemplateMode(TemplateMode.HTML);
        templateResolver.setCharacterEncoding(EMAIL_TEMPLATE_ENCODING);
        templateResolver.setCacheable(false);
        return templateResolver;
    }

    private ITemplateResolver stringTemplateResolver() {
        final StringTemplateResolver templateResolver = new StringTemplateResolver();
        templateResolver.setOrder(Integer.valueOf(3));
        // No resolvable pattern, will simply process as a String template everything not previously matched
        templateResolver.setTemplateMode("HTML5");
        templateResolver.setCacheable(false);
        return templateResolver;
    }

    ...

}
```



이메일 관련 엔진에 대해 세 개의 템플릿 리졸버를 구성했습니다. 하나는 TEXT 템플릿용, 다른 하나는 HTML 템플릿용, 세 번째는 편집 가능한 HTML 템플릿용으로 사용자에게 수정할 기회를 제공하고 템플릿 엔진은 일단 수정되면 단순한 **String**으로 사용됩니다.

세 템플릿 리졸버는 모두 순서대로 실행되어 템플릿 이름에 대해 확인 가능한 패턴을 일치시키고 이름이 일치하는 경우에만 지정된 템플릿을 확인하도록 정렬되어 있습니다.

또한 이 **TemplateEngine**은 이메일 처리 전용이며 웹 인터페이스에 사용되는 **TemplateEngine**과는 완전히 분리되어 있다는 점에 유의하세요. 이 웹 인터페이스용 템플릿 엔진은 **ThymeleafViewResolver**를 통해 Spring MVC와 통합될 것이며, 실제로는 **WebMvcConfigurerAdapter**를 확장하는 다른 **@Configuration** 파일에 정의되어 있습니다(이메일 처리에 집중하기 위해 여기서는 보여드리지 않겠습니다).



### 4.3 템플릿 엔진 실행

코드의 어느 지점에서 메시지 텍스트를 생성하기 위해 템플릿 엔진을 실행해야 합니다. 이 작업을 웹 계층이 아닌 비즈니스 계층의 책임이라는 점을 명확히 하기 위해 이 작업을 **EmailService** 클래스에서 수행하도록 선택했습니다.

Thymeleaf에서 평소와 같이 실행하기 전에 템플릿 실행 중에 사용할 모든 변수가 포함된 컨텍스트를 채워야 합니다. 이메일 처리가 웹에 종속적이지 않다는 사실을 감안할 때, **Context** 인스턴스가 충분합니다:

```java
final Context ctx = new Context(locale);
ctx.setVariable("name", recipientName);
ctx.setVariable("subscriptionDate", new Date());
ctx.setVariable("hobbies", Arrays.asList("Cinema", "Sports", "Music"));
ctx.setVariable("imageResourceName", imageResourceName); // so that we can reference it from HTML

final String htmlContent = this.templateEngine.process("html/email-inlineimage.html", ctx);
```

우리의 **email-inlineimage.html**은 인라인 이미지가 있는 이메일을 보내는 데 사용할 템플릿 파일이며 다음과 같습니다.

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
  <head>
    <title th:remove="all">Template for HTML email with inline image</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
  </head>
  <body>
    <p th:text="#{greeting(${name})}">
      Hello, Peter Static!
    </p>
    <p th:if="${name.length() > 10}">
      Wow! You've got a long name (more than 10 chars)!
    </p>
    <p>
      You have been successfully subscribed to the <b>Fake newsletter</b> on
      <span th:text="${#dates.format(subscriptionDate)}">28-12-2012</span>
    </p>
    <p>Your hobbies are:</p>
    <ul th:remove="all-but-first">
      <li th:each="hobby : ${hobbies}" th:text="${hobby}">Reading</li>
      <li>Writing</li>
      <li>Bowling</li>
    </ul>
    <p>
      You can find <b>your inlined image</b> just below this text.
    </p>
    <p>
      <img src="sample.png" th:src="|cid:${imageResourceName}|" />
    </p>
    <p>
      Regards, <br />
      <em>The Thymeleaf Team</em>
    </p>
  </body>
</html>
```

몇 가지 사항을 짚고 넘어가겠습니다:

* 전자의 템플릿은 완전한 위지와이그(WYSIWYG) 방식이므로 브라우저에서 열어보는 것만으로 템플릿이 어떻게 보이는지 확인할 수 있습니다. 결과를 확인하기 위해 이메일을 보내는 것보다 훨씬 낫지 않나요?
* Thymeleaf의 모든 기능을 사용할 수 있습니다. 예를 들어 여기서는 매개변수화된 `#{...}` 표현식과 함께 i18n을 사용했고, 목록을 반복하는 `th:each`, 날짜 서식을 지정하는 `#dates`를 사용했습니다.
* `img` 엘리먼트에는 프로토타이핑에 유용한 하드코딩된 `src` 값이 있으며, 이 값은 런타임에 첨부된 이미지 파일 이름과 일치하는 `cid:image.jpg`와 같은 값으로 대체됩니다.



### 4.4 Text (non-HTML) 이메일

텍스트 이메일은 어떨까요? 이미 텍스트 이메일 템플릿을 위한 템플릿 리졸버를 구성해 두었기 때문에 Thymeleaf의 텍스트 구문을 사용하여 템플릿을 생성하기만 하면 됩니다:

```
[( #{greeting(${name})} )]

[# th:if="${name.length() gt 10}"]Wow! You've got a long name (more than 10 chars)![/]

You have been successfully subscribed to the Fake newsletter on [( ${#dates.format(subscriptionDate)} )].

Your hobbies are:
[# th:each="hobby : ${hobbies}"]
 - [( ${hobby} )]
[/]

Regards,
    The Thymeleaf Team
```



## 5. 모든 것을 종합하기

## 5.1 서비스 클래스

마지막으로 `EmailService`클래스에서 이 이메일 템플릿을 실행하는 메서드가 어떤 모습인지 살펴봅시다:

```java
public void sendMailWithInline(
  final String recipientName, final String recipientEmail, final String imageResourceName,
  final byte[] imageBytes, final String imageContentType, final Locale locale)
  throws MessagingException {

    // Prepare the evaluation context
    final Context ctx = new Context(locale);
    ctx.setVariable("name", recipientName);
    ctx.setVariable("subscriptionDate", new Date());
    ctx.setVariable("hobbies", Arrays.asList("Cinema", "Sports", "Music"));
    ctx.setVariable("imageResourceName", imageResourceName); // so that we can reference it from HTML

    // Prepare message using a Spring helper
    final MimeMessage mimeMessage = this.mailSender.createMimeMessage();
    final MimeMessageHelper message =
        new MimeMessageHelper(mimeMessage, true, "UTF-8"); // true = multipart
    message.setSubject("Example HTML email with inline image");
    message.setFrom("thymeleaf@example.com");
    message.setTo(recipientEmail);

    // Create the HTML body using Thymeleaf
    final String htmlContent = this.templateEngine.process("email-inlineimage.html", ctx);
    message.setText(htmlContent, true); // true = isHtml

    // Add the inline image, referenced from the HTML code as "cid:${imageResourceName}"
    final InputStreamSource imageSource = new ByteArrayResource(imageBytes);
    message.addInline(imageResourceName, imageSource, imageContentType);

    // Send mail
    this.mailSender.send(mimeMessage);

}
```

사용자가 업로드한 이미지를 첨부하기 위해 이전에 `byte[]`로 변환한 `org.springframework.core.io.ByteArrayResource` 객체를 사용했음을 참고하세요.

파일시스템에서 직접 파일을 첨부하여 메모리에 파일을 로드하지 않으려면 `FileSystemResource`를 사용하거나 원격 파일을 첨부하려면 `UrlResource`를 사용할 수도 있습니다.



## 5.2 컨트롤러

이제 서비스를 호출하는 컨트롤러 메서드입니다:

```java
/*
* Send HTML mail with inline image
*/
@RequestMapping(value = "/sendMailWithInlineImage", method = RequestMethod.POST)
public String sendMailWithInline(
  @RequestParam("recipientName") final String recipientName,
  @RequestParam("recipientEmail") final String recipientEmail,
  @RequestParam("image") final MultipartFile image,
  final Locale locale)
  throws MessagingException, IOException {

    this.emailService.sendMailWithInline(
        recipientName, recipientEmail, image.getName(),
        image.getBytes(), image.getContentType(), locale);
    return "redirect:sent.html";

}
```

이보다 더 쉬울 수는 없습니다. 업로드된 파일을 모델링하고 그 내용을 서비스에 전달하기 위해 Spring MVC `MultipartFile` 객체를 사용하는 방법에 주목하세요.





## 6. 더 많은 예제

간결성을 위해 애플리케이션에서 보낼 수 있는 5가지 이메일 유형 중 한 가지 유형만 자세히 설명했습니다. 그러나 5가지 유형의 이메일을 모두 만드는 데 필요한 소스 코드는 [문서 페이지](https://www.thymeleaf.org/documentation.html)에서 다운로드할 수 있는 `springmail` 예제 애플리케이션에서 확인할 수 있습니다.
# 이것 저것 기타등등... 낙서판



## 1에서 99까지에서 1이 나타나는 갯수를 구하시요?

> 게임 속 이벤트로 왠 퀴즈가 나와서 그냥 프로그래밍 적으로 풀고 싶었다. 😅😅😅

* 처음 했던 것

  ```java
  jshell> IntStream.rangeClosed(1, 99).filter(i->Integer.toString(i).contains("1")).count()
  $8 ==> 19
  ```

  * 그런데 잘못 풀었다. 11이란 숫자에 1이 두개 들어가는 걸 감지하지 못한다. 😅

* 두번 째

  ```java
  jshell> IntStream.rangeClosed(1, 99).reduce(0, (subTotal, i) -> subTotal + (int) Integer.toString(i).chars().filter(c -> c == '1').count())
  $7 ==> 20
  ```

  * 그래서 reduce를 활용해서 각각 숫자에 1이 몇개 포함된는지 합산하는식으로 고침.. 😅



### Thymeleaf를 사용하여 Spring에서 이메일 보내기.

* [Sending email in Spring with Thymeleaf 의 번역 문서](Thymeleaf-SpringEmail.md)
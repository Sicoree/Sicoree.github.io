---
title: ResponseEntity 사용법, 사용이유
author: Sicoree EE
date: 2022-11-15 13:49:00 +0900
categories: [Spring Boot]
tags: [Spring Boot, Refactoring]

---

# ResponseEntity ?

+ httpentity를 상속받는, 결과 데이터와 HTTP 상태 코드를 직접 제어할 수 있는 클래스

+ HTTP 아키텍쳐 형태에 맞게 Response를 보내주는 것에도 의미가 있음
  
  + 에러 코드와 같은 HTTP상태 코드를 전송하고 싶은 데이터와 함께 전송할 수 있기 때문에 세밀한 제어가 가능하다.

# ResponseEntity 구조

+ ResponseEntity는 HttpEntity를 상속받으며 사용자의 응답 데이터가 포함된 클래스 이기 때문에 다음을 포함한다
  
  + HttpStatus
  
  + HttpHeaders (요청, 응답에 대한 요구사항)
  
  + HttpBody (헤더에 대한 내용)

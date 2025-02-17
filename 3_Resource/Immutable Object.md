---
title: Immutable Object
aliases:
  - 불변 객체
classification: resource
tags:
  - OOP
created: 2024-07-29 18:14
updated: 2025-01-18T21:17
---
- 불변 객체는 Thread-Safe 하여 동기화를 고려하지 않아도 된다.
	- 가변 객체를 통해 작업을 하는 도중 예외(Exception)가 발생하면 해당 객체가 불안정한 상태에 빠질 수 있어 또 다른 에러를 유발할 수 있는 위험성이 있기 때문이다.
- 불변 객체로 구성하면 다른 사람이 개발한 함수를 위험없이 이용을 보장할 수 있어 협업에도 유지보수에도 유용하다.

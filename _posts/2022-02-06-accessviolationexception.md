---
layout: post
title: "System.AccessViolationException"
date: 2022-2-6
categories: [ "C#", ".Net" ]
---

## 문제

문제가 생겼다

무슨짓을 해도 System.AccessViolationException을 핸들링할 수가 없다

답은 MS 공식 문서에 서술되어 있다

> Starting with .NET Framework 4, 
> AccessViolationException exceptions thrown by the common language runtime are not handled by the catch statement 
> in a structured exception handler if the exception occurs outside of the memory reserved by the common language runtime.

net4.x부터 CLR에 의해 예약된 메모리 외부에서 오류가 발생하여, CLR에 의해 throw된 AccessViolationException은 구조적인 예외 처리기 문(catch)에서 처리되지 않는다

## 해결

물론, 해결법 또한 MS 공식 문서에 서술되어 있다

> To handle such an AccessViolationException exception, 
> apply the HandleProcessCorruptedStateExceptionsAttribute attribute to the method in which the exception is thrown.

간단히, HandleProcessCorruptedStateExceptionsAttribute를 메서드에 추가하도록 하자

```Csharp
// AccessViolationException을 throw하는 함수
[method: DllExport("xxx.dll")]
private static extern void ThrowAccessViolationException();

[method: HandleProcessCorruptedStateExceptions]
private static void CatchAccessViolationException()
{
	try
	{
		ThrowAccessViolationException();
	}
	catch (AccessViolationException)
	{
		...
	}
}
```

혹은, 호환성을 위해 net3.x 이전의 동작으로 되돌리고 싶다면, App.config를 만들어보자

configuration태그 안의 runtime태그 안에 legacyCorruptedStateExceptionsPolicy태그를 생성하고, enable속성을 true로 바꾸어주자

App.config를 전에 건드린적이 없다면, 아래와 같은 모습이 되어야 한다

```xml
<configuration>
	<runtime>
		<legacyCorruptedStateExceptionsPolicy enabled="true"/>
	</runtime>
</configuration>
```
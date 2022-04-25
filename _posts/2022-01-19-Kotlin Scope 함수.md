---
title: "Kotlin Scope 함수"
date: 2022-01-19 18:19:00 +0900
categories: 안드로이드
tags: 안드로이드 코틀린
---
코틀린에는 객체의 context 내에서 코드 블록을 실행하기 위한 함수들이 있는데 이를 scope 함수라고 한다. Scope 함수에 람다 표현식을 전달하면 해당 코드 블록이 실행되며 이름 없이 객체에 접근할 수 있다. Scope 함수에는 `let`, `run`, `with`, `apply`, `also`가 있다.

기본적으로 이 함수들이 하는 역할 람다로 전달된 코드 블록을 실행하는것으로 같으며, 차이점은 반환 타입과 코드 블록 내에서 객체에 접근하는 방법이다.

{% raw %}
```kotlin
data class GameCharacter(var name: String, var level: Int, var race: String) {
    fun levelUp() {
        level++
    }
}

fun main() {
    // Scope 함수 사용 안 할 시
    val character1 = GameCharacter("Tom", 1, "Orc")
    println(character1) //GameCharacter(name=Tom, level=1, race=Orc)
    character1.levelUp()
    println(character1) //GameCharacter(name=Tom, level=2, race=Orc)
    
    // Scope 함수 사용할 시
    val character2 = GameCharacter("Tom", 1, "Orc").let {
        println(it)     //GameCharacter(name=Tom, level=1, race=Orc)
        it.levelUp()
        println(it)     //GameCharacter(name=Tom, level=2, race=Orc)
    }
}
```
{% endraw %}

Scope 함수를 사용하면 객체의 이름을 일일히 쓰지 않고도 객체에 접근하여 프로퍼티를 변경하거나 메서드를 실행할 수 있다.

## 함수 고르기
상황에 맞는 함수를 고르기 쉽게 코틀린 공식 문서에 다음과 같은 표가 있다.

| 함수 | 객체 참조 | 반환값 | 확장함수인가 |
|:--:|:--:|:--:|:--:|
| `let` | `it` | 람다 결과 | O |
| `run` | `this` | 람다 결과 | O |
| `run` | - | 람다 결과 | X(객체 없이 호출) |
| `with` | `this` | 컨텍스트 객체 | O |
| `also` | `it` | 컨텍스트 객체 | O |

## 함수 종류
많은 경우에 함수를 서로 바꿔도 동작하니 소속 조직의 컨벤션을 따르는 것이 좋다고 한다.
### `let`
반환값: 람다 결과  
컨텍스트 객체: `it`

{% raw %}
```kotlin
//let 안 쓴 버전
val numbers = mutableListOf("one", "two", "three", "four", "five")
val resultList = numbers.map { it.length }.filter { it > 3 }
println(resultList)

//let 쓴 버전
val numbers = mutableListOf("one", "two", "three", "four", "five")
numbers.map { it.length }.filter { it > 3 }.let { 
    println(it)
}

//람다가 하나의 it를 인자로 받는 하나의 함수인 경우 ::로 사용이 가능하다
val numbers = mutableListOf("one", "two", "three", "four", "five")
number.map { it.length }.filter { it > 3 }.let(::println)
```
{% endraw %}

`let`을 사용하면 non-null일때만 실행해야 하는 코드를 작성할 수 있다.

{% raw %}
```kotlin
//non-null일 시에만 실행할 코드
val str: String? = "Hello"
val length = str?.let {
    println("let() called on $it")
    it.length
}
```
{% endraw %}
### `with`
`with`는 일단 확장함수가 아니며, 컨텍스트 객체를 인자로 받는다.

반환값: 람다 결과  
컨텍스트 객체: `this`

람다 결과를 제공하지 않고 컨텍스트 객체에 대해 함수들을 호출할 때 쓰는 것이 권장된다(`with` 컨텍스트 객체 이러이러한 것을 하라).

```kotlin 
val numbers = mutableListOf("one", "two", "three")
with(numbers) {
    println("'with' is called with argument $this")
    println("It contains $size elements")
}
```
### `run`
반환값: 람다 결과  
컨텍스트 객체: `this`

`run`은 `with`와 동일하지만 `let`처럼 컨텍스트 객체의 확장 함수로 호출된다.  
객체 초기화와 결과값 계산을 둘 다 하고 싶을 때 `run`을 쓰면 유용하다. `this`의 경우 생략이 가능하지만 바깥 scope의 프로퍼티와 헷갈릴 수 있으니 적절히 생략해야 한다.

```kotlin
val service = MultiportService("https://example.kotlinlang.org", 80)

val result = service.run {
    port = 8080
    query(prepareRequest() + " to port $port")
}
```
`run`은 컨텍스트 객체의 확장 함수뿐만 아니라 단독으로 사용 가능하다. 이 경우 코드 블록을 실행하고 람다 결과를 반환한다.
```kotlin
val hexNumberRegex = run {
    val digits = "0-9"
    val hexDigits = "A-Fa-f"
    val sign = "+-"

    Regex("[$sign]?[$digits$hexDigits]+")
}
```
### `apply`
반환값: 컨텍스트 객체  
컨텍스트 객체: `this`

`this`가 생략이 가능하고 람다 결과를 반환하지 않기 때문에 객체의 설정을 할 때 쓰인다. 또 객체 자체를 반환하기 때문에 체이닝하여 뒤에 다른 작업을 할 수 있다.
```kotlin
val laptop = Computer("MyPC").apply {
    cpu = "ryzen"
    memory = 4
    storage = 120
}
```
### `also`
반환값: 컨텍스트 객체  
컨텍스트 객체: `it`

반환값이 컨텍스트 객체이니 체이닝해서 다른 작업을 이어서 할 수 있다.
```kotlin
val numbers = mutableListOf("one", "two", "three")
numbers
    .also { println("The list elements before adding new one: $it") }
    .add("four")
```
참고: [Scope functions](https://kotlinlang.org/docs/scope-functions.html)
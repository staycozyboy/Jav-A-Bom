# 동작 파라미터화(Behavior Parameterization)

## 변화하는 문제를 효율적으로 해결하기 위해
소프트웨어 엔지니어링에서 끊임없이 변화하는 문제들을 마주치게 된다. 뛰어난 엔지니어라면, 그에 대한 효율적인 해결책을 찾을 수 있어야한다. 즉, 엔지니어링 비용이 가장 최소화될 수 있는 해결책을 찾아야 한다. 또한, 새로 추가한 기능은 쉽게 구현할 수 있어야 하며 장기적인 관점에서 유지보수가 쉬워야 한다.

이때 유용하게 활용될 수 있는 개념이 바로 동작 파라미터화(Behavior Parameterization)이다. 이는 아직 어떻게 실행할 것인지 결정되지 않은 코드 블럭을 파라미터화시켜 사용할 수 있게 만드는 것을 의미한다. 쉽게 설명하면 **메소드를 일급 객체로** 만들어 준다. 동작 파라미터화를 사용하여 유연하게 코드를 작성할 수 있다는 장점을 얻지만, 쓸데없는 코드가 늘어나게 된다.

자바 8은 람다(Lambda)를 통해 이 문제를 해결한다. 한 가지 문제를 푸는 다양한 해결책을 비효율적인 것부터 순차적으로 살펴보자. 직관적으로 각 개념의 장점을 이해할 수 있을 것이다.

## 변화하는 문제; 사과 골라내기
기존의 농장 재고목록 애플리케이션에 리스트에서 녹색 사과만 필터링하는 기능을 추가한다고 가정하자. 즉, 다음과 같은 문제다. 같이 해결해보자.

* 입력 : 사과의 리스트
* 문제 조건 : 녹색 사과만 골라내라(filter)
* 출력 : 녹색 사과의 리스트

### 기존의 방식으로 문제 풀기 1
다음과 같은 enum class가 존재한다고 가정하자.

`enum Color { RED, GREEN }`

다음은 기존의 방식(고전적인 객체지향)으로 문제를 해결한 코드이다.

```java
public static List<Apple> filterGreen(List<Apple> inventory) {
	List<Apple> result = new ArrayList<>();
    for (Apple apple: inventory) {
    	if (GREEN.equals(apple.getColor()) {  // 문제의 핵심 조건
        	result.add(apple);
        }
    }
	return result;
}
```

위 코드에서 알 수 있듯이 사과 색깔의 녹색 여부가 문제(problem)의 핵심 조건이었다. 이러한 방식으로 문제를 해결했다고 하자. 하지만, 이번에는 빨간 사과만 필터링하는 기능이 필요하다. 이 문제는 어떻게 해결할 것인가? 마찬가지로 이런 방식으로 접근한다면, 위의 코드를 복사하여 핵심 조건 부분만 빨간 사과인지 체크하는 코드로 변경할 것이다. 이러한 방식은 다음과 같은 문제(issue)를 야기한다.

* 보다 더 다양한 색(짙은 빨강, 연한 초록 등)을 필터링하는 문제에 유연하게 대응할 수 없다.
  * 복사/붙여넣기 자체가 비효율적이다(DRY 준수 x).
  * 코드가 길어지고 더러워져서, 직관적이지 못하고 가독성이 떨어진다.
  * 성능 개선 혹은 에러 발생등으로 코드를 수정해야 할 때, 복붙한 모든 코드를 일일이 수정해야한다.

### 기존의 방식으로 문제 풀기 2
조금 더 효율적인 해결책을 구상해보자. 사과의 색상을 파라미터화시키는 것이다. 코드는 다음과 같을 것이다.

```java
public static List<Apple> filterByColor(List<Apple> inventory, Color color) {
	List<Apple> result = new ArrayList<>();
    for (Apple apple: inventory) {
    	if (apple.getColor().equals(color)) {  // 문제의 핵심 조건
        	result.add(apple);
        }
    }
	return result;
}
```
이제 다양한 색을 필터링하는 문제는 복붙없이 해결가능하다! 하지만, 무게가 150그램 이상인 사과만 필터링해야하는 문제를 만난다면 어떻게 될까? 또 복붙하여, 매개변수와 핵심 조건부분만 수정하고 있는 본인을 발견하게 될 것이다. 마찬가지로, **보다 다양한 조건을 기준으로 사과를 필터링하는 문제에 유연하게 대응할 수 없다.** 엔지니어의 관점에서 생각해보자. 이를 어떻게 효율적으로 해결할 수 있을까?

정답은 바로 `filter` 동작을 수행하는 메소드에 필터링 기준을 효과적으로 전달하는 것이다. 그렇다면, 하나의 메소드로도 모든 기준에 대응할 수 있으니까! 이러한 유연성은 동작 파라미터화를 통해 달성할 수 있다.

### 동작 파리미터화로 문제 풀기
그럼, *동작 파리미터화를 사용해 기준을 전달한다* 에서 기준은 어떻게 정의될까? 결국 기준은 사과의 속성을 특정한 조건에 맞게 참/거짓(boolean) 값을 반환해야할 것이다. 이렇게 참/거짓 값을 반환하는 함수를 일반적으로 predicate라고 한다. 이러한 함수를 포함하여 선택 조건을 결정하는 인터페이스(함수형 인터페이스)는 다음과 같은 구조일 것이다.

```java
public interface ApplePredicate {
	boolean test (Apple apple);
}
```

이러한 인터페이스를 구현하는 다양한 클래스를 정의함으로써, 필터링 메소드에 전달할 수 있는 다양한 기준을 만들어 낼 수 있다. 예시는 다음과 같다.

```java
public class AppleGreenColorPredicate implements ApplePredicate {
	public boolean test(Apple apple) {
		return GREEN.equals(apple.getColor());
	}
}
```

이를 활용하면 다양한 기준에 대한 필터링 기능을 수행할 수 있다. `filter` 기능을 하는 메소드에서 `ApplePredicate` 객체를 받고, 그에 따라 사과의 속성을 검사하도록 메소드를 정의해야한다. 이렇게 동작 파라미터화, 즉 메소드가 다양한 동작을 받아서 내부적으로 다양한 동작을 수행할 수 있다. 코드는 다음과 같을 것이다.

```java
public static List<Apple> filterApples(List<Apple> inventory, ApplePredicate predicate) {
	List<Apple> result = new ArrayList<>();
	for (Apple apple: inventory) {
		if (predicate.test(apple)) {  // predicate로 기준을 캡슐화
			result.add(apple);
		}
	}
	return result;
}
```

위 코드에서 확인할 수 있듯이, 비교적 유연한 코드를 얻었으며 동시에 가독성도 더 좋아졌고, 사용하기도 직관적이고 쉬워졌다. 여기서 가장 주목할 점은 동작 파라미터화를 사용하여, **컬렉션 탐색 로직과 각 항목에 적용할 동작을 분리할 수 있다**는 점이다. 따라서, 같은 탐색 로직을 바탕으로 다른 동작을 수행해야하는 경우에도 유연하게 대응할 수 있다(즉, 코드를 재활용할 수 있다).

지금까지 동작을 추상화하여 변화하는 문제에 유연하게 대응할 수 있는 코드를 작성해보았다. 여기까지만 해도 매우 휼륭하고 만족스러운 해결책이라고 느낄 수 있다. 하지만 여러 클래스를 구현해 인스턴스화 하는 과정은 상당히 번거로운 작업이며, 시간낭비다. **이러한 복잡한 과정들을 간소화 시켜보자.**

### 익명 클래스(Anonymous Class)로 간소화하기
익명 클래스(Anonymous Class)는 지역 클래스(Local Class)와 비슷한 개념이다. 말그대로 이름이 없는 클래스이며, 이를 이용해 **클래스 선언과 인스턴스화를 동시에** 할 수 있다. 즉, 다음과 같이 구현할 수 있다.

```java
List<Apple> greenApples = filterApples(inventory, new ApplePredicate() {
	public boolean test(Apple apple) {
    	    return GREEN.equals(apple.getColor());
    }
});
```
휼륭하게 클래스를 선언하는 동시에 인스턴스화를 수행하였지만, 여전히 코드가 장황하다는 느낌을 지울 수 없다. 이는 유지보수 및 협업을 진행하는데 있어 심각한 장애물이다. 한눈에 직관적으로 이해할 수 있는 깔끔한 코드를 지향해야한다. 람다(Lambda)를 이용하여 이러한 목표를 달성해보자.

### 람다(Lambda)로 간소화하기
이전에 익명 클래스를 사용하여 수행한 기능과 동일한 기능을 람다를 사용하면 다음과 같이 구현할 수 있다.

```java
List<Apple> greenApples = filterApples(inventory, (Apple apple) -> GREEN.equals(apple.getColor()));
```

더욱 간결해지고, 한눈에 직관적으로 이해된다. 이것이 바로 람다의 강력한 장점이다. 지금까지의 흐름을 통해 왜 자바가 람다를 도입했는지(도입할 수 밖에 없었는지) 확실히 이해할 수 있길 바란다.

## Reference
* 윤성우의 열혈 Java 프로그래밍(윤성우, 오렌지 미디어, 2017)
* 모던 자바 인 액션(라울-게이브리얼 우르마 외2명, 우정은, 한빛미디어, 2019)
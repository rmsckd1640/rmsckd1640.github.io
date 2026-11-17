---
title: "자바 리플렉션(Reflection)"
date: 2026-08-06 15:26:08 +0900
tags: ["Java", "Spring"]
summary: "리플렉션(reflection)은 반사, 비춰봄이라는 뜻이다. 프로그램이 실행되는 도중에 자기 자신의 구조를 들여다보고, 조작할 수 있는 기능을 말한다. 보통 자바 코드를 쓸 때는 클래스, 메서드, 필드가 컴파일 시점에 이미 다 정해져 있다. 여기서 Product가 어떤 필드를 가지고 있는지,"
---

## 1. 리플렉션이란?

리플렉션(reflection)은 반사, 비춰봄이라는 뜻이다. 프로그램이 실행되는 도중에 자기 자신의 구조를 들여다보고, 조작할 수 있는 기능을 말한다.

보통 자바 코드를 쓸 때는 클래스, 메서드, 필드가 컴파일 시점에 이미 다 정해져 있다.

```java
Product product = new Product();
product.setName("사과");
```

여기서 `Product`가 어떤 필드를 가지고 있는지, `setName`이라는 메서드가 존재하는지는 컴파일러가 컴파일 시점에 미리 검사한다. 오타를 내면 컴파일 자체가 안 된다.

리플렉션은 이 검사를 컴파일 시점이 아니라 실행 시점(런타임)으로 미룬다. 클래스 이름을 문자열로만 들고 있어도, 그 클래스가 어떤 필드와 메서드를 가지는지 실행 중에 알아내고 호출할 수 있다.

```java
Class<?> clazz = Class.forName("com.example.Product");
Object product = clazz.getDeclaredConstructor().newInstance();

Method setName = clazz.getMethod("setName", String.class);
setName.invoke(product, "사과");
```

코드만 보면 번거로워 보이지만, 핵심은 `"com.example.Product"`, `"setName"` 같은 게 전부 문자열이라는 점이다. 이 문자열은 설정 파일에서 읽어올 수도 있고, 어노테이션에서 읽어올 수도 있고, 사용자 입력으로 받을 수도 있다. 즉 어떤 클래스를 다룰지를 코드를 짤 때 미리 정해두지 않고, 실행 중에 결정할 수 있게 된다.

### 리플렉션의 핵심 도구들

리플렉션은 `java.lang.reflect` 패키지와 `Class` 클래스를 중심으로 동작한다.

| 클래스 | 역할 |
|---|---|
| `Class<T>` | 클래스 자체를 나타내는 객체. 모든 리플렉션의 출발점 |
| `Field` | 필드 정보. 값을 읽거나 쓸 수 있다 |
| `Method` | 메서드 정보. 실행할 수 있다 |
| `Constructor` | 생성자 정보. 인스턴스를 만들 수 있다 |
| `Annotation` | 클래스/메서드/필드에 붙은 어노테이션 정보 |

모든 자바 객체는 자신이 속한 클래스에 대한 `Class` 객체를 갖고 있고, 여기서부터 모든 리플렉션 작업이 시작된다.

```java
Class<?> clazz = product.getClass();          // 인스턴스에서 얻기
Class<?> clazz = Product.class;                // 클래스 리터럴에서 얻기
Class<?> clazz = Class.forName("com.example.Product");  // 문자열 이름에서 얻기
```

---

## 2. 왜 리플렉션이 필요한가?

### 2.1 정적 타입 코드로는 안 되는 것들

자바는 정적 타입 언어라서, 어떤 객체를 다룰지 컴파일 시점에 확정해야 한다. 이건 대부분의 상황에서 안전하고 빠르지만, 코드를 짜는 시점에는 존재하지 않는 정보를 다뤄야 하는 상황에서는 한계가 된다.

예를 들어 JSON 문자열을 객체로 변환하는 라이브러리를 만든다고 해보자.

```java
String json = "{\"name\":\"사과\",\"price\":1000}";
Product product = objectMapper.readValue(json, Product.class);
```

이 라이브러리를 만드는 입장에서 생각해보면, `Product`라는 클래스가 어떤 필드를 가지고 있는지 라이브러리 코드를 짤 때는 전혀 알 수 없다. 사용하는 사람이 `Product`를 쓸 수도 있고 `Order`를 쓸 수도 있고, 임의의 클래스 아무거나를 넘길 수도 있다. 그런데도 라이브러리는 그 클래스의 필드에 값을 채워 넣어야 한다.

컴파일 시점에 정해진 코드만으로는 이걸 처리할 방법이 없다. `product.setName(...)` 같은 코드를 라이브러리 안에 미리 써둘 수가 없기 때문이다. 라이브러리가 알고 있는 건 오직 `Class<?> clazz` 하나뿐이고, 여기서부터 필드 목록을 뽑아 하나씩 값을 채워 넣어야 한다.

```java
for (Field field : clazz.getDeclaredFields()) {
    field.setAccessible(true);
    Object value = jsonMap.get(field.getName());
    field.set(product, value);
}
```

이게 리플렉션이 풀어주는 문제다. 코드를 짤 때 몰랐던 타입을, 실행할 때 알아내서 다룰 수 있게 해준다.

### 2.2 리플렉션이 없다면

리플렉션 없이 이런 범용 처리를 하려면 남는 방법은 사실상 하나뿐이다. 모든 가능한 타입에 대해 분기 처리를 직접 써두는 것이다.

```java
if (target instanceof Product p) {
    p.setName(map.get("name"));
    p.setPrice(map.get("price"));
} else if (target instanceof Order o) {
    o.setId(map.get("id"));
    ...
} else if (target instanceof Member m) {
    ...
}
```

이건 라이브러리가 아니다. 사용자가 새로운 클래스를 만들 때마다 라이브러리 코드 자체를 고쳐야 하기 때문이다. 범용 라이브러리, 프레임워크라는 게 성립하려면 개발자가 만들 클래스가 뭔지 미리 알 수 없는 상태에서도 동작해야 하고, 이걸 가능하게 하는 게 리플렉션이다.

### 2.3 대가 — 리플렉션의 비용

리플렉션이 강력한 만큼 대가도 있다.

첫째, 성능이다. 리플렉션으로 메서드를 호출하면 일반 메서드 호출보다 느리다. JIT 컴파일러가 런타임에 최적화할 수 있는 일반 호출과 달리, 리플렉션 호출은 타입 검사와 접근 제어 검사를 실행 중에 매번 거친다.

둘째, 타입 안전성이다. `field.set(product, value)`에서 `value`의 타입이 실제 필드 타입과 맞는지는 컴파일러가 검사해주지 않는다. 잘못된 타입을 넣으면 컴파일은 통과하고 런타임에 `IllegalArgumentException`이 터진다.

셋째, 캡슐화가 깨진다. `setAccessible(true)`를 쓰면 `private` 필드나 메서드에도 접근할 수 있게 되는데, 이건 원래 자바가 접근 제어자로 막아둔 걸 강제로 뚫는 것이다. 라이브러리 코드에서는 필요하지만, 일반 애플리케이션 코드에서 남발하면 캡슐화를 지키는 의미가 없어진다.

그래서 리플렉션은 애플리케이션 로직에서 직접 쓰기보다는, 프레임워크나 라이브러리가 내부적으로 쓰고 개발자는 어노테이션이나 설정으로만 다루는 형태로 감싸서 제공되는 경우가 대부분이다.

---

## 3. 리플렉션은 어디에 사용되는가?

### 3.1 스프링의 의존성 주입(DI)

스프링이 `@Autowired`가 붙은 필드나 생성자를 보고 알아서 객체를 채워 넣는 것도 리플렉션이다.

```java
@Service
public class OrderService {

    @Autowired
    private ProductRepository productRepository;
}
```

스프링 컨테이너 입장에서는 `OrderService`라는 클래스가 어떤 필드를 가지고 있는지 컴파일 시점에는 알 방법이 없다. 컨테이너가 뜰 때 리플렉션으로 `OrderService.class`의 필드 목록을 조회하고, `@Autowired`가 붙은 필드를 찾은 다음, 해당 타입의 빈을 찾아 리플렉션으로 값을 주입한다.

```java
Class<?> clazz = Class.forName("com.example.OrderService");
Object orderService = clazz.getDeclaredConstructor().newInstance();   // 인스턴스 생성도 리플렉션

for (Field field : clazz.getDeclaredFields()) {
    if (field.isAnnotationPresent(Autowired.class)) {
        field.setAccessible(true);
        Object bean = applicationContext.getBean(field.getType());
        field.set(orderService, bean);
    }
}
```

실제 스프링 내부 구현은 이보다 훨씬 복잡하지만, 기본 원리는 이와 같다. 생성자 주입도 마찬가지로 `Constructor` 객체를 리플렉션으로 가져와서 `newInstance()`로 인스턴스를 만든다.

### 3.2 프록시 생성 — JDK 다이나믹 프록시

프록시가 받은 메서드 호출을 원본에 전달할 때 쓰는 게 `Method.invoke()`다.

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    System.out.println("[LOG] " + method.getName() + " 호출");
    return method.invoke(target, args);   // 리플렉션으로 원본 호출
}
```

여기서 `method`가 어떤 메서드인지는 런타임에 결정된다. 프록시를 만드는 코드는 대상 인터페이스가 어떤 메서드를 가졌는지 컴파일 시점에 전혀 몰라도 동작한다. 인터페이스가 뭐든, 메서드가 몇 개든 상관없이 똑같은 `InvocationHandler` 하나로 처리할 수 있는 이유가 리플렉션이다.

### 3.3 어노테이션 읽기 — @Transactional, @Test 같은 것들

어노테이션은 그 자체로 실행되는 코드가 아니라, 컴파일된 클래스 파일 안에 별도로 붙어 있는 메타데이터일 뿐이다. `@Transactional`을 붙인다고 해서 트랜잭션 코드가 메서드 안에 삽입되는 게 아니라, 단지 "이 메서드에 이 어노테이션이 붙어 있다"는 정보가 클래스 파일에 기록될 뿐이라는 뜻이다. 이 메타데이터를 읽어내는 유일한 방법이 리플렉션이다.

```java
Method method = OrderService.class.getMethod("placeOrder", Order.class);

if (method.isAnnotationPresent(Transactional.class)) {
    Transactional tx = method.getAnnotation(Transactional.class);
    System.out.println(tx.propagation());   // 어노테이션의 속성값도 읽을 수 있다
}
```

스프링이 `@Transactional`, `@Cacheable`이 붙은 메서드를 찾아 프록시를 씌울지 말지 판단하는 과정, JUnit이 `@Test`가 붙은 메서드를 찾아 테스트로 실행하는 과정, 모두 이 방식으로 어노테이션을 리플렉션으로 스캔하는 데서 시작한다.

```java
for (Method method : TestClass.class.getMethods()) {
    if (method.isAnnotationPresent(Test.class)) {
        method.invoke(testInstance);   // 테스트 메서드를 리플렉션으로 실행
    }
}
```

### 3.4 JSON 직렬화/역직렬화 — Jackson

Jackson 같은 JSON 라이브러리가 객체를 JSON으로 바꾸거나(직렬화), JSON을 객체로 바꾸는(역직렬화) 과정도 리플렉션이다.

```java
Product product = new Product("사과", 1000);
String json = objectMapper.writeValueAsString(product);
```

`objectMapper`는 `Product`라는 클래스를 컴파일 시점에 전혀 모른다. `writeValueAsString()`이 호출되면 내부에서 `product.getClass()`로 클래스 정보를 얻고, 필드 또는 getter 메서드를 리플렉션으로 순회하며 값을 꺼내 JSON 문자열로 조립한다. 반대로 역직렬화는 리플렉션으로 생성자나 setter를 호출해 필드를 채운다.

### 3.5 테스트 도구 — Mockito

`@Mock`으로 만든 가짜 객체가 실제로 어떤 메서드가 몇 번 호출됐는지 기록하고 검증하는 것도 리플렉션(과 앞서 본 동적 프록시 기술)의 조합이다.

```java
@Mock
private ProductRepository productRepository;

@Test
void 주문_조회시_레포지토리를_호출한다() {
    // 1. 테스트 대상 코드(예: OrderService)가 내부적으로 이 메서드를 호출
    //    → 이 순간 Mockito 프록시(또는 인터셉터)가 호출 사실을 가로채 기록해둔다
    productRepository.findById(1L);

    // 2. 기록해둔 호출 이력을 대조해서 검증
    verify(productRepository).findById(1L);
}
```

Mockito는 `ProductRepository` 인터페이스를 런타임에 분석해서 프록시를 만들고, 그 프록시가 받은 모든 메서드 호출 정보를 가로채 기록해뒀다가(메서드 메타데이터 분석 등) `verify()` 시점에 대조한다.

### 3.6 스프링 데이터 리포지토리

인터페이스만 선언하면 구현체가 없어도 동작하는 스프링 데이터의 리포지토리도 리플렉션으로 인터페이스의 메서드 목록을 읽는 데서 시작한다.

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByNameContaining(String name);
}
```

스프링 데이터는 컨테이너가 뜰 때 이 인터페이스를 리플렉션으로 스캔해서 `findByNameContaining`이라는 메서드 이름을 파싱하고, 거기서 `name` 필드에 대한 `LIKE` 쿼리를 만들어낼 프록시 구현체를 즉석에서 만든다.

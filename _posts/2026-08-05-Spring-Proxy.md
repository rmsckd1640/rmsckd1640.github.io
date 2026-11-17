---
title: "프록시(Proxy) 패턴과 Spring AOP"
date: 2026-08-05 10:57:34 +0900
tags: ["Java", "Spring"]
summary: "프록시(proxy)는 대리인이라는 뜻이다. 어떤 객체를 직접 쓰지 않고, 그 객체와 똑같이 생긴 다른 객체를 대신 쓰는 것을 말한다. 클라이언트 입장에서는 진짜를 쓰는지 대리인을 쓰는지 구별할 수 없어야 한다. 대리인은 요청을 받아 뭔가를 한 다음 진짜에게 넘기거나, 상황에 따라 아예 넘기지 않는다."
---

## 1. 프록시란 무엇인가?

프록시(proxy)는 대리인이라는 뜻이다. 어떤 객체를 직접 쓰지 않고, 그 객체와 똑같이 생긴 다른 객체를 대신 쓰는 것을 말한다.

```text
클라이언트 → 프록시 객체 → 실제(원본) 객체
```

클라이언트 입장에서는 진짜를 쓰는지 대리인을 쓰는지 구별할 수 없어야 한다. 대리인은 요청을 받아 뭔가를 한 다음 진짜에게 넘기거나, 상황에 따라 아예 넘기지 않는다.

프록시가 되려면 세 가지 조건을 만족해야 한다.

- 원본과 같은 타입이다 (같은 인터페이스를 구현하거나, 같은 클래스를 상속한다)
- 내부에 원본에 대한 참조를 들고 있다
- 호출을 받으면 뭔가 하고 나서 원본에게 위임한다 (또는 위임하지 않는다)

**예시**

```java
public interface UserService {
    void save(String name);
}

// 원본
public class UserServiceImpl implements UserService {
    public void save(String name) {
        System.out.println(name + " 저장 완료");
    }
}

// 프록시 — 원본과 같은 인터페이스를 구현하고, 원본을 들고 있다
public class LoggingUserService implements UserService {

    private final UserService target;   // 원본에 대한 참조

    public LoggingUserService(UserService target) {
        this.target = target;
    }

    public void save(String name) {
        System.out.println("[LOG] save 호출 시작");
        target.save(name);              // 원본에게 위임
        System.out.println("[LOG] save 호출 종료");
    }
}
```

```java
UserService service = new LoggingUserService(new UserServiceImpl());
service.save("사용자");
```

`service`의 타입은 여전히 `UserService`다. 쓰는 쪽 코드는 로깅이 끼어들었다는 걸 모르고, 알 필요도 없다. 이 투명성이 프록시의 핵심이다. 특별한 마법이 아니라, 그냥 자바 클래스 하나를 더 만든 것뿐이다.

---

## 2. 왜 프록시가 필요한가?

### 2.1 원본을 직접 고치면?

어떤 조회 메서드가 있고, 결과를 캐싱하고 싶다고 해보자.

```java
public class ProductService {
    public Product findById(Long id) {
        return productRepository.findById(id);
    }
}
```

가장 먼저 떠오르는 방법은 메서드 안에 캐시 코드를 바로 넣는 것이다.

```java
public class ProductService {
    private final Map<Long, Product> cache = new HashMap<>();

    public Product findById(Long id) {
        if (cache.containsKey(id)) {
            return cache.get(id);
        }
        Product result = productRepository.findById(id);
        cache.put(id, result);
        return result;
    }
}
```

동작은 한다. 문제는 상품을 조회한다는 한 줄짜리 관심사에 캐싱이라는 다른 관심사가 섞여 들어갔다는 것이다. 여기에 로깅, 권한 확인, 트랜잭션까지 얹으면 이렇게 된다.

```java
public Product findById(Long id) {
    if (!hasPermission()) throw new AccessDeniedException();

    long start = System.currentTimeMillis();
    TransactionStatus tx = transactionManager.getTransaction(definition);
    try {
        if (cache.containsKey(id)) return cache.get(id);

        Product result = productRepository.findById(id);   // ← 진짜 하려던 일

        cache.put(id, result);
        transactionManager.commit(tx);
        return result;
    } catch (Exception e) {
        transactionManager.rollback(tx);
        throw e;
    } finally {
        log.info("findById took {}ms", System.currentTimeMillis() - start);
    }
}
```

비즈니스 로직 한 줄이 스무 줄에 파묻혔다. 그리고 이 껍데기는 `findAll`, `save`, `delete`에도 똑같이 필요하다. 서비스 클래스가 열 개면 메서드 수십 개에 같은 코드를 복사해야 하고, 트랜잭션 정책 하나 바꾸려면 수십 곳을 전부 고쳐야 한다.

이렇게 여러 클래스를 가로질러 반복되는 부가 기능을 횡단 관심사(cross-cutting concern)라고 부른다. 비즈니스 로직이 아니라 인프라 코드인데, 비즈니스 로직 사이에 끼어 있다는 게 문제다.

### 2.2 프록시로 분리하기

원본은 그대로 두고, 캐싱만 담당하는 클래스를 따로 만들면 어떨까.

```java
public interface ProductService {
    Product findById(Long id);
}

// 원본 — 비즈니스 로직만 남는다
public class ProductServiceImpl implements ProductService {
    public Product findById(Long id) {
        return productRepository.findById(id);
    }
}

// 캐싱 담당 — 원본과 같은 인터페이스를 구현하는 프록시
public class CachingProductService implements ProductService {

    private final ProductService target;
    private final Map<Long, Product> cache = new HashMap<>();

    public CachingProductService(ProductService target) {
        this.target = target;
    }

    public Product findById(Long id) {
        if (cache.containsKey(id)) {
            return cache.get(id);            // 원본을 호출하지 않는다
        }
        Product result = target.findById(id); // 없을 때만 원본 호출
        cache.put(id, result);
        return result;
    }
}
```

원본은 순수하게 조회 로직만 담당하고, 캐싱은 프록시가 알아서 처리한다. 필요하면 여러 겹으로 쌓을 수도 있다.

```java
ProductService service =
    new LoggingProductService(
        new CachingProductService(
            new TransactionalProductService(
                new ProductServiceImpl())));
```

캐싱 프록시가 히트하면 그 안쪽(트랜잭션, 원본)은 아예 실행되지 않는다. 기능을 빼고 싶으면 조립에서 한 줄만 지우면 되고, 원본은 손대지 않아도 된다.

### 2.3 정적 프록시의 한계

지금까지 만든 건 정적 프록시다. 프록시 클래스를 개발자가 직접 손으로 썼다는 뜻이다. 원리를 이해하기엔 좋지만 실무에서 그대로 쓰기엔 문제가 있다.

- 클래스가 폭발한다 — 서비스가 20개인데 각각에 캐싱·로깅·트랜잭션을 붙이려면 프록시 클래스가 60개 필요하다.

- 같은 코드를 반복한다 — `CachingProductService`와 `CachingOrderService`의 캐싱 로직은 사실상 같은데 타입만 다르다.

- 인터페이스의 모든 메서드를 구현해야 한다 — 메서드가 15개인데 캐싱하고 싶은 게 하나뿐이어도, 나머지 14개를 그냥 `target`에 넘기는 코드로 채워야 한다.

- 원본이 바뀌면 프록시도 바뀐다 — 인터페이스에 메서드 하나 추가하면 프록시 클래스가 전부 컴파일 에러를 낸다.

문제의 핵심은 프록시 클래스를 사람이 쓴다는 것이다. 그러면 프록시 클래스를 사람이 아니라 프로그램이 런타임에 만들면 어떨까. 이게 동적 프록시다.

자바는 `java.lang.reflect.Proxy`로 인터페이스를 구현한 클래스를 즉석에서 생성하거나, CGLIB 같은 라이브러리로 바이트코드를 직접 조립할 수 있다.

동적 프록시를 쓰면 위 네 가지 문제가 한꺼번에 사라진다. 부가 기능 로직은 한 곳에만 작성하면 어떤 타입에도 적용할 수 있고, 관심 없는 메서드는 자동으로 원본에 위임되며, 인터페이스가 바뀌어도 코드를 고칠 필요가 없다.

스프링이 `@Transactional`이나 `@Cacheable` 어노테이션 하나로 부가 기능을 붙여주는 것이 바로 이 동적 프록시 기술을 활용한 것이다. 이제 스프링이 구체적으로 어떻게 쓰는지 본다.

---

## 3. 스프링의 프록시 사용법

### 3.1 AOP — 프록시를 쓰는 이유

스프링은 트랜잭션 처리, 로깅, 보안 체크 같은 횡단 관심사를 비즈니스 로직에서 분리하기 위해 AOP(Aspect-Oriented Programming)라는 개념을 도입했고, 이를 구현하는 방법으로 런타임 프록시를 선택했다.

`@Transactional`을 붙이면 트랜잭션이 자동으로 걸리는 것도, 스프링이 해당 빈을 프록시로 감싸서 메서드 호출 전에 트랜잭션 시작, 메서드 실행, 성공 시 커밋, 실패 시 롤백이라는 코드를 자동으로 끼워 넣어주기 때문이다.

참고로 AOP를 구현하는 방법이 프록시만 있는 건 아니다. AspectJ라는 별도 프레임워크는 컴파일 시점이나 클래스 로딩 시점에 바이트코드 자체를 직접 고치는 위빙(weaving) 방식을 쓴다.

| 방식 | 코드가 끼어드는 시점 | 스프링에서 |
|---|---|---|
| 런타임 프록시 | 애플리케이션 실행 중, 감싸는 객체가 가로챔 | Spring AOP (기본값) |
| 컴파일타임 위빙 | 컴파일 시점에 바이트코드에 직접 삽입 | AspectJ (별도 설정 필요) |
| 로드타임 위빙 | 클래스 로딩 시점에 바이트코드 변경 | AspectJ (별도 설정 필요) |

스프링이 런타임 프록시를 기본으로 택했기 때문에, 뒤에서 다룰 자기 자신 호출 문제 같은 특유의 함정이 생긴다. 이건 AOP 자체의 한계가 아니라 프록시 방식을 택한 것에 대한 대가다.

### 3.2 AOP 용어 정리

뒤에서 계속 쓰이는 용어라 먼저 정리한다.

| 용어 | 의미 |
|---|---|
| Join Point | 부가 기능을 끼워 넣을 수 있는 지점. Spring AOP에서는 메서드 실행뿐이다 |
| Pointcut | Join Point 중 어디에 적용할지 고르는 조건. (예: `@Transactional`이 붙은 메서드) |
| Advice | 그 지점에서 무엇을 할지. 실제 부가 로직 |
| Advisor | Pointcut + Advice 한 쌍. Spring AOP의 기본 단위 |
| Aspect | 여러 Advisor를 묶은 모듈. `@Aspect` 클래스가 이것 |

트랜잭션 Aspect는 `@Transactional`이 붙은 메서드(Pointcut)에서 트랜잭션을 열고 닫는다(Advice), 이런 식으로 읽으면 된다.

### 3.3 프록시는 언제 만들어지는가 — 빈 생성 과정

스프링 빈이 만들어지는 과정을 보면 프록시가 정확히 언제 개입하는지 알 수 있다.

```text
① BeanDefinition 읽기        OrderService 빈이 필요하다
② 인스턴스 생성               new OrderService(repository)   ← 원본 객체 생성
③ 의존성 주입                 필드/생성자 주입 완료
④ 초기화                      @PostConstruct 실행
⑤ BeanPostProcessor        	← 프록시 객체 생성
⑥ 컨테이너에 등록              ⑤가 반환한 객체가 최종 빈
```

②에서 원본 객체가 반드시 먼저 만들어진다. 프록시는 원본을 대체하는 게 아니라 감싸는 것이므로, 감쌀 대상이 있어야 한다.

⑤의 `BeanPostProcessor`는 빈 생성 후 다른 객체로 바꿔치기할 수 있는 확장점이다.

```java
public interface BeanPostProcessor {
    Object postProcessAfterInitialization(Object bean, String beanName);
    //     ^^^^^^ 여기서 반환한 객체가 최종 빈이 된다
}
```

Spring AOP는 이 지점에서 대략 이렇게 동작한다.

```java
public Object postProcessAfterInitialization(Object bean, String beanName) {
    Advisor[] advisors = findEligibleAdvisors(bean.getClass());

    if (advisors.isEmpty()) {
        return bean;                      // 적용할 Advisor가 없으면 원본 그대로
    }
    return createProxy(bean, advisors);   // 있으면 프록시로 감싸서 반환
}
```

즉 어노테이션이 하나도 없는 빈은 프록시가 아예 만들어지지 않는다.

어노테이션 자체는 실행되는 코드가 아니다. 컴파일된 바이트코드를 보면 `@Transactional` 같은 건 `RuntimeVisibleAnnotations`라는 별도 메타데이터 영역에만 기록되고, 실행 명령어 안에는 없다.

스프링은 컨테이너가 뜰 때 리플렉션으로 이 메타데이터를 한 번 읽고, 그걸 근거로 프록시를 만든다. 그 이후로는 어노테이션을 다시 읽지 않고, 프록시에 이미 심어진 코드가 실행될 뿐이다.

### 3.4 스프링이 프록시를 만드는 두 가지 방식

2.3에서 언급한 동적 프록시 기술을 스프링은 상황에 따라 둘 중 하나로 쓴다.

#### JDK 다이나믹 프록시

- 타겟 클래스가 인터페이스를 구현하고 있을 때 사용

- `java.lang.reflect.Proxy`가 인터페이스를 구현한 클래스를 런타임에 생성

- 원본 호출은 리플렉션(`Method.invoke()`)으로 수행

```java
UserService target = new UserServiceImpl();
UserService proxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class[]{ UserService.class },
    (p, method, args) -> {
        System.out.println("[LOG] " + method.getName() + " 호출");
        return method.invoke(target, args);   // 리플렉션으로 원본 호출
    }
);
```

한계는 두 가지다. 인터페이스가 없는 클래스는 프록시로 만들 수 없고, 반환되는 프록시의 실제 타입은 `$Proxy0` 같은 것이라 구현 클래스 타입으로 캐스팅하면 `ClassCastException`이 난다.

```java
UserService proxy = ...;                          // OK
UserServiceImpl impl = (UserServiceImpl) proxy;    // ClassCastException
```

#### CGLIB

- 타겟 클래스가 인터페이스 없이 구체 클래스만 있을 때 사용

- 타겟 클래스를 상속한 하위 클래스를 바이트코드 조작으로 동적 생성

- 원본 호출은 `Method.invoke()` 대신 FastClass(메서드마다 인덱스를 매겨 `switch`로 직접 호출)를 써서 더 빠름

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(UserServiceImpl.class);
enhancer.setCallback((MethodInterceptor) (obj, method, args, methodProxy) -> {
    System.out.println("[LOG] " + method.getName() + " 호출");
    return methodProxy.invokeSuper(obj, args);   // 부모(원본) 메서드 호출
});
UserServiceImpl proxy = (UserServiceImpl) enhancer.create();
```

`final` 클래스나 `final`/`private`/`static` 메서드는 상속·오버라이드가 불가능하므로 프록시가 걸리지 않는다.

또한 상속 기반이라 원래는 부모 생성자를 호출해야 하는데, 스프링은 Objenesis 라이브러리로 생성자 호출 없이 프록시 인스턴스를 만든다.

그래서 프록시 자신의 필드는 비어 있고, 실제 상태는 프록시가 감싸고 있는 원본 인스턴스가 따로 가지고 있다.

#### 비교

| 구분 | JDK 다이나믹 프록시 | CGLIB |
|---|---|---|
| 전제 조건 | 인터페이스 필요 | 인터페이스 불필요 |
| 구현 방식 | 인터페이스 구현 | 클래스 상속 |
| 내부 기술 | 리플렉션 | 바이트코드 조작 |
| 구현체 타입 캐스팅 | `ClassCastException` | 가능 |
| 제약 | 인터페이스에 없는 메서드는 프록시 불가 | `final`/`private`/`static` 프록시 불가 |
| 스프링 부트 기본값 | 미사용 (설정 시 가능) | 기본 전략 (2.0 이후) |

인터페이스가 있어도 스프링 부트가 CGLIB을 기본값으로 쓰는 이유가 여기 있다. 인터페이스 없는 클래스도 다룰 수 있고, 구현체 타입으로 주입받아도 문제없고, 성능도 더 낫기 때문이다.

### 3.5 호출이 들어오면 — 인터셉터 체인

프록시가 만들어진 뒤 메서드를 호출하면 내부적으로 인터셉터 체인을 통과한다.

```text
프록시.placeOrder(order)
      ↓
  proceed()
      ├─ [0] CacheInterceptor 같은 게 있다면
      │        캐시 확인 → 있으면 여기서 return (더 아래는 실행 안 됨)
      ├─ [1] TransactionInterceptor
      │        트랜잭션 시작 → proceed() 재귀 호출 → 커밋/롤백
      └─ 체인 끝 → target.placeOrder(order) 실제 원본 호출
```

앞단의 인터셉터가 `proceed()`를 안 부르면 그 뒤 체인 전체(트랜잭션, 원본 호출 포함)가 실행되지 않는다. 캐시 히트 시 트랜잭션조차 열리지 않고 끝나는 이유다.

### 3.6 자기 자신 호출(self-invocation) — 가장 유명한 함정

```java
@Service
public class OrderService {

    @Transactional
    public void update(Order order) { ... }

    public void updateAll(List<Order> orders) {
        for (Order o : orders) {
            update(o);          // ← 트랜잭션이 걸리지 않는다
        }
    }
}
```

`update()`에 `@Transactional`이 붙어 있지만, `updateAll()`을 통해 호출하면 트랜잭션이 열리지 않는다.

자바에서 객체 없이 메서드만 호출하는 건 없다. `update(o)`는 컴파일러가 `this.update(o)`로 취급한다. 호출 흐름을 따라가 보면:

```text
[1] 외부 코드: orderService.updateAll(list)
    └ orderService 변수에 담긴 건 프록시
    
[2] 프록시.updateAll(list)
    └ updateAll엔 @Transactional이 없음 → 그냥 target.updateAll(list) 호출
    
[3] 원본.updateAll(list)          ◀ 실행 위치가 원본 객체 안으로 들어옴
    		이 순간부터 this == 원본 객체
    └ update(o) = this.update(o) = 원본.update(o)
    
[4] 원본.update(o)
    └ 본문에 트랜잭션 코드가 없음 (프록시에만 있으므로)
```

[2]에서 프록시는 이미 자기 역할을 마쳤다. [3]부터는 실행 위치가 원본 객체 안이고, `this`는 원본이다. 그리고 원본은 자신을 감싼 프록시로 돌아갈 참조를 갖고 있지 않다. 참조는 `프록시 → 원본` 한 방향뿐이다.

```text
프록시 인스턴스
   target ──────▶  원본 인스턴스
                        ↑
                 여기서 프록시로 돌아갈 길이 없다
```

CGLIB 프록시는 원본을 상속한 서브클래스인데, 그럼 다형성으로 오버라이드된 쪽이 호출돼야 하지 않나 싶을 수 있다. 하지만 스프링은 프록시 인스턴스와 원본 인스턴스를 별개로 두 개 만든다.

`target`이 가리키는 건 순수한 원본 인스턴스지 서브클래스가 아니므로, 그 안에서 오버라이드할 대상 자체가 없다.

#### 해결 방법

| 방법 | 설명 |
|---|---|
| 다른 빈으로 분리 (권장) | `update()`를 호출하는 로직을 별도 빈으로 옮겨서, 주입받은 프록시를 통해 호출하게 함 |
| 자기 자신을 주입받기 | `@Autowired @Lazy` 필드로 자기 자신(프록시)을 주입받아 그걸 통해 호출 |
| `AopContext.currentProxy()` | `exposeProxy = true` 설정 후 현재 프록시를 코드에서 직접 꺼냄 |
| AspectJ 위빙으로 전환 | 프록시 대신 바이트코드 자체를 고치는 방식이라 자기호출도 잡히지만, 빌드 설정이 무거워짐 |

실무에서 특히 자주 문제 되는 경우는 트랜잭션 전파 속성을 다르게 주고 싶을 때다.

```java
@Transactional
public void 주문처리() {
    ...
    이력남기기();   // 메인이 롤백돼도 이력은 남기를 기대
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void 이력남기기() { ... }
```

같은 클래스 안에서 호출하면 `REQUIRES_NEW`가 무시되고 별도 트랜잭션이 열리지 않아, 메인 트랜잭션과 함께 롤백돼버린다. 장애가 났는데 로그가 하나도 안 남았다는 상황이 여기서 나온다.

### 3.7 프록시가 아예 안 걸리는 경우

CGLIB는 상속 후 오버라이드하는 방식이므로, 오버라이드가 불가능한 대상은 프록시가 적용되지 않는다.

| 대상 | 이유 | 증상 |
|---|---|---|
| `private` 메서드 | 오버라이드 불가 | 조용히 무시 |
| `final` 메서드 | 오버라이드 불가 | 조용히 무시 |
| `final` 클래스 | 상속 불가 | 빈 생성 시 예외 |
| `static` 메서드 | 인스턴스 메서드가 아님 | 조용히 무시 |
| `@PostConstruct` 안에서 호출 | 프록시가 아직 안 만들어짐 (④ 시점 < ⑤ 시점) | 조용히 무시 |

대부분 에러 없이 조용히 무시된다는 점이 실무에서 가장 위험하다. `private` 메서드에 `@Transactional`을 붙이면 컴파일도 되고 실행도 되지만 트랜잭션만 안 걸린다.

테스트에서도 마찬가지다. `@Mock` + `@InjectMocks`처럼 Spring 컨테이너 없이 `new`로 만든 객체에는 프록시가 없다. 트랜잭션이나 캐시 동작을 검증하려면 `@SpringBootTest`로 실제 컨테이너를 띄워야 한다.

Spring in Action Chapter 4 - Aspect-oriented Spring
========================

Define AOP 
---------

AOP에는 Advice, Pointcut, Join points가 존재한다.

Advice - The job of an aspect, 쉽게 설명하자면 Advice란 aspect가 수행할 작업을 말한다.

Joint point - Aspect가 들어갈 수 있는 위치

Pointcut - AspectJ's pointcut expression language를 사용하여 Advice에 실행될 곳(Joint Point)를 설정하는 것

Java config
---

```Java
execution(* concert.Performance.perform(..))
            && within(concert.*))
```

execution은 pointcut의 기본 실행 명령어이고, *는 모든 리턴타입을 수용한다는 것이다. 그리고 concert.Performnce class의 perform을 실행하겠다는 것이고 (..)은 모든 파라미터를 수용한다는 것이다. &&는 and를 나타내며, within은 conert class안의 클래스에서만 이 메소드가 실행시키겠다는 것을 나타낸다.

```Java
@Aspect
public class Audience {
    @Before("execution(** concert.Performance.perform(..))")
    public void silenceCellPhones() {
        System.out.println("Silencing cell phones");
    }
    @Before("execution(** concert.Performance.perform(..))")
    public void takeSeats() {
    System.out.println("Taking seats");
    }
    @AfterReturning("execution(** concert.Performance.perform(..))")
    public void applause() {
    System.out.println("CLAP CLAP CLAP!!!");
    }
    @AfterThrowing("execution(** concert.Performance.perform(..))")
    public void demandRefund() {
    System.out.println("Demanding a refund");
    }
}
```

실제 자바에서는 위와 같이 사용한다.

```Java
@Aspect
public class Audience {
    @Pointcut("execution(** concert.Performance.perform(..))")
    public void performance() {}
    @Before("performance()")
    public void silenceCellPhones() {
        System.out.println("Silencing cell phones");
    }
    @Before("performance()")
    public void takeSeats() {
        System.out.println("Taking seats");
    }
    @AfterReturning("performance()")
    public void applause() {
        System.out.println("CLAP CLAP CLAP!!!");
    } 
    @AfterThrowing("performance()")
    public void demandRefund() {
        System.out.println("Demanding a refund");
    }
}
```

위와 같이 Pointcut을 변수 같이 선언하여 ID를 참조만 해서 사용할 수도 있다.

```Java
@Configuration
@EnableAspectJAutoProxy
@ComponentScan
public class ConcertConfig {
    @Bean
    public Audience audience() {
        return new Audience();
    }
}
```

위와 같이 @EnableAspectJAutoProxy를 사용하여 @AspectJ로 선언 된 빈들을 proxy-based aspect로 만든다. 기본적으로, 스프링은 proxy-based aspect를 사용한다. proxy-based apsect는 메소드에서만 advice를 삽입할 수 있으므로, 생성자나 멤버변수 레벨에서 사용하고 싶으면 AspectJ aspects를 사용해야 된다.

```Java
@Aspect
public class Audience {
    @Pointcut("execution(** concert.Performance.perform(..))")
    public void performance() {}
    @Around("performance()")
    public void watchPerformance(ProceedingJoinPoint jp) {
        try {
            System.out.println("Silencing cell phones");
            System.out.println("Taking seats");
            jp.proceed();
            System.out.println("CLAP CLAP CLAP!!!");
        } catch (Throwable e) {
            System.out.println("Demanding a refund");
        }
    }   
}
```

위와 같이 Around를 사용해서 4개의 함수들은 한 곳에 다 넣을 수도 있다. ProceedingjoinPoint의 proceed()메소드를 통해 원하는 곳에 실제 실행해야 될 메소드를 실행시킬 수 있다.

```Java
@Aspect
public class TrackCounter {
    private Map<Integer, Integer> trackCounts =
    new HashMap<Integer, Integer>();
    @Pointcut(
    "execution(* soundsystem.CompactDisc.playTrack(int)) " +
    "&& args(trackNumber)")
    public void trackPlayed(int trackNumber) {}
    @Before("trackPlayed(trackNumber)")
    public void countTrack(int trackNumber) {
        int currentCount = getPlayCount(trackNumber);
        trackCounts.put(trackNumber, currentCount + 1);
    }
    public int getPlayCount(int trackNumber) {
        return trackCounts.containsKey(trackNumber)
                ? trackCounts.get(trackNumber) : 0;
    }
}
```

위와 같이 실제 메소드에 실행될 때 받는 파라미터를 이용하고 싶으면 포인트컷에 파라미터를 선언하고 args()를 사용하면 된다. 그리고 args()쓰인 파라미터 명이랑 실제 pointcut이 애노테이션된 함수에 쓰인 파라미터명을 같게 만들면 된다.

Annotating introductions
---

```Java
package concert;

public interface Encoreable {
    void performEncore();
}
```

```Java
@Aspect
public class EncoreableIntroducer {
    @DeclareParents(value="concert.Performance+",
                    defaultImpl=DefaultEncoreable.class)
    public static Encoreable encoreable;
}
```

위와 같이 Encoreable 인터페이스를 만들고 아래에서 Encoreable를 선언한다. 이렇게 하면 Performance 타입의 빈은 Encoreable을 implements를 받은것 처럼 되어서 performEncore()를 실행할 수 있다. 그리고 실제 구현은 defaultImpl에 설정한 DefaultEncoreable.class에 되어 있다. 이와 같이 introduction을 사용하면 코드를 변환하지 않고도 인터페이스를 상속받은 것처럼해서 다른 클래스들에서 인터페이스 메소드를 사용할 수 있다.

Declaring aspects in XML
---
사실은 XML은 Java config와 별 다른것은 없고 Java config와 소스를 갖고 있어야 애노테이션을 할 수 있는데, XML은 애노테이션을 할 필요가 없어서 소스코드가 없는 라이브러리도 aspect로 선언할 수 있다는 장점이 있다.

Injecting AspectJ aspects
---
스프링 AOP와 달리 AspectJ의 aspects를 이용하면 메소드 단계뿐만 아닌 생서자 단계에서도 Aop를 사용할 수 있다. 그리고 AspectJ에서 스프링의 DI를 이용할 수도 있다.

```Java
public aspect CriticAspect {
    public CriticAspect() {}
    
    pointcut performance() : execution(* perform(..));
    
    afterReturning() : performance() {
        System.out.println(criticismEngine.getCriticism());
    }
    private CriticismEngine criticismEngine;

    public void setCriticismEngine(CriticismEngine criticismEngine) {
        this.criticismEngine = criticismEngine;
    }
}
```

```Java
public class CriticismEngineImpl implements CriticismEngine {
    public CriticismEngineImpl() {}
    public String getCriticism() {
        int i = (int) (Math.random() * criticismPool.length);
        return criticismPool[i];
    }
    // injected
    private String[] criticismPool;
    public void setCriticismPool(String[] criticismPool) {
        this.criticismPool = criticismPool;
    }
}
```

```XML
<bean id="criticismEngine"
        class="com.springinaction.springidol.CriticismEngineImpl">
    <property name="criticisms">
        <list>
            <value>Worst performance ever!</value>
            <value>I laughed, I cried, then I realized I was at the
                    wrong show.</value>
            <value>A must see show!</value>
        </list>
    </property>
</bean>
```

```XML
<bean class="com.springinaction.springidol.CriticAspect"
    factory-method="aspectOf">
    <property name="criticismEngine" ref="criticismEngine" />
</bean>
```

AspectJ는 AspectJ 런타임일때 객체가 생성되므로, 스프링에서 빈으로 등록할때는 factory-method="aspectOf"를 이용해야 된다. AspectJ에서는 aspectOf()메소드를 지원하는데 이것은 ApsectJ 객체의 싱글톤 인스턴스를 제공해준다. 그리서 이와 같이 Bean을 등록할때는 factory-method를 사용하여 빈을 직접 생서하는 것이 아니로 aspectOf()에서 객체의 레퍼런스를 가지고 와서 사용해야 된다. 그리고 그 다음은 spring에서 일반적으로 사용하듯이 property 애트리뷰트를 사용해서 criticismEngine을 DI해주면 된다.




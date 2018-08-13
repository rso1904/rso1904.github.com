Spring in Action Chapter 3 - Advanced wiring
========================

Profile
---------

@Profile("dev")</br>
@Profile("prod")</br>
이와 같은 method에서 Profile 애노테이션을 설정하여 각 환경설정에 맞게 빈을 생성되도록 할 수 있다.

~~~Java
@Configuration
public class DataSourceConfig {

    @Bean(destroyMethod="shutdown")
    @Profile("dev")
    public DataSource embeddedDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("classpath:schema.sql")
            .addScript("classpath:test-data.sql")
            .build();
    }

    @Bean
    @Profile("prod")
    public DataSource jndiDataSource() {
        JndiObjectFactoryBean jndiObjectFactoryBean =
        new JndiObjectFactoryBean();
        
        jndiObjectFactoryBean.setJndiName("jdbc/myDS");
        jndiObjectFactoryBean.setResourceRef(true);
        jndiObjectFactoryBean.setProxyInterface(javax.sql.DataSource.class);
        
        return (DataSource) jndiObjectFactoryBean.getObject();
    }
}
~~~

이와 같이 메소드 범위에서도 @Profile을 선언할 수 있으며 하나의 Config Class로 묶을 수 있다.

XML에서는 `<beans>`에 profile Attribute를 선언할 수 있다.</br>
`<beans profile=''>`

~~~XML
<context-param>
    <param-name>spring.profiles.default</param-name>
    <param-value>dev</param-value>
</context-param>

<servlet>
    <servlet-name>appServlet</servlet-name>
    <servlet-class>
        org.springframework.web.servlet.DispatcherServlet
    </servlet-class>
    <init-param>
        <param-name>spring.profiles.default</param-name>
        <param-value>dev</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
~~~

이와 같이 servelt과 context의 param으로써 spring.profiles.default를 설정할 수도 있다.

테스트를 할 때에는 @ActiveProfiles를 사용하여 원하는 profile의 빈을 생성한 후에 테스트를 실행할 수 있다.

Conditional Beans
---------
스프링4 부터는 @Conditional이 추가되었는데 이것은 빈 메소드에 적용하여 필요할때만 빈을 생성할 수 있다.

```Java
@Bean
@Conditional(MagicExistsCondition.class)
    public MagicBean magicBean() {
        return new MagicBean();
}
```

이와 같이 @Conditional을 선언한후 ~ExisitsCondition.class를 통해 true를 반환하면 빈은 생성되고 false는 그냥 무시하고 넘어간다.

```Java
public class MagicExistsCondition implements Condition {
    public boolean matches(
    ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment env = context.getEnvironment();
        return env.containsProperty("magic");
    }
}
```
@Conditional에 쓰이는 클래스는 Condition을 implements를 해야 된다.
위 클래스는 간단하게 enviroments에서 간단하게 magic 프로퍼티가 있으면 true를 반환하는 클래스이다.
그래서 magic 프로퍼티가 있으면 magicBean() 빈은 생성되게 된다.

```Java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@Conditional(ProfileCondition.class)
    public @interface Profile {
        String[] value();
    }
```

@Profile도 스프링4에서부터는 @Conditional을 사용해서 다시 리팩토링되었다.

```Java
class ProfileCondition implements Condition {

    public boolean matches(
    ConditionContext context, AnnotatedTypeMetadata metadata) {
        if (context.getEnvironment() != null) {
            MultiValueMap<String, Object> attrs =
            metadata.getAllAnnotationAttributes(Profile.class.getName());
            if (attrs != null) {
                for (Object value : attrs.get("value")) {
                    if (context.getEnvironment()
                        .acceptsProfiles(((String[]) value))) {
                        return true;
                    }
                }            
            return false;
            }
        }    
    return true;   
    }
}
```
ProfileCondition 클래스는 위와 같이 선언되어 있다.


Addressing ambiguity in autowiring
---

```Java
@Bean
@Primary
public Dessert iceCream() {
    return new IceCream();
}
```
이런식으로 @Primary를 사용해서 @Autowired를 할 때 모호함을 없앨 수 있지만 @Primary가 2개 이상이 될 때에는 여전히 모호함이 없어지지 않는다.

```XML
<bean id="iceCream" 
    class="com.desserteater.IceCream"
    primary="true" />
```

그리고 또 하나의 방법으로는 @Qualifier가 있다. @Qualifier를 바로 선언해서 사용할 수도 있지만 나 자신의 Qulifier를 주는 게 더 좋다.

```Java
@Component
@Qualifier("cold")
public class IceCream implements Dessert { ... }
```
```Java
@Autowired
@Qualifier("cold")
public void setDessert(Dessert dessert) {
    this.dessert = dessert;
}
```

```Java
@Bean
@Qualifier("cold")
public Dessert iceCream() {
    return new IceCream();
}
```
그러나 여기에서도 모호함이 나타날 수 있으며, @Qulifier를 2개를 선언해서 모호함을 좁혀나갈 수 있다고 생각할 수 있지만, 자바에서는 Multiple 애노테이션을 지원하지 않는다.

그래서 우리는 우리만의 커스텀 애노테이션을 만들어서 모호함을 없앨 수 있다.
```Java
@Target({ElementType.CONSTRUCTOR, ElementType.FIELD,
ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Cold { }
```
```Java
@Target({ElementType.CONSTRUCTOR, ElementType.FIELD,
ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Creamy { }
```
```Java
@Component
@Cold
@Creamy
public class IceCream implements Dessert { ... }
```
```Java
@Component
@Cold
@Fruity
public class Popsicle implements Dessert { ... }
```
```Java
@Autowired
@Cold
@Creamy
public void setDessert(Dessert dessert) {
    this.dessert = dessert;
}
```

Scope Beans
---

스프링은 기본적으로 싱글톤으로 빈을 생성하지만, 사용자는 필요한 경우 빈의 생성방식을 설정할 수있다.

- Singleton—One instance of the bean is created for the entire application.
- Prototype—One instance of the bean is created every time the bean is injected
into or retrieved from the Spring application context.
- Session—In a web application, one instance of the bean is created for each session.
- Request—In a web application, one instance of the bean is created for each
request.

```Java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class Notepad { ... }
```

```Java
@Bean
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public Notepad notepad() {
    return new Notepad();
}
```

```Java
<bean id="notepad"
    class="com.myapp.Notepad"
    scope="prototype" />
```

위와 같이 @Scope를 사용하여 생성방식을 설정할 수 있다.

Request and Session Scope에서는 아래와 같이 사용한다.
```Java
@Component
@Scope(
value=WebApplicationContext.SCOPE_SESSION,
proxyMode=ScopedProxyMode.INTERFACES)
public ShoppingCart cart() { ... }
```

Session과 Request는 유저가 접속하기전에는 빈이 생성되지 않기 때문에 이것은 프록시를 사용하여 StoreService에 ShoppingCart대신에 프록시를 Inject한다. 그리고 프록시가 나중에 lazily resolve를 한다.
그리고 여기에서 ShoppingCart가 인터페이스인것이 가장 좋은 것이지만, ShoppingCart가 클래스라면 GGLib를 사용하여 클래스 기반의 프록시를 만든다.

Runtime value injection
---
런타임 값 인젝션은 아래와 같이 2가지 방법이 있다.

- Property placeholders
- The Spring Expression Language ( S p EL )

먼저 Property placeholders에 대해서 알아보자.

```Java
@Configuration
@PropertySource("classpath:/com/soundsystem/app.properties")
public class ExpressiveConfig {
    @Autowired
    Environment env;

    @Bean
    public BlankDisc disc() {
        return new BlankDisc(
            env.getProperty("disc.title"),
            env.getProperty("disc.artist"));
    }
}
```
위와 같이 하드코딩으로 title과 artist를 넣는 것이 아니라 app.properties에서 정보를 가져와서 런타임에 정보를 넣어줄 수가 있다.

```Java
Class<CompactDisc> cdClass =
 env.getPropertyAsClass("disc.class", CompactDisc.class);
```

위와 같이 Class로써도 프로퍼티를 받을 수 있다.

```Java
public BlankDisc(
@Value("${disc.title}") String title,
@Value("${disc.artist}") String artist) {
    this.title = title;
    this.artist = artist;
}
```

위와 같이 플레이스홀더를 이용해서 바로 값을 주입할 수도 있다.

```Java
@Bean
public
static PropertySourcesPlaceholderConfigurer placeholderConfigurer() {
    return new PropertySourcesPlaceholderConfigurer();
}
```
Placeholder value를 사용할려면 위와 같이 PropertySourcesPlaceholderConfigurer나 PropertyPlaceholder-
Configurer의 빈을 생성해줘야 된다.

그리고 SpEL(Spring Expression Language)를 사용하는 방법이 있다.

- The ability to reference beans by their IDs
- Invoking methods and accessing properties on objects
- Mathematical, relational, and logical operations on values
- Regular expression matching
- Collection manipulation

SpEL는 #{...}형태로 사용한다.

#{T(System).currentTimeMillis()} 는 java.lang.System의 currentTimeMlillis()메소드를 사용하는 것이다.</br>
T()연산자는 java.lang.System을 평가한다. 그리고 T() operator는 java.lang.System의 Class Object를 나타낸다.

#{sgtPeppers.artist}를 통해 sgtPeppers 빈의 artist 프로터리를 불러올 수도 있으며, #{systemProperties['disc.title']}를 통해 시스템 프로퍼티인 dist.title을 불러 올수 도 있다.

```Java
public BlankDisc(
    @Value("#{systemProperties['disc.title']}") String title,
    @Value("#{systemProperties['disc.artist']}") String artist) {
    this.title = title;
    this.artist = artist;
}
```
그리고 Placeholder와 같이 @Value를 통해서도 값을 주입할 수 있다.

XML도 아래와 같이 사용한다.
```XML
<bean id="sgtPeppers"
    class="soundsystem.BlankDisc"
    c:_title="#{systemProperties['disc.title']}"
    c:_artist="#{systemProperties['disc.artist']}" />
```

SpEL은 LITERAL VALUES로 사용할 수 있으며, #{9.87E4}와 같이 scientific notation도 사용 가능하다.

#{artistSelector.selectArtist()}와 같이 artistSelector의 selectArtist()의 메소드를 호출할 수도 있으며, #{artistSelector.selectArtist().toUpperCase()와 같이 selectArtist()의 return type이 String이면 toUpperCase()를 호출하여 대문자로 출력할 수도 있다.

#{artistSelector.selectArtist()?.toUpperCase()}와 같이 ?.를 통하여 null값인지 체크할수도 있다.

그리고 #{2 * T(java.lang.Math).PI * circle.radius}와 같이 Arithmetic 연산자는 물론, Comparison, Logical, Conditional, Regular expression를 사용할 수 있다.

#{scoreboard.score > 1000 ? "Winner!" : "Loser"}와 같이 Java's ternary operator도 사용할 수 있다.

#{admin.email matches '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.com'}는 Regular expression의 예시이다.

#{jukebox.songs[4].title}와 같이 collection and arrays들을 다룰 수도 있다.

#{jukebox.songs.?[artist eq 'Aerosmith']}와 같이 .?[]를 이용하여 collection의 필터된 subset을 구할수 도 있다. .^[]를 통하여 처음 매칭된 엔트리도 구할 수 있다. 물론 .$[]를 통하여 마지막에 매칭된 엔트리도 구할 수 있다.

#{jukebox.songs.![title]}와 같이 projection operator를 통해 the song object가 아닌 a collection of all the song titles를 원할 때 projection operator를 사용할 수 있다.








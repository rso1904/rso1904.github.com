Spring in Action Chapter 5 - Building Spring web applications
========================

Spring MVC
---------

Spring MVC에는 DispatcherServlet, Model and logical view name, Controller, Handler mapping, ViewResolver, View가 존재한다. 각각의 구성요소는 각각의 역할들이 있다.<br/>
 1) DispatcherServlet이 모든 요청을 받는다.<br/>
 2) 받은 요청들을 핸드러 매핑에 던져줘서 이 요청에 알맞는 컨트롤러를 알아낸다.<br/>
 3) 알아낸 컨트롤러에 해당 요청을 던진다. <br/>
 4) 컨트롤러에서 해당 로직을 수행한후 모델과 뷰이름을 다시 DispathcerServlet에 던져준다.<br/>
 5) 뷰 리졸버에게 뷰이름을 던져줘서 실제 뷰의 위치를 얻어온다.<br/>
 6) 실제 뷰에 모델를 던져줘서 렌더링으르 하고 렌더링한 파일은 클라이언트에게 던져준다.<br/>

```Java
public class SpittrWebAppInitializer
extends AbstractAnnotationConfigDispatcherServletInitializer {
    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RootConfig.class };
    }
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { WebConfig.class };
    }
}
```

원래 예전에는 web.xml로 DispatcherServelt를 설정했지만, 지금은 Servlet3과 Spring 3.1 덕분에 java configure로도 가능해졌다. 위의 코드가 그 예시이다. 위와 같이 AbstractAnnotationConfigDispatcherServletInitializer를 extend 받은 후에 ServletMappings과 ConfigClasses들을 설정해주어야 한다.

Servlet 3.0 환경에서는 javax.servlet
.ServletContainerInitializer 인터페이스를 구현한 클래스들을 찾는데, 그것이 스프링에서는 Web-ApplicationInitializer이다. 그리고 Web-ApplicationInitializer를 구현한 것이 AbstractAnnotationConfigDispatcherServletInitializer인데 이와 같은 이유로 우리는 서블릿은 자바에서 configure를 할려면 이와 같이 AbstractAnnotationConfigDispatcherServletInitializer 상속받아야 한다.

```Java
@Configuration
@EnableWebMvc
@ComponentScan("spitter.web")
public class WebConfig
extends WebMvcConfigurerAdapter {
    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver resolver =
        new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        resolver.setExposeContextBeansAsAttributes(true);
        return resolver;
    }
    @Override
    public void configureDefaultServletHandling(
    DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
```
그리고 위와 같이 WebConfigure를 설정할 수 있는데 @EnableWebMvc와 @ComponentScan을 사용한다. @ComponentScan를 쓰는 이유는 이와 같은 Config파일들도 스프링에서 빈으로 등록되어야 쓸 수 있기 때문이다. 그리고viewResolver()함수와 같이 ViewResolver를 설정하여서 실제 view의 파일의 위치를 알려줄 수 있도록 설정해야 한다. 그리고 configureDefaultServletHandling()메소드와 같이 DefaultServletHandlerConfigurer를 설정해주어야 하는데 이것은 스태틱한 콘텐츠들을 다루는 것을 설정해주는 것이다.
그리고 이번 챕터는 Web개발 관련이기 때문에 Root configure는 나중에 다른 챕터에서 다룰것이다. 

```Java
@Controller
@RequestMapping({"/", "/homepage"})
public class HomeController {
    @RequestMapping(method=GET)
    public String home() {
        return "home";
    }
}
```

위와 같이 @Controller와 @RequestMapping을 이용해서 컨트롤러를 설정할 수 있다. @RequestMapping은 URL에 알맞는 클래스와 메소드를 실행될 수 있게 해준다.

```Java
public class HomeControllerTest {
    @Test
    public void testHomePage() throws Exception {
        HomeController controller = new HomeController();
        MockMvc mockMvc =
                standaloneSetup(controller).build();
        mockMvc.perform(get("/"))
                .andExpect(view().name("home"));
    }
}
```

그리고 위와 같이 mockMvc를 이용해서 web server와 web browser없이 Spring MVC를 간편하게 테스트를 할 수 있다. 

```Java
public class Spittle {
    private final Long id;
    private final String message;
    private final Date time;
    private Double latitude;
    private Double longitude;

    public Spittle(String message, Date time) {
        this(message, time, null, null);
    }
    public Spittle(
        String message, Date time, Double longitude, Double latitude) {
        this.id = null;
        this.message = message;
        this.time = time;
        this.longitude = longitude;
        this.latitude = latitude;
    }
    public long getId() {
        return id;
    }
    public String getMessage() {
        return message;
    }
    public Date getTime() {
        return time;
    }
    public Double getLongitude() {
        return longitude;
    }
    public Double getLatitude() {
        return latitude;
    }
    @Override
    public boolean equals(Object that) {
        return EqualsBuilder.reflectionEquals(this, that, "id", "time");
    }
    @Override
    public int hashCode() {
        return HashCodeBuilder.reflectionHashCode(this, "id", "time");
    }
}
```
위와 같이 Spittle를 위한 Data Object를 만드는데 여기서 equals와 hashcode메소드는 간단히 구현하기 위해 Apache Commons Lang를 사용했다.

```Java
@Test
public void shouldShowRecentSpittles() throws Exception {
    List<Spittle> expectedSpittles = createSpittleList(20);
    SpittleRepository mockRepository = mock(SpittleRepository.class);
    when(mockRepository.findSpittles(Long.MAX_VALUE, 20)).thenReturn(expectedSpittles);
    
    SpittleController controller = new SpittleController(mockRepository);

    MockMvc mockMvc = standaloneSetup(controller)
                        .setSingleView(new InternalResourceView("/WEB-INF/views/spittles.jsp"))
                        .build();

    mockMvc.perform(get("/spittles"))
            .andExpect(view().name("spittles"))
            .andExpect(model().attributeExists("spittleList"))
            .andExpect(model().attribute("spittleList", hasItems(expectedSpittles.toArray())));
}

private List<Spittle> createSpittleList(int count) {
    List<Spittle> spittles = new ArrayList<Spittle>();
    for (int i=0; i < count; i++) {
        spittles.add(new Spittle("Spittle " + i, new Date()));
    }
    return spittles;
}
```
위와 같이 메소드를 생성하기 전에 먼저 Test case를 생성한다.(저자가 TDD를 기반으로 개발을 하는 것으로 보인다.)

```Java
@Controller
@RequestMapping("/spittles")
public class SpittleController {
    private SpittleRepository spittleRepository;
    
    @Autowired
    public SpittleController(SpittleRepository spittleRepository) {
            this.spittleRepository = spittleRepository;
    }

    @RequestMapping(method=RequestMethod.GET)
    public String spittles(Model model) {
        model.addAttribute(spittleRepository.findSpittles(Long.MAX_VALUE, 20));
        return "spittles";
    }
}
```
그리고 위와 같이 컨트롤에 spittles를 불러오는 핸들러 메소드를 생성한다.
모델 객체는 spring에서 사용하는 객체인데 이것은 key-value pair방식이다. 하지만 위와 같이 key없이 value만 넣었을 경우 값의 객체 타입을 통해서 key을 임의로 지정하는데 여기서는 spittleList가 될 것이다.

```Java
@RequestMapping(method=RequestMethod.GET)
public String spittles(Model model) {
    model.addAttribute("spittleList", spittleRepository.findSpittles(Long.MAX_VALUE, 20));
    return "spittles";
}
```

```Java
@RequestMapping(method=RequestMethod.GET)
public String spittles(Map model) {
    model.put("spittleList", spittleRepository.findSpittles(Long.MAX_VALUE, 20));
    return "spittles";
}
```

```Java
@RequestMapping(method=RequestMethod.GET)
public List<Spittle> spittles() {
    return spittleRepository.findSpittles(Long.MAX_VALUE, 20));
}
```

위와 메소드들은 전부 같은 기능을 하는 메소드들로 이와 같이 여러 방식으로 나타낼 수 있다.

```JSP
<c:forEach items="${spittleList}" var="spittle" >
    <li id="spittle_<c:out value="spittle.id"/>">
        <div class="spittleMessage">
            <c:out value="${spittle.message}" />
        </div>
        <div>
            <span class="spittleTime"><c:out value="${spittle.time}" /></span>
            <span class="spittleLocation">
                (<c:out value="${spittle.latitude}" />,
                <c:out value="${spittle.longitude}" />)</span>
        </div>
    </li>
</c:forEach>
```
그리고 위와 같이 jsp를 이용해서 클라이언트를 나타낼 수 있는데, model의 값은 실제 request의 attributes로 전달되서 클라이언트에 값을 나타낼 수 있게 된다.

파라미터를 전달하는 방법에는 다음 3가지가 있다.
- Query parameters
- Form parameters
- Path variables

```Java
@Test
public void shouldShowPagedSpittles() throws Exception {
    List<Spittle> expectedSpittles = createSpittleList(50);
    SpittleRepository mockRepository = mock(SpittleRepository.class);
    when(mockRepository.findSpittles(238900, 50)).thenReturn(expectedSpittles);

    SpittleController controller = new SpittleController(mockRepository);
    MockMvc mockMvc = standaloneSetup(controller).setSingleView(new InternalResourceView("/WEB-INF/views/spittles.jsp")).build();

    mockMvc.perform(get("/spittles?max=238900&count=50"))
            .andExpect(view().name("spittles"))
            .andExpect(model().attributeExists("spittleList"))
            .andExpect(model().attribute("spittleList", hasItems(expectedSpittles.toArray())));
}
```
위와 같이 먼저 테스트를 생성하고 핸들러 메소드를 구현한다.

```Java
@RequestMapping(method=RequestMethod.GET)
public List<Spittle> spittles(
    @RequestParam("max") long max,
    @RequestParam("count") int count) {
    return spittleRepository.findSpittles(max, count);
}
```

위와 같이 쿼리파라미터 방식으로 넒어오는 것은 @RequestParam을 이용하여 값을 받을 수 있다.

```Java
@RequestMapping(value="/{spittleId}", method=RequestMethod.GET)
public String spittle(@PathVariable("spittleId") long spittleId, Model model) {
    model.addAttribute(spittleRepository.findOne(spittleId));
    return "spittle";
}
```
그리고 Path variables는 위와 같이 placeholders와 @PathVariable를 이용해서 값을 받을 수 있다. 여기서 {}값과 파라미터로 받는 변수명이 같다면 @PathVariable long spittleId와 같이 간략하게 적을 수도 있다.

```Java
@Test
public void shouldProcessRegistration() throws Exception {
    SpitterRepository mockRepository = mock(SpitterRepository.class);
    Spitter unsaved = new Spitter("jbauer", "24hours", "Jack", "Bauer");
    Spitter saved = new Spitter(24L, "jbauer", "24hours", "Jack", "Bauer");
    when(mockRepository.save(unsaved)).thenReturn(saved);

    SpitterController controller = new SpitterController(mockRepository);

    MockMvc mockMvc = standaloneSetup(controller).build();
    mockMvc.perform(post("/spitter/register")
            .param("firstName", "Jack")
            .param("lastName", "Bauer")
            .param("username", "jbauer")
            .param("password", "24hours"))
            .andExpect(redirectedUrl("/spitter/jbauer"));
    verify(mockRepository, atLeastOnce()).save(unsaved);
}
```
위는 Form parameters의 test메소드 코드이다. 위와 같이 mockMvc의 param파라미터를 이용해서 form의 파라미터처럼 파라미터를 보낼 수 있다.

```Java
@Controller
@RequestMapping("/spitter")
public class SpitterController {
    private SpitterRepository spitterRepository;
    @Autowired
    public SpitterController(
        SpitterRepository spitterRepository) {
        this.spitterRepository = spitterRepository;
    }
    @RequestMapping(value="/register", method=GET)
    public String showRegistrationForm() {
        return "registerForm";
    }
    @RequestMapping(value="/register", method=POST)
    public String processRegistration(Spitter spitter) {
        spitterRepository.save(spitter);
        return "redirect:/spitter/" + spitter.getUsername();
    }
}
```
위와 같이 form에서 보낸 파라미터를 객체를 통해서 받을 수 있다. form에서 보낸 파라미터명이 객체의 변수명과 같으면 그 변수에 파라미터의 값을 할당한다. 그리고 view이름 앞에 redirect는 해당 url을 redirect하겠다는 표시이다.

```Java
@RequestMapping(value="/{username}", method=GET)
public String showSpitterProfile(@PathVariable String username, Model model) {
    Spitter spitter = spitterRepository.findByUsername(username);
    model.addAttribute(spitter);
    return "profile";
}
```
그리고 form을 post로 submit(Post request)할때는 중복된 값을 방지할 목적(Browser refresh로 인해 우연히 두번이나 form을 보내지 않기 위해)으로 리다이렉션을 해야 되는데 지금 예제에서는 리다이렉션으로 그 유저의 프로파일로 url을 보낸다. 위의 코드가 그에 해당하는 메소드이다.

```Java
public class Spitter {
    private Long id;
    @NotNull
    @Size(min=5, max=16)
    private String username;
    @NotNull
    @Size(min=5, max=25)
    private String password;
    @NotNull
    @Size(min=2, max=30)
    private String firstName;
    @NotNull
    @Size(min=2, max=30)
    private String lastName;
    ...
}
```
위의 코드와 같이 @NotNull, @Size 등의 애노테이션을 사용하여 request로 받은 값을 검증할 수 있다. 

```Java
@RequestMapping(value="/register", method=POST)
public String processRegistration(@Valid Spitter spitter, Errors errors) {
    if (errors.hasErrors()) {
        return "registerForm";
    }
    spitterRepository.save(spitter);
    return "redirect:/spitter/" + spitter.getUsername();
}
```
그리고 위와 같이 파라미터를 받는 객체에 @Valid를 애노테이션해서 객체에 Validation 한것을 apply할 수 있다. 그리고 Errors객체의 hasErrors메소드를 실행하여 검증이 실패했는지 알 수 있고, 실패했을 경우 에러 처리를 한다. 참고로 Erros객체는 @Valid를 선언한 파라미터 옆에 와야 된다.



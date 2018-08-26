Spring in Action Chapter 8 - Working with Spring Web Flow
===

Configuring Web Flow in Spring
---
우선, 웹 플로는 Java config를 지원하지 않으므로 XML로 설정을 하여야 된다.

웹 플로를 XML에서 사용하기 위해서는 우선 아래처럼 namespace를 선언해주어야 된다.
~~~XML
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xmlns:flow="http://www.springframework.org/schema/webflow-config"
 xsi:schemaLocation=
  "http://www.springframework.org/schema/webflow-config
   http://www.springframework.org/schema/webflow-config/[CA]
   spring-webflow-config-2.3.xsd
http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd">
~~~

아래와 같이 플로 실행을 진행하는 플로 실행기를 선언할 수 있다.
~~~XML
<flow:flow-executor id="flowExecutor" />
~~~

flow registry를 통해 flow definitions를 불러올 수 있다.
불러오는 방법에는 아래와 같이 다양한 방법들이 있다.
~~~XML
<flow:flow-registry id="flowRegistry" base-path="/WEB-INF/flows">
           <flow:flow-location-pattern value="*-flow.xml" />
</flow:flow-registry>
~~~

~~~XML
<flow:flow-registry id="flowRegistry">
           <flow:flow-location path="/WEB-INF/flows/springpizza.xml" />
</flow:flow-registry>
~~~

~~~XML
<flow:flow-registry id="flowRegistry">
           <flow:flow-location id="pizza"
                 path="/WEB-INF/flows/springpizza.xml" />
</flow:flow-registry>
~~~

그리고 DispatcherServlet이 Spring Web Flow로 요청을 보내기 위해서는 아래와 같은 설정 또한 필요하다.
~~~XML
<bean class=
        "org.springframework.webflow.mvc.servlet.FlowHandlerMapping">
  <property name="flowRegistry" ref="flowRegistry" />
</bean>
~~~

The components of a flow && Putting it all together: the pizza flow 
---

책에는 위 2가지의 부분이 따로 있지만, 이렇게 둘이 함께 설명하는 것이 더 이해가 잘 갈 것이라고 생각하여 같이 정리를 할려고 한다.

우선, 이 책에서는 피자주문과정을 웹 플로를 사용하여 개발하였는데, 이 것을 2가지의 관점에서 본다.
높은 계층에서 플로우를 만들고 낮은 게층을 플로우를 만드는 것이다.

책에서 나와 있듯이 높은 계층에서의 플로우는 identify Customer, buildOrder, takePayment, saveOrder, thankCustomer의 5가지 상태가 간략하에 나타나있고, 각 이벤트가 발생했을 때 transitions이 일어난다.

이러한 과정들은 아래와 같이 설정할 수 있다.
~~~XML
<?xml version="1.0" encoding="UTF-8"?>
<flow xmlns="http://www.springframework.org/schema/webflow"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.springframework.org/schema/webflow http://www.springframework.org/schema/webflow/spring-webflow-2.3.xsd"> <var name="order"
     class="com.springinaction.pizza.domain.Order"/>
<subflow-state id="identifyCustomer" subflow="pizza/customer">
  <output name="customer" value="order.customer"/>
  <transition on="customerReady" to="buildOrder" />
</subflow-state>
<subflow-state id="buildOrder" subflow="pizza/order">
  <input name="order" value="order"/>
  <transition on="orderCreated" to="takePayment" />
</subflow-state>
<subflow-state id="takePayment" subflow="pizza/payment">
    <input name="order" value="order"/>
  <transition on="paymentTaken" to="saveOrder"/>
</subflow-state>
<action-state id="saveOrder">
  <evaluate expression="pizzaFlowActions.saveOrder(order)" />
    <transition to="thankCustomer" />
  </action-state>
  <view-state id="thankCustomer">
    <transition to="endState" />
  </view-state>
  <end-state id="endState" />
  <global-transitions>
    <transition on="cancel" to="endState" />
  </global-transitions>
</flow>
~~~

여기서 보면 var로 전체 플로우에서 사용할 변수를 선언할 수 있고, subflow-state를 통해 각각의 상태를 정의할수 있고, 그에 따른 subflow로 선언할 수 있다.
그리고 output요소를 사용하여 결과값을 받을 수 있고, input을 사용하여 데이터를 보내줄 수도 잇다.
action-state는 빈의 메소드를 수행할 수 있고, view-state는 사용자가에 정보를 표시해주고 값을 받을 수 있는 역할을 한다.
transition요소는 on에 있는 이벤트나 상태가 되었을 때, to의 상태로 플로를 transition시킨다.
이와 같은 것들이 web flow의 전반적인 기능과 설정방법들이다.

그리고 아래는 변수로 선언된 order의 소스코드이다.
~~~Java
package com.springinaction.pizza.domain;
import java.io.Serializable;
import java.util.ArrayList;
import java.util.List;
public class Order implements Serializable {
   private static final long serialVersionUID = 1L;
   private Customer customer;
   private List<Pizza> pizzas;
   private Payment payment;

   public Order() {
      pizzas = new ArrayList<Pizza>();
      customer = new Customer();
   }
   public Customer getCustomer() {
      return customer;
   }
   public void setCustomer(Customer customer) {
      this.customer = customer;
   }
   public List<Pizza> getPizzas() {
      return pizzas;
   }
   public void setPizzas(List<Pizza> pizzas) {
      this.pizzas = pizzas;
   }
   public void addPizza(Pizza pizza) {
      pizzas.add(pizza);
   }
   public float getTotal() {
      return 0.0f;
   }
   public Payment getPayment() {
      return payment;
   }
   public void setPayment(Payment payment) {
      this.payment = payment;
    } 
}
~~~

 web flow에서는 처음 선언된 상태가 처음 시작상태로 설정되는데, 아래와 같이 start-state를 선언하여 명시적으로 나타낼 수도 있다.
 ~~~XML
<?xml version="1.0" encoding="UTF-8"?>
<flow xmlns="http://www.springframework.org/schema/webflow"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.springframework.org/schema/webflow http://www.springframework.org/schema/webflow/spring-webflow-2.3.xsd" start-state="identifyCustomer">
... </flow>
~~~

그리고 아래와 같이 view단에는 flowExecutionUrl이 내장되어 있는데, 이것은 flow를 위한 URL을 포함하며 뷰에서 사용할 수 있다. 그리고 이 URL뒤에 _eventId의 파라미터를 붙임으로써 웹 플로에서 이벤트를 발생시킬 수 있으며, 플로우의 전이를 발생시킨다.

~~~JSP
<html xmlns:jsp="http://java.sun.com/JSP/Page">
  <jsp:output omit-xml-declaration="yes"/>
  <jsp:directive.page contentType="text/html;charset=UTF-8" />
  <head><title>Spizza</title></head>
  <body>
    <h2>Thank you for your order!</h2>
    <![CDATA[
    <a href='${flowExecutionUrl}&_eventId=finished'>Finish</a>
    ]]>
    </body>
</html>
~~~

그리고 책에서는 각각이 subflow를 전부 다 설명하고 있는데, 제가 생각할 때는 가장 표준적인 1개의 예만 설명을 해도 전부 이해하고 충분히 이 기능을 활용할 수 있을 것이라고 생각한다.

~~~XML
<?xml version="1.0" encoding="UTF-8"?>
<flow xmlns="http://www.springframework.org/schema/webflow"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.springframework.org/schema/webflow http://www.springframework.org/schema/webflow/spring-webflow-2.3.xsd"> <var name="customer"
     class="com.springinaction.pizza.domain.Customer"/>
<view-state id="welcome">
   <transition on="phoneEntered" to="lookupCustomer"/>
</view-state>
<action-
   state id="lookupCustomer">
       <evaluate result="customer" expression=
"pizzaFlowActions.lookupCustomer(requestParameters.phoneNumber)" />
 <transition to="registrationForm" on-exception=
     "com.springinaction.pizza.service.CustomerNotFoundException" />
  <transition to="customerReady" />
</action-state>
<view-state id="registrationForm" model="customer">
   <on-entry>
    <evaluate expression=
  "customer.phoneNumber = requestParameters.phoneNumber" />
</on-entry>
  <transition on="submit" to="checkDeliveryArea" />
</view-state>
<decision-state id="checkDeliveryArea">
  <if test="pizzaFlowActions.checkDeliveryArea(customer.zipCode)"
      then="addCustomer"
      else="deliveryWarning"/>
</decision-state>
<view-state id="deliveryWarning">
   <transition on="accept" to="addCustomer" />
</view-state>
<action-state id="addCustomer">
         <evaluate expression="pizzaFlowActions.addCustomer(customer)" />
    <transition to="customerReady" />
  </action-state>
  <end-state id="cancel" />
  <end-state id="customerReady">
    <output name="customer" />
  </end-state>
  <global-transitions>
    <transition on="cancel" to="cancel" />
  </global-transitions>
</flow>
~~~

위의 예는 identifyCustomer의 subflow인 pizza/customer로 여기에 있는 기능들은 아까 설명한 것과 같다. 설명한 것중에는 end-state와 global-transitions이 있는데, end-state는 말그대로 이 플로가 끝나는 상태이고, global-transitions는 공통적으로 쓸 수 있는 상태를 선언하여, 이 플로내에서 전체적으로 그 상태를 쓸 수 있게 해주는 요소이다.

~~~XML
<html xmlns:jsp="http://java.sun.com/JSP/Page"
     xmlns:form="http://www.springframework.org/tags/form">
  <jsp:output omit-xml-declaration="yes"/>
  <jsp:directive.page contentType="text/html;charset=UTF-8" />
  <head><title>Spizza</title></head>
  <body>
<h2>Welcome to Spizza!!!</h2>
    <form:form>
      <input type="hidden" name="_flowExecutionKey"
             value="${flowExecutionKey}"/>
       <input type="text" name="phoneNumber"/><br/>
      <input type="submit" name="_eventId_phoneEntered"
             value="Lookup Customer" />
     </form:form>
  </body>
</html>
~~~

그리고 위와 같이 뷰에서는 flowExecutionKey라는 것이 있는데 이것은 flowExecutionUrl와 마찬가지로 뷰에 내장되어 있는 것인데 이것은 뷰에서 다시 플로로 이동하였을 때, 플로가 중단된 지점부터 다시 시작할 수 있도록 도와주는 tag같은 것이다.
그리고 name에 _eventID_'EventId'이런식으로 값을 주게 되면 EventId인 이벤트가 발생하여 그 이벤트에 해당하는 transition이 발생되도록 해준다.

이렇게 위와 같이 큰 틀의 플로에다가 필요한 subflow를 차레대로 구성하게 되면 플로를 잘 사용할 수 있게 된다.

Securing web flows
---
웹 프로 보안은 secured 요소를 사용하여 각 권한에 맞는 유저만 사용할 수 있도록 지정할 수 있다.

간단한 예는 아래와 같다.
~~~XML
<view-state id="restricted">
  <secured attributes="ROLE_ADMIN" match="all"/>
</view-state>
~~~


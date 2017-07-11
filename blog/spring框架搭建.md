#<center>JAVA开发血泪之路：一步步搭建spring框架</center>

##  前言
作为一个服务端开发感觉一直挺排斥框架这种东西的，总觉得什么实现逻辑都帮你封装在里面了，你只需要配置这配置那个，出了问题也不知道怎么排查，之前即使写web程序也宁愿使用jetty这样的嵌入式的web server实现，自己写servlet，总感觉从main函数开始都在自己的掌控范围之内，但是这样的方式的确有点原始，也看到各种各样的开源系统使用spring实现web服务，虽然代码总是能够看明白，但是还是不晓得一步步是怎么搭建的，于是抽出一个周末折腾折腾，不搞不知道，原来这玩意能把一个不熟悉的用户搞崩溃，本文主要介绍我是如何搭建一个spring环境的（话说还真的分不清spring和springmvn），虽然在大多数web开发看来这是雕虫小技。

本文使用的环境是eclipse luna+spring比较新的一个版本（按照我选择版本的规则是除非有什么新功能新版本才有，否则尽量不使用最新的版本，然后选择较新的N个版本中使用人数比较多的，例如本文选用的spring版本是4.3.7.RELEASE）。

下面就从纯工程的角度上解释如何一步步的搭建这样的环境的，没有原理，有原理也是我纯属猜测的，没有看过源码。

## 详细步骤

### 第一步：创建一个maven工程

这是再熟悉不过的流程了，但是一般我不推荐选择Archetype，只是创建一个simple project就可以了，前者总是创建失败（创建Archetype模式的可以让IDE做更多的事情）。其实在何谓maven工程，在我看来就是一个带有pom.xml的java工程罢了，然后再把代码的路径设置为src/main/java类似这样的结构，所以我们只需要用IDE帮我们创建一个带有pom.xml的工程就可以了，我们自己写一个dependency和build参数。

配置的时候除了填写正确的group id 和artifact id，主要把packaging选择为war，这样可以在tomcat下运行。

### 第二步：修改工程配置

这里需要修改的配置有两个，只需要注意修改之后的样子就可以了：

* 1、Project Facets：虽然不知道这里配置什么的，但是一次次的吃了这个的亏，这里要配置的是Java选择1.6以上（最好1.7吧），然后选择Dynamic Web Module，下方出现如下的界面：
<center>![这里写图片描述](http://img.blog.csdn.net/20170425145424479?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveXU2MTY1Njg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)</center>

如果没出现则可以先勾掉Dynamic Web Module，然后保存，然后再次点进去Project Facets，选择Dynamic Web Module，这时候就出现了这样的界面，注意最好不要选择3.0，之前遇到过3.0不兼容的问题，jdk1.7 + 2.5版本是可以正常运行的。

点进去“Further configuration avaliable...”进行配置，将Context directory修改成，并选择生成web.xml，保存。如下图：

<center>![这里写图片描述](http://img.blog.csdn.net/20170425145543731?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveXU2MTY1Njg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)</center>

此时你会看到你的工程结构如下图，src/main目录下出现了java/resources/webapp三个目录。

<center>![这里写图片描述](http://img.blog.csdn.net/20170425145643217?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveXU2MTY1Njg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)</center>

* 2、配置Deployment Assembly，这里配置的Source和Deploy Path，表示在工程部署的时候会将source目录下的内容拷贝到tomcat部署目录/Deploy Path下。这里需要配置的如下图所示：

<center>![这里写图片描述](http://img.blog.csdn.net/20170425145947808?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveXU2MTY1Njg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)</center>

例如第一条表示会将工程中src/main/java目录下的源代码编译之后放到部署目录/WEB-INF/classes目录下，最后一条表示会将该工程的maven依赖拷贝到部署目录/WEB-INF/lib目录下。据我观察发现，其实tomcat目录运行过程中会将部署部署目录/WEB-INF/classes、部署目录/WEB-INF/lib加入到classpath中，所以将配置文件和编译完成的class文件放到classes下，依赖的jar放到lib目录下都是可以在启动java程序时找得到的。

### 第三步：下载spring依赖

spring的jar比较多，最基本的功能也需要如下的几个dependency：

	  <properties >
	       <spring.version >4.3.7.RELEASE </spring.version>
	  </properties >
	
	  <dependencies >
	        <dependency>
	           <groupId> org.springframework</groupId >
	           <artifactId> spring-context</artifactId >
	           <version> ${spring.version}</version >
	        </dependency>
	        <dependency>
	           <groupId> org.springframework</groupId >
	           <artifactId> spring-core</artifactId >
	           <version> ${spring.version}</version >
	        </dependency>
	        <dependency>
	           <groupId> org.springframework</groupId >
	           <artifactId> spring-beans</artifactId >
	           <version> ${spring.version}</version >
	        </dependency>
	       <dependency >
	           <groupId> org.springframework</groupId >
	           <artifactId> spring-web</artifactId >
	           <version> ${spring.version}</version >
	        </dependency>
	        <dependency>
	           <groupId> org.springframework</groupId >
	           <artifactId> spring-webmvc </artifactId>
	           <version> ${spring.version}</version >
	        </dependency>
	  </dependencies >

spring的不同的依赖使用的版本要保持一致。

### 第四步：编写代码

我们写一个简单的controller，这个controller返回一个“Hello ${username}”字符串。

	package com.fengyu.test;
	
	import org.springframework.web.bind.annotation.RequestMapping;
	import org.springframework.web.bind.annotation.RequestMethod;
	import org.springframework.web.bind.annotation.RequestParam;
	import org.springframework.web.bind.annotation.ResponseBody;
	import org.springframework.web.bind.annotation.RestController;
	
	@RestController
	@RequestMapping("/test" )
	public class SimpleController {
	        @RequestMapping(value = "hello", method = RequestMethod.GET)
	        @ResponseBody
	        public String helloWorld(@RequestParam ("user") String userName) {
	               return "Hello " + userName + " !" ;
	       }
	}

### 第五步：spring配置文件

spring配置文件一般取名叫"applicationContext.xml"，当然这个不是spring默认的配置名，还是需要在web.xml中指定，这里我们只配置spring的配置文件，其实spring配置中主要是对一些bean的配置，这里我们暂时不需要创建任何bean，就只需要简单地加一下扫描路径就可以了。

	<beans xmlns= "http://www.springframework.org/schema/beans"
	       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	       xmlns:context="http://www.springframework.org/schema/context"
	       xmlns:mvc="http://www.springframework.org/schema/mvc"
	        xsi:schemaLocation="http://www.springframework.org/schema/beans
	        http://www.springframework.org/schema/beans/spring-beans.xsd
	        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
	        http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd ">
	     
	    < context:annotation-config />
	    <!-- 自动扫描web包 ,将带有注解的类 纳入spring容器管理 -->  
	    <context:component-scan base-package="com.fengyu.test"></context:component-scan>  
	</beans>

这种配置文件一般放在src/main/resources目录下，前面我们已经配置部署拷贝的设置，他会在部署时被拷贝到WEB-INF/classes/目录下，这里只配置了两项其中context:annotation-config是告诉spring识别注解配置，后面的scan表示要扫描的类的package。

### 第六步：配置web.xml

web.xml使我们第二步配置时自动生成的，它是tomcat启动时需要依赖的配置，里面可以配置一些servlet、filter、listener等，对于简单使用spring而言，一般只配置一个servlet，所有的请求都有这个servlet进行路由处理。

	<?xml version= "1.0" encoding ="UTF-8"?>
	<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd" id= "WebApp_ID" version ="2.5">
	    <servlet > 
	        <servlet-name> DispatcherServlet</servlet-name >
	        <servlet-class> org.springframework.web.servlet.DispatcherServlet</servlet-class > 
	        <init-param>  
	            <param-name> contextConfigLocation</param-name > 
	            <param-value> classpath:applicationContext.xml</param-value > 
	        </init-param>  
	        <load-on-startup> 1</ load-on-startup>
	    </servlet > 
	    <servlet-mapping > 
	        <servlet-name> DispatcherServlet</servlet-name > 
	        <url-pattern> /</ url-pattern>
	    </servlet-mapping >
	</web-app>   

可以看到我们只配置了一个DispatcherServlet，它处理所有的url请求，它初始化需要的配置文件会classpath下找applicationContext.xml，刚才已经介绍，在部署的时候resources下的文件会拷贝到WEB-INF/classes目录下并且加入到java启动的classpath下。

### 第七步：部署tomcat

首先需要下载一个tomcat，tomcat 7.x是一个不错的选择，由于tomcat是绿色的（大多数java实现的都是绿色的），可以直接解压，然后在eclipse中配置。在eclipse中Window->Server->Runtime Environment中Add一个tomcat实例，注意需要选择jdk，需要选择当前jdk支持的版本。

然后再下方任务栏里面找到Servers，如果没有可以再Window->Show View中选择Servers添加，添加了之后可以在这里create a new server。选择刚刚创建的tomcat实例，然后从Avaliable的resources中选择一个加入到右边Configured中（Avaliable是根据工程中是否有web.xml生成的）。

###　第八步：配置tomcat

双击新创建的Tomcat Server，进入Overview页面，这个页面可以配置这个工程运行tomcat实例，主要配置包括端口号，部署目录等等，如果端口或文件不冲突的话尽量不要修改，需要修改的一处是Server Options中勾选上“Publish module contexts to separate XML files”，虽然不知道这个是做什么的，但是血和泪的教训告诉我要选上这个，直接保存。

### 第九步：启动tomcat

这里可能有人要问了，为什么eclipse上没有配置tomcat的插件啊，我到哪里去启动tomcat！难道没有那个猫头就不行了吗？说实话，配置那个猫头经常遇到网络问题失败，于是不再尝试了，而直接通过右击第七步创建的tomcat实例就可以完成猫头所能完成的所有功能，为什么还配置那个插件呢？右键Start，然后祈祷启动的时候不要出现什么错误，出现错误就google一下吧，基本上spring的问题都有人踩过坑的。

### 第十步：测试

启动完成之后，一般不会出现什么错误，打开浏览器输入http://localhost:8080/SpringTest/test/hello?user=World，此时就可以看到如下的输出：

<center>![这里写图片描述](http://img.blog.csdn.net/20170425150751872?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveXU2MTY1Njg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)</center>

出现了我们想要的结果，此时的心情只能用愉悦来形容，但是我们指定了需要携带user参数，如果url中不带参数则出现如下的错误：

<center>![这里写图片描述](http://img.blog.csdn.net/20170425150856030?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveXU2MTY1Njg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)</center>

如果不希望出现这样的情况可以再helloWorld这个参数的@RequestParam修改为@RequestParam(value="user",required=false, defaultValue="World")，当然这只是一个非常小的example说明注解的强大，spring还提供了丰富的注解来实现不同的需求，简直就是用配置和注解来编程。

### 第十一步：复杂数据结构

上面的测试都是返回一个String，但是我们一般开发的时候会涉及到复杂的数据结构，大家一般都会用json来作为通信的序列化工具，那么怎么在spring中支持json呢？在测试这个之前，先看一下现在对于复杂数据结构（例如Map）怎么返回的。
我们在Controller中添加一个函数：

        @RequestMapping(value = "helloMap" , method = RequestMethod.GET)
        @ResponseBody
        public Map<String, String> helloMap(@RequestParam(value="user" ,required=false, defaultValue= "World") String userName) {
              Map<String, String> ret = new HashMap<String, String>();
               ret.put( "hello", userName );
               return ret ;
       }

然后再浏览器中测试一下，出现如下的错误" The resource identified by this request is only capable of generating responses with characteristics not acceptable according to the request "accept" headers.",这个感觉是因为accept默认只接受text格式，而这个接口的返回值没有返回text对象。

如果要支持json，其实只需要在spring中配置一下message-converters就可以了，每一个json库都提供这样的Converter的，我们这里以Fastjson作为示例吧，首先需要在pom.xml添加fastjson的依赖。然后在applicationContext.xml中添加如下配置：

	<mvc:annotation-driven>
               <mvc:message-converters register-defaults="false" >
                      <bean
                             class= "com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter" >
                            <property name= "supportedMediaTypes">
                                   <list>
                                          <value> text/html; charset=UTF-8</value >
                                          <value> application/json; charset=UTF-8</value >
                                   </list>
                            </property>
                      </bean>
               </mvc:message-converters>
        </mvc:annotation-driven>

这时候再重试刚才的url发现可以返回map转换成json的样子了。

<center>![这里写图片描述](http://img.blog.csdn.net/20170425151840698?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveXU2MTY1Njg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)</center>

### 第十一步：折腾

好了，上面的示例已经足够完成一个比较简单的web服务了，当然我们还没有前端，只不过是通常作为服务端提供的Http服务罢了，不过有了spring我们可以省去大量的代码，当然通过HTTP+JSON提供服务端接口比thrift等RPC框架要中一些，效率或许要低一些，但是它的优势是比较简单啊，调试比较方便啊。。。感觉大量的前端和服务端通信大都使用这样的方式。

## 总结

学会了spring框架的搭建，妈妈再也不用担心我写web接口了，当然spring还能够适配各种各样的组件，例如通常使用的mybatis连接数据库，jedis连接redis等；还有丰富的功能让你尽量通过配置和注解降低不必要的代码。最近比较好的Spring boot貌似也是web开发神奇。这个以后有时间再折腾。
当然本文只是记录了血泪历程，spring小白是如何一步步的搭建spring开发环境的，大神轻喷。。

本文代码可以在 https://github.com/terry-chelsea/spring-demo.git 找到
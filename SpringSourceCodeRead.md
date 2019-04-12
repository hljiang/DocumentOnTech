## Spring_Source_Code_Read
### 1、property 解析

```
1、在PropertyPlaceholderBeanDefinitionParser的父类中shouldGenerateId返回true
    1.1 即默认会为每一个bean生成一个唯一的名字
    1.2 如果使用了两个<context:property-placeholder则注册了两个PropertySourcesPlaceholderConfigurer Bean
    1.3 所以不是覆盖（而且bean如果同名是后边的bean定义覆盖前边的）
2、PropertySourcesPlaceholderConfigurer本质是一个BeanFactoryPostProcessor
    2.1 spring实施时如果发现这个bean实现了Ordered，则按照顺序执行；默认无序；
3、此时如果给<context:property-placeholder加order属性，则会反应出顺序
    3.1 order值越小优先级越高即越早执行；
example:
   <context:property-placeholder order="2" location="classpath*:com/zyc/jdbc.properties"/> 
   <context:property-placeholder order="1" location="classpath*:com/zyc/jdbc1.properties"/>
此时会先扫描order='1' 的，如果value没在1中找到，会抛出异常，如果忽略异常，会继续扫描order='2'的继续查找
4、默认情况下ignore-unresolvable；即如果没找到的情况是否抛出异常。默认false：即抛出异常；
<context:property-placeholder location="classpath*:conf/conf_a.properties" ignore-unresolvable="false"/> 
```

#### 1.1 PropertyPlaceeholderConfigurer

```
// 在beanDefinition中尝试用已知配置来替代拥有${...}占位符的property
@Override
protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props) throws BeansException {
	StringValueResolver valueResolver = new PlaceholderResolvingStringValueResolver(props);
	doProcessProperties(beanFactoryToProcess, valueResolver);
}
```

##### 1) PlaceholderConfigurerSupport
 
```
public abstract class PlaceholderConfigurerSupport extends PropertyResourceConfigurer
		implements BeanNameAware, BeanFactoryAware{....}
// 两种占位符写法，一种是注解，一种是xml直接配置
<bean id="dataSource" 
class="org.springframework.jdbc.datasource.DriverManagerDataSource"/>
    <property name="driverClassName" value="${driver}"/>
    <property name="url" value="jdbc:${dbname}"/>
</bean>
@Value("${person.age}")
-> org.springframework.beans.factory.annotation.Value
```
> 可以定义多个configurers，可以拥有自己的placeholder syntax，其中涉及到ignoreUnresolvablePlaceholders属性，如果一个Configurer管理的properties中，解析Placeholder失败，默认是会抛出异常，如果是true，会忽略异常，然后继续下一个configurer里查找

```
PlaceholderConfigurerSupport中就定义了基本的占位符格式，perfix-> ${，suffix -> }，分隔符-> :
<property name="url" value="jdbc:${dbname:defaultdb}"/>
spring会找到placeholder根据：来进行split，:后面是默认值
```
 
#### 1.2 PropertyOverrideConfigurer 允许设定默认值，然后重写property的value

```
@See PropertyOverrideConfigurer --> beanName.property=value 
//beanName 是属性占位符企图覆盖的bean 名， property 是企图覆盖的属性名
public class PropertyOverrideConfigurer extends PropertyResourceConfigurer
----------------------------------------------------------
Example properties file:
    dataSource.driverClassName=com.mysql.jdbc.Driver
    dataSource.url=jdbc:mysql:mydb
----------------------------------------------------------
<!-- 配置一个属性占位符能自动识别PropertyPlaceholderConfigurer bean --> 
	<bean id="propertyOverrider"  class= "org.springframework.beans.factory.config.PropertyOverrideConfigurer"> 
	    <property name="locations"> 
	        <list> 
	           <value>dbconn.properties</value> 
	           <!--如果有多个属性文件，依次在下面列出来 -->
	       </list> 
	   </property> 
	</bean> 
	
	<!-- 配置本地的DBCP 数据源 -->
	<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource" destroy-method="close">
	    <property name="driverClassName" value="test"/> // 该属性可以被覆盖
	    <property name="url" value="test"/>  // 该属性也可以被覆盖，根据example
	    <property name="username" value="dd"/> 
	    <property name="password" value="dd"/> 
	</bean>
```

#### 1.3 PropertySourcesPlaceholderConfigurer
 
```
public class PropertySourcesPlaceholderConfigurer extends PlaceholderConfigurerSupport implements EnvironmentAware {...}
// 可以替代PropertyPlaceeholderConfigurer
```

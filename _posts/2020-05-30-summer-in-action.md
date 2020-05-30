# Spring实战
## Spring配置
- Java配置@Configuration
- XML配置（可以使用Spring Tool Suite）

## Bean声明
- @Component声明
- Bean声明，包括Java配置的@Bean和XML配置的\<bean\>

## Bean发现
- 自动发现，包括Java配置的@ComponentScan和XML配置的\<context:component-scan\>，能发现@Component注解的组件
- @Bean或\<bean\>声明时发现

## Bean装配
- 自动装配@Autowired，会自动装配扫描或者声明时发现的Bean
- Bean声明时作为参数自动装配，会自动装配扫描或者声明时发现的Bean

## XML配置
- c-命名空间对应\<constructor-arg\>构造器注入，p-命名空间对应\<property\>属性注入
  
## 混合配置
- Java与Java配置混合————@Import
- XML配置混入Java配置————@ImportResource
- XML与XML配置混合————\<import\>
- Java配置混入XML配置————\<Bean\>

## Profile定义
- Java配置使用@Profile注解配置类或方法
- XML配置使用\<beans\> profile属性

## Profile激活
- 分为设定值和默认值，都没有时仅创建没有定义在Profile中的Bean

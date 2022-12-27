# 领域驱动设计与Hibernate

## 1 为什么DDD
Evans书副标题——Tracking complexity in the heart of software

### DDD的优势
- 增加开发人员与领域专家的交流，促进开发人员与业务人员的相互提升，使软件能更好地表达业务
- 提升业务价值，使组织更关注业务战略

### 解决的一些问题——贫血症和失忆症
JavaBean的泛滥，使得项目中大部分对象只有getter和setter，Service承担过多的职责

### DDD的挑战
- 为创建通用语言腾出时间和精力
- 持续地将领域专家引入项目
- 改变开发者对领域的思考方式

## 2 战略
战略对于开发人员来说是比较抽象的，特别是一些田园式的开发实践中，技术人员最能把控的，也就只有战术实施了（有时也称为DDD-Lite）。
在实践过程中，还需要许多的磨合。

Brandolini发明了事件风暴这种方法来进行战略设计。

### 通用语言
通用语言是团队创建的、作用于某一界限上下文的、可以演化的、表达领域模型的语言。
也是团队协作的模式，用于捕捉业务领域的概念和术语，反映了团队的思维。
它可能包含：
- UML
- 文档、表格里的术语和词组
- 代码
- 其它交流中表达的语言

### 限界上下文
不同团队对同一对象做了不同假设，使之能在自己的上下文中使用，当代码等被组合到一起时，就会产生问题。
因此限界上下文明确地定义模型所应用的上下文，根据团队的组织、软件系统各个部分的用法以及物理表现来设置模型边界。
限界上下文和通用语言之间是一对一的关系，在一个特定的限界上下文中，只能使用同一套通用语言。
Vernon把限界上下文看作是解决方案空间。

限界上下文的内容应该具有非歧义的领域语言。
同一现实中的事物，可以在不同的限界上下文中建模，共享同一个标识。

导致限界上下文大小不正确的因素：
- 采用架构来指导模型的设计开发，而不是通用语言；
- 根据开发任务的分配拆分（可以使用模块等战术化手段管理任务的分配，如OSGi或Jigsaw）。

多个团队，共享部分限界上下文（共享内核，多个团队共享一部分模型、相关代码、数据库设计），是不推荐的。

### 领域
为专注核心问题，而把领域分成多个子域。意在：
- 帮助所有团队成员掌握系统的总体设计及协调；
- 找到适度的核心模型并添加到通用语言，促进沟通；
- 指导重构；
- 专注最有价值的部分；
- 指导外包、现成组件的使用以及任务委派。

Vernon将子域看作是问题空间。

实践中普遍存在的一个问题是，稀缺的高水平开发人员往往会把工作重点放在基础设施或者已经被很好地解决了的问题上，而真正体现应用程序价值的却由水平稍差的开发人员完成。

### 上下文映射图
不同限界上下文通过上下文映射集成。限界上下文之间不同的关系，也代表了不同的团队关系。

不同界限上下文集成时，可以将外部的领域模型同步成本地的领域对象，此时将获得更大的自治性，这些对象保留着本地模型所需的最小状态集。

## 3 架构
软件架构是有关软件整体结构与组件的抽象描述，用于指导大型软件系统各个方面的设计。
架构可以很大，也可以具体到代码的组织上。

### 分层架构模式
分层架构被广泛使用（阿里巴巴Java开发手册中可见一种），是单一职责这一设计原则的体现。一个典型的传统DDD系统从上到下分为
1. 用户界面层 该层负责用户请求和结果，用户有可能是人或者其它系统。一般在实践中的Spring项目，用户界面层由@Controller、@RestController+前端来实现。
2. 应用层 DDD中的该层指挥领域层对象来解决问题，使它们互相协作，主要表达用例和用户故事。
3. 领域层 负责表达核心业务概念、业务状态信息及业务规则。
4. 基础设施层 为各层提供通用的技术能力。

每层的元素只依赖本层的其它元素或其下层的元素。

### 六边形、端口与适配器、洋葱架构
在六边形架构中，不同的"外部"通过"平等"的方式与"内部"交互。
左侧，应用层API通过输入适配器接收输入。右侧，持久化、消息通过输出适配器完成转化。

### 服务架构
服务架构一般是针对单体应用来说的。
> 剖析单体架构之前，我们有必要先厘清一个概念误区，许多微服务的资料里，单体系统往往是以“反派角色”的身份登场的，譬如著名的微服务入门书《微服务架构设计模式》，第一章的名字就是“逃离单体的地狱”。这些材料所讲的单体系统，其实都是有一个没有明说的隐含定语：“大型的单体系统”。
> ——icyfenix.cn

大型单体在部署、隔离等方面较为复杂，如OSGi。围绕服务架构也有许多受争议的地方（例如SOA与微服务，从历史的角度看大概能看明白），个人认为比较重要的几点设计原则是：
1. 服务自治性 每个服务有自己控制的环境及资源以保持独立，如每个服务可以选择合适的语言来实现、可以通过不同的限界上下文来对现实中的同一对象建模；
2. 服务组合性与重用性

James Lewis和Martin Fowler的《Microservices》还指出微服务应具备的特性：
1. 围绕业务能力组织 根据康威定律，组织架构会影响软件架构，而部门墙会带来不利的影响，围绕业务能力组织颇有敏捷中自组织团队的意思。
2. 容错
3. 演进式
> How big is a microservice?
> Although “microservice” has become a popular name for this architectural style, its name does lead to an unfortunate focus on the size of service, and arguments about what constitutes “micro”. In our conversations with microservice practitioners, we see a range of sizes of services. The largest sizes reported follow Amazon's notion of the Two Pizza Team (i.e. the whole team can be fed by two pizzas), meaning no more than a dozen people. On the smaller size scale we've seen setups where a team of half-a-dozen would support half-a-dozen services.
> This leads to the question of whether there are sufficiently large differences within this size range that the service-per-dozen-people and service-per-person sizes shouldn't be lumped under one microservices label. At the moment we think it's better to group them together, but it's certainly possible that we'll change our mind as we explore this style further.

### REST
在比较经典的开发实践中，可能会遇到这样的情况：
- 动态地展示某一前端控件的需求，例如管理后台在某一记录有效时显示"管理"按钮，展示的如果是无效记录时不显示按钮；
- 某一用例中，需要根据业务对象不同的状态，进行不同的交互流程。

在一般的实现中，会在前端做逻辑控制，根据后端返回的某个业务对象（JSON）的属性。
稍微"后端驱动"一点的，是后端直接返回控制属性，这个属性面向前端而不是后端对象，例如返回增加一个独立于业务对象属性的manageable来标识是否展示"管理"按钮，逻辑放在后端。
但是阅读代码、重构时，因为人的"失忆"或者人员的流动，用属性无法被快速地解释，特别是前端使用业务属性时。

而HATEOAS返回的JSON，就像是一个Java对象，既有状态，又有方法，具备更强的表达能力。

在结合DDD与REST时，可能会直观地将领域模型（业务对象）当作REST中的资源发布出去，这是不被建议的。可以为REST创建单独的限界上下文。

### CQRS
大数据近来被炒得火热，数据的分析、查询展示有着越来越复杂的需求（如跨服务跨库的查询）。
为了更好地让模型被表达和实现复杂的业务需求，可以把查询模型与命令模型分离开来。

虽然查询模型与命令模型分开了，但在实际开发中还可能遇到命令模型需要在执行前，需要验证是否满足某些约束的情况。
这时，命令模型里面对象的方法，也不仅仅只有"写"的方法，验证约束也可以成为命令模型的方法。
更进一步，这些业务规则，可以通过显式地创建谓词形式的对象（Specification），来表达约束，该对象只有一个方法——isSatisfiedBy(Object candidate)。
曾经参与的一个项目中，把读写完全分开建模，我不赞同这种做法。

### 事件驱动架构

#### 命令和事件
应当小心地区分事件和命令。
> 当来自用户的请求第一次到达时，它最初是一个命令：此时它可能仍然会失败，例如因为违反了某些完整性条件。
> 应用程序必须首先验证它是否可以执行该命令。
> 如果验证成功并且命令被接受，它将变成一个持久且不可变的事件。
> 
> 例如，如果用户试图注册某个特定的用户名，或在飞机上或剧院中预订座位，则应用程序需要检查用户名或座位尚未占用。
> 当检查成功时，应用程序可以生成一个事件来指出特定的用户名被特定的用户ID注册了，或者特定的座位已预留给特定的某个客户了。
> 
> 不允许事件流的消费者拒绝事件：当消费者看到事件时，它已经是日志中不可变的部分，并且可能已经被其他消费者看到。
> 因此，任何命令的验证都需要在它成为事件之前同步发生，例如，通过使用能够原子地验证命令并发布事件的可序列化事务。
> 
> 或者，预订座位的用户请求可以分成两个事件：第一个是暂时预约，第二个是确认预约后的单独确认事件。
> 这种划分允许验证过程异步进行。
> 
> ——DDIA

又是曾经的项目中，强行在命令中事件驱动，为了EDA而EDA，引以为鉴。

#### 事件溯源
与CDC相似。

## 4 战术

### 实体
有这么一个说法，人是其过往所有经历的总和。
在对象的世界里，实体是那些在整个生命周期中具有连续性的对象，它们的形式和内容可能发生根本改变，而我们需要对这些对象进行跟踪。
同时，为了有效地跟踪这些对象，必需定义它们的标识。因此实体也具有可变性及唯一性。
为了让开发者的关注点放在领域而不是数据上，DDD中的模型（实体、值对象）除了属性还需要有行为。

#### 标识稳定性
在多数情况下，我们都不应该修改实体的唯一标识，特别是当对象标识已经暴露给其它对象，被其它对象所持有时。

#### 验证
验证的主要目的在于检查模型的正确性，使得对象满足业务定义的约束。
Vernon列举了三个级别的验证：单个属性、单个对象的属性组合、多个对象，以及几个验证方法：领域对象的自封装前置条件检查、同包验证类或领域服务延迟验证。

### 值对象
用于描述或度量、不被关注连续性、描述改变时可用另一值对象替换的、大多时候不可变的对象，就是值对象。

对象在某一上下文可能会建模成值对象，但在其它上下文可能需要建模成实体。

Vernon还提出了标准类型，并建议在消费方的限界上下文中建模成值对象。
在实践中，通常这些标准是诸如ISO等组织制定的如币种这样的标准。

### 领域服务
领域服务是处理不适合放在实体和值对象的业务逻辑的。

### 模块
使用模块的主要原因是"认知超载"。模块内高内聚，模块间低耦合。模块表示了一个命名的容器，用于存放内聚的类、子模块。

模块应该和应该和领域概念保持协调一致，并且命名应该根据通用语言来命名。我们可以使用模块来避免创造一些微小的限界上下文。

在Java中，模块通常意味着包、OSGi的bundle或Jigsaw的模块。

### 聚合

#### Unit of Work模式
Unit of Work是Martin Fowler提出来的。
UoW对象维护了一些状态，用以跟踪事务中各个对象变更的情况。并在提交时通过调用Repository，来实现数据库的一次性访问。
更具体的见网络上的栗子。

#### 聚合
聚合是一组相关对象的集合，作为数据修改的单元。每个聚合都有一个根，根是一个特定的实体，是唯一能被外部对象持有对它的引用的元素。

DDD中的聚合是为解决复杂关联对象的一致性的，将（强）一致性圈定在聚合的范围内。也就是说，在设计聚合时，主要关注的是一致性边界，而不是创建一个对象树。
要找到大小合适的聚合，需要对领域有深入的了解，如对象之间的更改频率因素等。小的聚合，无法保证更大范围的一致性约束。大的聚合，会造成数据竞争导致失败的问题。

设计良好的限界上下文，一个事务只修改一个聚合实例。不同的聚合之间，可以通过领域事件达到最终一致。
虽然聚合内的实体可以引用另一聚合根，但Vernon建议通过引用聚合根ID值对象定位其它聚合，因而可以较大程度上避免聚合内部修改另一聚合的情况，进而实现一个事务只修改一个聚合。

### 领域事件

#### 时延
使用事件，比较重要的一个方面就是时延的问题，为了更好地理解事件时延，引用Vernon
> 在没有计算机之前业务是如何开展的，如今，将我们手中的计算机扔掉，我们的业务又将如何开展？也许最简单的基于纸张的系统也不见得有多好的最终一致性。因此，自动化的计算机系统也是有理由存在事件时延的。

### 工厂
使用工厂的动机：被创建的对象本身不适合复杂的装配操作，让客户端直接负责创建对象又会使客户的设计陷入混乱破，破坏对象的封装。

实体、领域服务也可以承担工厂的职责。

### 仓库
仓库是用来处理持久化相关的对象。
> 为每种需要全局访问的对象类型创建一个对象，这个对象就相当于该类型的所有对象在内存中的一个集合的"替身"。
> 通过一个众所周知的接口来提供访问。提供添加和删除对象的方法，用这些方法来封装在数据存储中实际插入或删除数据的操作。
> 提供根据具体标准来挑选对象的方法，并返回属性值满足查询标准的对象或对象集合，从而使实际的存储和查询技术封装起来。
> 只为那些确实需要直接访问的聚合根提供仓库。让客户始终聚焦于模型，而将所有对象存储和访问操作交给仓库来完成。

#### 仓库与工厂
工厂主要负责对象生命周期的开始，而仓库主要负责中间和结束。

仓库也可以委托工厂来处理客户查询，将存储的查询结果委托给工厂来重建对象。

对于复杂的查询，如需求涉及多个聚合的数据，可以使用用例优化查询的方法直接查询所需要的数据，然后将查询结果放在一个值对象中返回。
对于更复杂的情况，可以使用CQRS。

#### 仓库与DAO
仓库更偏向于对象，而DAO通常只是对数据库表的一层封装。

### 用户界面（层）
广义的用户界面可以包括所有本应用的用户，包括提供给下游系统的服务。

传统的数据传输对象DTO用来处理用户界面的数据，而不是将内部的对象直接暴露，这样可以起到隔离内外环境的作用。
然而，这种方式同样存在一定的问题——需要DTO组装器深度访问内部领域对象的状态。

一种改进的方法是将组装的职责放到聚合、实体，然后通过抽象的中介者实现用户界面数据的获取。
另外就是使用用例优化查询或CQRS。

DTO或者其它数据容器用在用户界面时的表达能力很多时候还是比较弱，此时可以增加用户界面层的职责，抽象出展现层模型。
这样展现模型根据视图所需不止提供了DTO的属性，还有一些行为，比如需要根据领域模型来展示一个按钮等（HATEOAS）。同时，展现层模型还可以提供JavaBean要求的getter和setter。

## Hibernate（JPA）

### 工程搭建
正常的Spring boot项目直接引入starter包spring-boot-starter-data-jpa和数据库驱动即可。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>me.liuweiqiang</groupId>
    <artifactId>jpa</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.13</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```
应用配置
```yaml
spring:
  datasource: # 使用默认的连接池
    url: jdbc:mysql://localhost:3306/idempotent
    username: lwq
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: validate # 一般的生产配置都会将DDL权限和DML权限分开
```

### 快速入门
照猫画虎，按 [官网教程](https://spring.io/guides/gs/accessing-data-jpa/) 即可，略。

### Hibernate手册
手册来自 [官方5.4](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html) 。
Hibernate支持xml和注解，同时还支持审计功能（Envers）、以及除了原生Hibernate API的JPA，这里以注解、JPA为例（毕竟引入的包为Spring Data JPA）。

![JPA与Hibernate](https://raw.githubusercontent.com/lvv9/lvv9.github.io/123c4f02c9f8e4b176fc2730e663baa1747a6440/pic/JPA_Hibernate.svg)

#### 领域模型
Tips：Hibernate模型可以不需要实现JavaBean规范，或者getter/setter可以是private的。

#### 值类型
值类型包含
- 基本类型
  javax.persistence.Basic注解的成员变量，如String等，可不写。
  可通过org.hibernate.annotations.Type显式指定序列化/反序列化类型。
  对于枚举类型，可通过javax.persistence.Enumerated或javax.persistence.Convert+javax.persistence.Converter实现。
  java.util.Date则通过javax.persistence.Temporal。
- 可嵌类型
  javax.persistence.Embeddable注解的类。
  被嵌入的对象引用时需使用注解javax.persistence.Embedded。
  由于一个对象可能嵌入多个相同类型的可嵌类型，且可嵌类型被多个不同的对象所引用而不应将表定义放在Embeddable类时，可使用javax.persistence.AttributeOverrides来显式指定。
  有时被嵌的对象声明的成员是抽象的，就需要org.hibernate.annotations.Target来指定具体的类。同时，可嵌对象可以通过org.hibernate.annotations.Parent注入被嵌对象。
- 集合类型

#### 实体
实体由javax.persistence.Entity注解提供实体配置，还提供javax.persistence.Table来显式指定表名。
同时，实体需要有public或protected或package的无参构造器。

##### 标识
围绕标识有许多不同的观点，个人的想法，是一个实体可以建多几个标识，以满足不同的需要：
1. 面向数据库的自增ID，纯粹为了数据库而建的，如MySQL这种，应用层无需关注，甚至SQL中都不必出现（Vernon也大概是这种观点，称为委派标识）；
2. 面向本上下文的实体创建的ID，因为对于本上下文而言，能较大程度上保证这种标识的稳定性，不受外界影响，同时不应依赖数据库生成；
3. 自然键。

通常1是主键（MySQL），其它两种是候选键。生成的键由javax.persistence.GeneratedValue指定策略。

##### Session（Hibernate）或EntityManager（JPA）
有四种状态描述了对象与Session的关系，transient（新建的对象）、managed（接管的对象）、detached（接管而又脱管的对象）、removed（删除的对象）。

##### 实现equals()和hashCode()
由于跨Session、标识生成时机、Set集合约定之间可能的冲突，因此基于2实现equals()和hashCode()是不错的想法。

##### 视图
可以用org.hibernate.annotations.Subselect建立Hibernate层的视图。

##### 复合键
复合键用javax.persistence.EmbeddedId，JPA要求EmbeddedId类需实现Serializable接口。

复合键不支持自动生成。

##### 自增的键
用javax.persistence.GeneratedValue指定，支持三种策略（AUTO则交给Hibernate选择）：
- IDENTITY 用在MySQL这种支持Auto Increment的，但在插入前无法知道序号是多少
- SEQUENCE 用在Oracle这种支持sequence的，Hibernate支持自动降级到IDENTITY
- TABLE 使用表建模sequence，较为通用，但并发等方面的性能较差

这些是框架的自动分配（注入），如果ID存在一定的业务逻辑，可使用org.hibernate.annotations.GenericGenerator或领域层显式地分配。

一般的实现基于性能考虑会将序列取出来后以n（n>1）为一个step来缓存。

#### 关联
Tips：如果启用了字节增强，双向的关联只需调用一个实体的方法。

##### 多对一
即javax.persistence.ManyToOne，"多"指的是 @Entity 实体，一般"多"的称为子实体，"一"的为父实体。

多对一关系是通过子实体的表列来维护的，可以使用javax.persistence.JoinColumn显式定义子实体的join列。

##### 一对多
javax.persistence.OneToMany，"多"的子实体可以@ManyToOne引用"一"的父实体，父实体也可以通过@OneToMany引用子实体。如果子实体没有@ManyToOne引用父实体，则称关系为单向的。
由于单向时子实体无相关表列来维护关系，因此需要创建中间表来维护关系。
而双向的关系需要父实体使用mappedBy参数来避免创建中间表，指定从子实体的表列推断出关系。
单向的一对多实现上没有双向的效率高，单向的关系更新时需要先删除所有关系，再新增需要的关系。
而双向的关系需要应用层保证两个方向的引用保持同步。
单向的多对一不存在这种问题。

org.hibernate.annotations.CascadeType只在父实体操作有效；orphanRemoval参数用来删除孤立的子实体。

##### 一对一
javax.persistence.OneToOne，单向的一对一从结构上看与单向的多对一相似，保存关系的称为子实体。双向的与上面类似，可以用mappedBy参数来避免冗余。
optional参数用来指明是否必需存在这种关系。

##### 多对多
与前面一样，多对多也有单向和双向，同时还可以用两对@OneToMany+@ManyToOne来替换@ManyToMany。

为了支持范型，可以用org.hibernate.annotations.Any和org.hibernate.annotations.ManyToAny。

##### Cascade
Cascade将父实体的变更传播到子实体，见javax.persistence.CascadeType。

#### 集合
Hibernate支持以下容器接口：
- java.util.Collection
- java.util.List
- java.util.Set
- java.util.Map
- java.util.SortedSet
- java.util.SortedMap
- org.hibernate.usertype.UserCollectionType

集合类型不允许嵌套，因此当可嵌类型被放入集合时可嵌类型的成员不应包含集合。可嵌类型集合将被持久化在无键值的表中。

对于值类型的集合，需要加注解javax.persistence.ElementCollection。

#### NaturalId
除了@Id的另外一个Id，不过可选是否可变。如果是可变的，需要@NaturalId(mutable = true)。
如果是可变的，Hibernate会在Session和二级缓存范围内维护NaturalId与Id的映射。

这也启示我们，对于ID可能会变化的时候，可以在设计之初预留一对多的映射。

#### 动态映射
Hibernate支持Map+mapping文件来运行时动态地映射。

#### 继承
存在多种继承方式：
1. javax.persistence.MappedSuperclass，不同子类不同的表，基类无表，无法多态查询
2. javax.persistence.Inheritance(strategy = InheritanceType.SINGLE_TABLE)，不同子类相同的表，DTYPE作为默认的类型列
3. javax.persistence.Inheritance(strategy = InheritanceType.JOINED)，多态查询时需要join父表与所有子表
4. javax.persistence.Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)，一个类一个表（包括基类），多态查询时使用UNION

#### EntityManager（Session）
保存：javax.persistence.EntityManager#persist 与 org.hibernate.Session#save(java.lang.Object)，transient状态变为managed。

删：javax.persistence.EntityManager#remove 与 org.hibernate.Session#delete(java.lang.Object)，managed变为removed。

非初始化加载：javax.persistence.EntityManager#getReference 与 org.hibernate.Session#load(java.lang.Class<T>, java.io.Serializable)，managed状态。

带初始化加载：javax.persistence.EntityManager#find(java.lang.Class<T>, java.lang.Object) 与 org.hibernate.Session#get(java.lang.Class<T>, java.io.Serializable)，managed状态。

实体状态更新后无需再进行调用，在flush的时候会被自动被更新；更新操作默认会更新全部列，可以使用org.hibernate.annotations.DynamicUpdate来启动动态更新。

刷新：javax.persistence.EntityManager#refresh(java.lang.Object) 与 org.hibernate.Session#refresh(java.lang.Object)，当子实体是transient时，刷新会抛异常EntityNotFoundException。

reattach：org.hibernate.Session#lock(java.lang.Object, org.hibernate.LockMode) 与 org.hibernate.Session#saveOrUpdate(java.lang.Object)
> It does not mean that an SQL UPDATE is immediately performed. It does, however, mean that an SQL UPDATE will be performed when the persistence context is flushed since Hibernate does not know its previous state against which to compare for changes. If the entity is mapped with select-before-update, Hibernate will pull the current state from the database and see if an update is needed.

合并：javax.persistence.EntityManager#merge 与 org.hibernate.Session#merge(java.lang.Object)，detached与managed对象合并。

detach：javax.persistence.EntityManager#detach、javax.persistence.EntityManager#clear 与 org.hibernate.Session#evict、org.hibernate.Session#clear。
javax.persistence.EntityManager#contains用来验证。

##### 缓存
除了EntityManager（Session）级别的缓存，Hibernate还存在二级缓存。
默认不开启。

##### 异常
当Hibernate层发生异常时，对象可能处于中间态，此时需要将Session关闭：javax.persistence.EntityManager#close。

##### 刷出flush
因为Session跟踪着对象的状态，因此可以将Session看作transactional write-behind cache。JPA刷盘策略：
- AUTO，prior to committing a Transaction、prior to executing a JPQL/HQL query that overlaps with the queued entity actions、before executing any native SQL query that has no registered synchronization
- COMMIT

Hibernate支持更多：
- ALWAYS
- MANUAL

#### 并发控制

##### 乐观锁
Tips：如果实体版本号为空，会被认为处于transient状态。

使用javax.persistence.Version，同时版本号属性需要是以下类型：
- int or Integer
- short or Short
- long or Long
- java.sql.Timestamp
- org.hibernate.usertype.UserVersionType实现

##### 悲观锁
javax.persistence.EntityManager#find(java.lang.Class<T>, java.lang.Object, javax.persistence.LockModeType, java.util.Map<java.lang.String,java.lang.Object>)

#### 查询

##### Direct fetching vs. entity queries
通过JPA，有多种查询数据的方式：
- 前面提到的javax.persistence.EntityManager#find(java.lang.Class<T>, java.lang.Object)，官方称为Direct fetching
  使用这种方式获取数据时，如果存在多对象EAGER关联，那么生成的SQL会包含LEFT JOIN
- javax.persistence.EntityManager#createQuery(java.lang.String, java.lang.Class<T>)，官方称为Entity query（不查询Session缓存）
  String参数表示一个HQL/JPQL，如果HQL/JPQL无JOIN同时存在多对象EAGER关联，Hibernate会首先查询1个父实体，然后查询N个子实体，造成1+N的问题
- javax.persistence.EntityManager#getCriteriaBuilder + javax.persistence.EntityManager#createQuery(javax.persistence.criteria.CriteriaQuery<T>)
  与上面这种相似，但是通过Criteria保证类型安全
- find()在懒加载的时候也会存在1+N的问题，且这种查询方式是静态的（即编译后无法改变），为了使运行时的性能得到改善，可以用带Map参数的find()

##### FetchType
除了JPA的FetchType.LAZY和FetchType.EAGER，Hibernate还提供org.hibernate.annotations.Fetch + FetchMode.SUBSELECT，通过谓词IN来避免1+N。

##### Steam
JPA提供了JPQL流处理相关的方法javax.persistence.Query#getResultStream，底层默认实现仅是getResultList().stream()的缩写。

Hibernate重写了这个方法org.hibernate.query.Query#getResultStream，并在org.hibernate.query.internal.AbstractProducedQuery#stream使用了ScrollableResults。
由于stream是java.lang.AutoCloseable的，且操作ScrollableResults是IO方法，官方提示放在try-with-resources语句中。

### Spring Data JPA

#### 手册
手册来自 [官方2.4.13](https://docs.spring.io/spring-data/jpa/docs/2.4.13/reference/html/#reference) 。

#### Spring Data Repository
Spring提供了抽象的Repository完成核心的操作。

完整的搭建包含四个步骤：
1. 声明一个org.springframework.data.repository.Repository，包括继承或注解org.springframework.data.repository.RepositoryDefinition（注解更解耦）；
2. 声明相应的方法，当采取注解的方式时可以复制Repository子类的方法；
3. 添加Spring配置，spring boot starter可省略；
4. 客户端调用。

如果继承层次大于1，可通过org.springframework.data.repository.NoRepositoryBean避免中间Repository的生成。

Spring默认会根据方法名来生成相应的查询，同时参数还可以包含org.springframework.data.domain.Pageable等来执行分页和排序等。
其中的org.springframework.data.domain.PageRequest实现page以0开始计数。

Spring同时支持Optional和null返回。
以及支持Stream返回，同Hibernate，需要close()。

##### 自定义实现
Spring除了提供丰富的Repository接口外，还支持自定义部分实现。
在声明好接口及相应的实现后（实现类需要配置为Spring Bean），通过Repository extends该接口即可。

可通过org.springframework.data.jpa.repository.config.EnableJpaRepositories#repositoryImplementationPostfix来指定实现类的后缀。

##### Named Query
Spring还支持通过方法名绑定JPA的Named Query。
或者通过在Repository查询方法上添加org.springframework.data.jpa.repository.Query实现，这种方法同时还支持Native Query。

##### 投影Projection
投影即只select部分属性，只需将返回声明成对应属性的getter接口即可，支持嵌套。

##### 锁
org.springframework.data.jpa.repository.Lock

#### 领域事件
通过org.springframework.data.domain.DomainEvents与org.springframework.data.domain.AfterDomainEventPublication来发布、清除事件。

个人想法：
为了保证事件的可靠性，我们应该同时将事件也持久化，并与实体状态改变的持久化保持原子性（由于事件机制不是领域的一部分，因此这样做是被允许的）。
事件表同时增加状态字段，保证发布、投递后更新状态。

事件监听建议使用org.springframework.transaction.event.TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)，以保证事务Commit后被订阅。

如果listener是TransactionPhase.BEFORE_COMMIT的，则可能需要@Transactional(propagation = Propagation.REQUIRES_NEW)，因为同步监听器的Session是共享的，这样可以保证实体持久化前不被读取。

### 演示项目
https://github.com/lvv9/jpa
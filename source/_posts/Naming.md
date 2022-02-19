---
title: Software Complexity：Naming
date: 2022-01-01 15:57:26
tags: ['翻译']
categories: 技术
---

原文：https://alexoliveira.cc/software-naming.html

并非逐句逐词，而是根据自己的理解进行了翻译。

<!-- more -->

![The Magic Acid, or the art of naming](https://alexoliveira.cc/images/posts/software-naming/naming.png)

> There are only two hard things in Computer Science: cache invalidation and naming things.
>
> Phil Karlton

当我们尝试阅读和理解软件系统时，我们希望代码通顺，希望它能用业务领域中的词汇，清晰而准确地传达意图和关系。代码虽然只写一次，但是会被阅读很多次。

烂代码使人迷惑，它是在 "代码对作者有意义，对读者没意义" 的假设或认知下编写的。如果一段代码需要在别人的帮助下才能被理解，或者需要花费与之不相称的巨大努力后才能看懂它在做什么，那就是烂代码。

我们如何才能写出好的代码呢？

不管想在任何领域取得成功，第一步就是要对领域内的组成元素，以及元素对人和环境的影响进行充分的了解。优秀的电影导演在看了无数个小时的电影后，能将电影领域中的元素和规律总结出来，然后操纵它们来有效地讲故事。同样的道理也适用于小说家、音乐家、棋手。

类似的，程序猿应该也能够通过观察业务领域并发现其元素和规律。

领域中的元素有些很明显，但有些则不然，这就是问题所在：为这些元素起一个名字并非易事，因为有些元素在现实中并不存在；亦或者是有些元素存在，但缺少每个人都能理解的名称。

有些命名使元素个性化，而有些则将元素聚合在一起；有些命名表示抽象，而另一些则使事物具体化；有些命名表明了角色，有些表明性质...... 

许多人取不好名字只是因为缺乏灵感。

# 给组件和元素命名

元素的类型以及元素间的关系有一个完整的宇宙。有些是在同一层级（如左手、右手），有些则是包含的关系（如公司、员工）。有些元素依赖于另外的元素（如支付、支付方式）。而有些元素则会产生其他的元素（如账单、付款）。

为了揭示概念和它们的名字，我们从问一些简单的问题开始。

## 问题1/3

这是什么房间？

![A couch, what room does it belong to?](https://alexoliveira.cc/images/posts/software-naming/couch-in-what-room.png)

通过这个家具来看，很可能客厅。

基于一个组件，我们知道我们所在的房间的名称，这很简单。

## 问题2/3

这是什么房间？

![A toilet, what room does it belong to?](https://alexoliveira.cc/images/posts/software-naming/toilet-in-what-room.png)

根据这个物体，我们可以比较肯定的说这是洗手间。

看到规律了吗？房间的名称是一个标签， 它定义了房间里的内容。有了这个标签，我们不需要查看容器内部就能够知道里边有什么元素。这使得我们能够建立起我们的第一个推论：

**推论1: 容器名是关于其元素的函数（container name is a function of its elements）**

注意，这其实就是[鸭子类型](https://zh.wikipedia.org/wiki/%E9%B8%AD%E5%AD%90%E7%B1%BB%E5%9E%8B)：它有床吗? 那很可能是卧室。

反过来也是正确的：如果我们谈论的是一间卧室，那么它很可能有一张床。这使我们能够定义第二个推论：

**推论2: 我们可以根据容器名推断出它有哪些元素（we can infer components based on the container name）**

很明显，我们有了一些规则，让我们尝试把它们应用到隔壁的房间。

## 问题3/3

这是什么房间？

![What room has a toilet and a bed?](https://alexoliveira.cc/images/posts/software-naming/toilet-bed.png)

如果我们的系统是一所房子，那么在同一个房间里的一张床和一个马桶使得这个房间的定义变得非常非常模糊。

这时使用推论1和推论2好像就很难给房间命名，或许我们可以给他命名为 "奇怪室"。

根据我的经验，大多数时候命名的难题都是由于糟糕的建模造成的。当软件中的元素没有被连贯且合乎逻辑地被引入时，有一个好的命名几乎是不可能的。**由于排期很紧，开发者被迫先想到什么就交付什么。**

在家里，我们把具有相同功能、目的和意图的元素放在一起。这使得组织变得更容易，然而通过混淆这些元素的职责，我们不能确定作者想要什么元素或者如何使用这些元素，从而阻塞了流程。

**推论3: 容器定义的清晰程度与其组件的紧密程度成正比（the clarity with which a container is defined is proportional to how closely related its components are）**

![Clarity vs relation](https://alexoliveira.cc/images/posts/software-naming/clarity-vs-relation.png)

当组件间相互关联时，就更容易找到一个好的命名。而当事物间没有关联时，起名就变得很难。

这里所说的 "关联" 可以是它们的功能、目的、策略、类型或其他的任何方面。在讨论**标准**之前，"关联" 这个词本身没有多大意义。我们在后边的章节就会讲到它。

请注意，如果我们的系统是一座监狱，而不是一所房子，这些假设就会颠倒过来。在这个场景下，读者很容易就能认出这个房间是一件牢房。这就引出了另外一个重要推论：

**推论4: 一个容器的定义受上下文约束（a container definition is constrained to its context）**

当我们只关注容器的定义时，这个推论是显而易见的，但其实还有一个问题：系统常常没有定义上下文。

当大多数代码结构是打平的时，上下文的定义就会丢失，从而淡化了容器之间的区别。这导致 "提示某段代码该去哪里去看的提示不存在"，所以感觉代码像是随机的。

可以想象，如果所有的业务逻辑都打平在一个类中实现，那基本是不可读的。

由于缺少层级的概念，这样的实现相当于变相鼓励了开发人员在任何地方添加代码。就像  Alan Kay 所说的:

> 如今的大多数软件很像埃及金字塔，由数百万块砖头堆砌而成，没有任何的结构完整性，只是由蛮力和成千上万的奴隶完成的。
>
> Most software today is very much like an Egyptian pyramid with millions of bricks piled on top of each other, with no structural integrity, but just done by brute force and thousands of slaves.

**例1: HTTP domain and a car**

HTTP 是一个有 request 和 responses 的域。如果我们往里边放进了一个叫做 "汽车" 的组件，我们就不能再称它为 HTTP 了。在这种情况下，它变得令人困惑。

``` java
public interface WhatIsAGoodNameForThis {
	
	/* methods for a car */
	public void gas();
	public void brake();

	/* methods for an HTTP client */
	public Response makeGetRequest(String param);	
}
```

![](https://alexoliveira.cc/images/posts/software-naming/car-http.png)

**例2: Coupling through words**

一个常见的模式是在类名后加个 "Builder" 或者其他 er 结尾的单词，比如 SomethingBuilder、UserBuilder,、AccountBuilder、AccountCreator、UserHelper、JobPerformer......

![UserBuilder](https://alexoliveira.cc/images/posts/software-naming/builders.png)

通过类名来判断，我们可能得出三个推论：

首先，类名中的动词 build 意味着它是一个函数（当然实际上 build 只是类名中的片段）。函数负责做事情，类则包含实体和上下文。函数不是实体，如果没有很好的定义实体，代码库很快就会演变成过程代码，因为它没有充分利用 "class pattern" 的初衷（面向对象）。

其次，它有两个内部的、隐式的实体，即 User 和 Builder，这意味着可能违背封装原则。我们的意思是 User 不应该负责生成新的 User。

第三，它意味着 Builder 可以访问 User 的内部的函数，因为它们毕竟是互相绑在一起的。

我已经看到整个代码库都充斥着这种代码。有一些复杂的模式试图解决这个问题，比如 **Factory Pattern**。许多时候它们是很有用的，但当我们遇到建模问题时，我们最好是修复它，而不是给它打个绷带/创可贴。

**例 3: Base**

让我们看一个真实代码的例子。它是 Ruby gem 的代码 [I18n](https://github.com/svenfuchs/i18n/blob/master/lib/i18n.rb)（为了简洁起见，只列出类和方法名）

``` ru
class Base
  def config
  def translate
  def locale_available?(locale)
  def transliterate
end
```

![Base](https://alexoliveira.cc/images/posts/software-naming/base.png)

在这个例子中， ***Base*** 并没有传达出什么意思。它可以 *config*，可以 *translate*，以及可以确定 *local* 是否可用。它做了一些不同的、不相关的事情。

**例4: names guiding design**

当我们谈到命名如何指导我们的设计时，Discourse 上有几个例子，其中一个很有意思。

``` ruby
class PostAlerter
  def notify_post_users
  def notify_group_summary
  def notify_non_pm_users
  def create_notification
  def unread_posts
  def unread_count
  def group_stats
end
```

"PostAlerter" 的命名暗示了它的功能可能是："alert" 某人关于某个 "post"。然而，*unread_posts*、*unread_count*、和 *group_stats* 显然是要处理其他东西，这使得这个类名不适合它所做的事情。将这三个方法转移到一个名为 *PostsStatistics*  的类中，使得这一切变得更加清晰和可预测。

``` ruby
class PostAlerter
  def notify_post_users
  def notify_group_summary
  def notify_non_pm_users
  def create_notification
end

class PostsStatistics
  def unread_posts
  def unread_count
  def group_stats
end
```

**例5: ambiguous names**

Spring 框架中也有几个例子，一个组件做了太多的事情，因此需要起一个类似我们上文中提到的 "奇怪室" 的名称。

可以看[这个例子](https://docs.spring.io/spring-framework/docs/2.5.x/javadoc-api/org/springframework/aop/config/SimpleBeanFactoryAwareAspectInstanceFactory.html)：

```jav
class SimpleBeanFactoryAwareAspectInstanceFactory {
  public ClassLoader getAspectClassLoader()
  public Object getAspectInstance()
  public int getOrder() 
  public void setAspectBeanName(String aspectBeanName) 
  public void setBeanFactory(BeanFactory beanFactory)
} 
```

**例6: good naming, for a change**

不好的命名我们已经说的够多了，我们再来看看好的命名。

 D3 的 [arc](https://github.com/d3/d3-shape/blob/master/src/arc.js) 命名就很好，例如：

```js
export default function() {
  /* ... */
  arc.centroid     = function() { /* ... */ }
  arc.innerRadius  = function() { /* ... */ }
  arc.outerRadius  = function() { /* ... */ }
  arc.cornerRadius = function() { /* ... */ }
  arc.padRadius    = function() { /* ... */ }
  arc.startAngle   = function() { /* ... */ }
  arc.endAngle     = function() { /* ... */ }
  arc.padAngle     = function() { /* ... */ }
  return arc;
}
```

这些方法中的每一个都是完全有意义的：它们都是以弧的名称命名的。我喜欢下边这张图，因为它太简洁了。

![arc](https://alexoliveira.cc/images/posts/software-naming/arc.png)

---

接下来讲讲当我们起名遇到麻烦时，可以怎么办。

### 办法1: 拆分

![Divide and... name](https://alexoliveira.cc/images/posts/software-naming/divide-and-name.png)

**何时使用**：你不能为一个类或者组件找到一个好的名字，但是你已经有了一些独立的概念，并希望为它们的分组找到合适的名称。

这个方法由两部组成：

1. 明确已有的概念
2. 把它们分开

在 "马桶 + 床" 的场景中，我们把不同的东西分开：把床推到左边，把马桶推到右边。现在我们有了两个独立的东西，我们终于可以用自然的方式来推理（房间的名称）了。

当你不能为某个东西找到一个好名字时，很可能是因为有不仅一个东西摆在你面前。而且，正如你现在知道的，给多个事物起名字是很难的。当遇到麻烦时，试着找出你面前的东西由哪些部分和行为组成。

**举例**

我们有一个未命名的类，它里边包含了 *request*、*response*、*headers*、*URLs*、*body*、*caching* 和 *timeout*。把所有这些从主类中分离出来，只剩下组件 `Request`、 `Response`、 `Headers`、 `URLs`、 `ResponseBody`、 `Cache`、 `Timeout`。如果摆在我们面前的只是这些类的名称，我们可以相当肯定地假设我们正在处理一个 web 请求。

`HTTPClient` 这个命名很适合 web 请求组件。

当代码难写时，不要直接考虑整体，先想想部分。

### 办法2: 发现新的概念

![Compound concept](https://alexoliveira.cc/images/posts/software-naming/compound-concepts.png)

**何时使用**：当一个类不简洁或不条理清晰的时候。

发现新的概念需要有业务领域的知识。当软件中使用与业务相同的术语时，就形成了一种普遍存在的语言（Evans，2003），这种语言允许来自不同专业领域的专业人士使用相同的术语。

**例1: 将组件封装到一个新概念中**

几年前，一家公司差点失去一份大合同，因为团队在发布新功能和修复 bug 方面进展缓慢。

这个电子商务市场通过多个支付网关，根据不同国家的不同规则向学生收取费用，需求相当复杂。

当我看到这个收费代码  `PaymentGateway` 时，我震惊于它是多么地复杂，有多个依赖项，包括：`User`、 `UserAddress`、 `CreditCard`、 `BillingAddress`、 `SellerAddress`、 `LineItems`、 `Discounts`，等等。它的构造函数是巨大的，这种复杂性使得添加新规则变得困难，因为修改一个东西就会破坏其他东西，并且需要我们更改所有网关适配器。

这个问题已经超出了支付的范畴。通过在 messaging class 中再次聚合这些数据的方式，他们向学生发送了电子邮件。技术支持部门有自己的数据大盘屏幕，他们第三次聚合了这些数据，除了这个特定的地方使用了一个名为 `Aggregator`  的类（如果没有上下文，这个词就没有意义）。我们必须要点什么来解决这个架构上的障碍。

为了解决这个问题，我先是做了一个思维练习。以下是思维的过程：

> （假设我就身处在 `PaymentGateway` 的世界里，在我面前是我需要你( `PaymentGateway`)为我收费的细节。如果这是一张桌子，我会把这些文件整理好，我可能会把这些文件称为发票（Invoices）。所以，如果我创建一个名为 `Invoice` 的类会怎么样呢？它只不过是所有这些细节的集合，这样网关就不需要知道这些规则是如何起作用的，因为 `Invoice` 类会搞定这些？不用注入一大堆对象，我只给你一个就行？
>
> Here I am, with these details about things I need you (the `PaymentGateway`) to charge for me. If this was a desk, I’d have these papers organized and I’d probably call them Invoices. So what if I created one class called `Invoice`, which is nothing more than the aggregation of all these other details, such that the gateway doesn’t need to know how those rules are done because `Invoice` will? Instead of injecting a million objects, I just hand one over to you?

*Invoice* 一词在系统中的任何地方都没有使用。我们花了一个月的时间进行重构，一旦我们完成了重构，我们就能够更快地修改软件。

Invoice 是一个 "概念" 的一个很好的例子，它是来自许多源的数据的集合，大多数人都知道它是什么。最后的解决方案是通过外观模式](https://en.wikipedia.org/wiki/Facade_pattern)把 `Invoice` 类单独注入到网关中，并隐藏了其他类和对象。

好的命名不仅仅是使用华丽的辞藻，而是要准确地写出需要表达的内容。

**例2: 基于业务领域的变体名称（另一种叫法）**

在一个新开发的拼车项目中，我们从头开始设计了这个系统。在研究交通解决方案的过程中，描述"某个人在某一天从出发地到目的地的一段旅程"的合适词汇是 `trip`，而这类人则被称为 `ride`。我们发布了词汇表，这样公司的其他人就可以讨论和共享相同的通用语言。

在项目推出后，我们的客户总是把 `trip` 叫成 `rides`。很快我们就遇到了问题，在经历了巨大的痛苦之后，我们决定是时候重构了，于是我们把 `trips` 重构成 `rides`，把 `rides`重构成 `carpools`。这解决了一家公司说两种不同语言的问题。

**例3: 抽象级别**

![](https://alexoliveira.cc/images/posts/software-naming/abstract-canvas.png)

一个人说”移动右腿、然后左腿、然后右腿“；另一个人说”走路“。两者的意思相同，但后者更抽象。

理想情况下，随着代码越来越接近公共 API，它就越努力匹配业务上的命名方式。而当它接近数据库甚至金属时，它则更倾向于使用与上下文相关的物理上的命名。在这两者之间， 有着从更多抽象到更少抽象的变化趋势。

在我们公司，业务人员会说 *post Tweet*，因此在公共 API 中，像 `postTweet()`这样的名称比 `makeHttpRequest()` 这样的名称更有意义。在一个有更多技术服务的公司里，后者则更合适。

其次，需要考虑特殊性。`postTweet()`是非常具体的，而 `makeHttpRequest()`则非常通用，它可以应用于 Facebook 或其他基本上任何涉及 HTTP 的东西。一个通用的名字可以很容易地被重复使用，但代价是含糊不清。**这就解释了为什么框架代码与业务软件代码如此不同**。

**例4: 泛化**

很久以前，CMS 有  *news*、 *history*、 *videos*、 *articles*、 *pages* 等数据库表，它们中的大多数都有相同的列：*title*、*summary* 和 *text*。不同的是，*videos* 表有额外的属性，比如如 *url(嵌入 YouTube )*；*history* 表有着 *date* 属性，这样页面就会按年显示历史时间列表。所有这些表看起来都像是副本，各处都有一些细微差异，添加新功能需要重新编写大量的样板代码。

我将所有这些表聚合成一个名为 `contents` 的外键，这个外键指向了 `sections` 表，`sections` 表中包含了 news、history、videos 等。现在，只写一个 `contents` 代码就足够了。

多年后，一位朋友不得不编写一个小型的 CMS，我建议他采用相同的方法。

一旦完成了用于管理内容的表单，实现任何内容所需的时间是原先所需时间的 1/N，因为对于相同类型的每个新部分，它都已经完成了。

**通过给它另一个名字使其泛化可以提高开发效率**。

### 办法3: 分组的标准（对应上文所说的 "关联" 标准）

**何时使用**：当各组件命名很好，但彼此间不搭配的时候

组件可以根据各种标准进行分组，包括物理性质、经济、情感、社会，以及在软件中最常用的标准——功能。

相框是根据情感因素来分组的，产品是根据经济动机来分组的。沙发和电视放在同一个房间里，它们是根据功能标准组合在一起，因为都有相同的功能或目的——提供休闲。

在软件世界里，我们倾向于把组件按照功能分组。如果把你的项目文件列表列出来，你可能会发现其中有 `contollers/`、`models/`、`models/`、`adapters/`、`templates/` 等。 但是，有些时候这些分组可能让你感觉不那么舒适，这时就是重新评估模块间的结构的最佳时机了。

**例: 根据策略分组**

有一个用于自动把用户操作归档的库，它基于代码生成一个规范文件(例如 API Blueprint)， lints 文件(以保证格式正确)，并上传到云(例如 S3)。

基于文档的格式，将自动做出各种后续决策。选择 Blueprint API 将会使用不同的 linter、不同的 tester 以及不同的 API 元素转换器。在这里，根据一个输入将所有这些不同的功能分组的关键是*策略*。

因此，这个库包含了一个名为 Strategy 的模块，用于将文件格式、检查器、文档测试器和存储程序组合在一起。这使得这个库能够将 uploaders、parsers 和 command-line 等普通的文件操作与业务的核心策略分开。

# 利用上下文

每个应用都有不同的上下文，以及其中的每个模块、每个类、甚至每个功能。仅 User 一词就可能有多个语义，比如它可以表示系统的用户，但也可能是数据库表，或者第三方服务的身份凭据。

`lib/billing/user` 与 `lib/booking/user` 不同，但他们都是 `user`。

![user](https://alexoliveira.cc/images/posts/software-naming/contexts.png)

假设每个容器（比如模块）都是一个桶。在他们内部，组件与外部世界隔离。你可以随意给这些类命名，这使得不需要绞尽脑汁地为一个普通的事物寻找深奥的名字。

微服务（许多孤立的桶）比单体结构（一个大桶，里边有许多小桶）更有说服力的一点是，它强制约束了每个服务的职责，因为这样你就不能轻易地把完全不相关的东西放在一起。 Billing 存在于 BillingApp 中， booking 存在于 BookingApp 中，等等。

在单体架构中，虽然这些各自的服务名称可以是简单的模块名称，但并不是每个人都有保持整洁的原则。

**例: 命名空间**

Mark 正在开发一个广告平台，需要生成成千上万则广告，然后将其发送给 AdWords、Facebook 和 Bing，所有这些都通过图形用户界面进行管理。

Mark 从一个叫做 Ad 的实体开始，这个实体很快就变得臃肿起来。AdWords 的广告有 `headline_part1` 和 `headline_part2`，而 Facebook 没有，而 Bing 只有 `headline`。他需要想办法分割他的实体。他思考了不同的上下文，以及如何利用语言的命名空间的特性来表达这一点。他设计出了下边的结构：

- `Adwords::Ad`: 代表 Adwords 中的一个广告对象，它有 Adwords 特有的属性，逻辑可以包含在这个类中
- `Facebook::Ad`: 与之前一样，它有 Facebook 特有的属性以及逻辑
- `Bing::Ad`: 与上边一样
- `RemoteAdService::Ad`:  这是`Adwords::Ad`、 `Facebook::Ad`、 `Bing::Ad` 与系统其他部分之间的服务接口。这意味着这三个类有着相同的 public API，从而使系统可以利用多态的特性
- `Database::Ad`: 这是 `ads` 表的 ORM，它使用 ActiveRecord、DataMapper 或其他自定义的方式来实现
- `GUI::Ad`: 表示在 UI 中显示广告所需要的属性。它可能具有 "展示" 和 "国际化" 的功能
- `API:Ad`: 广告的 HTTP endpoint 有着自己的自定义属性，因此这里有一个序列化的逻辑是合理的

根据上下文，单词可以有不同的含义，当我们利用上下文时，我们可以为组件选择更简单的单词。

在这个例子中，我们不需要做任何复杂的动作来找到这些组件的名称， 因为它们都是一个东西——广告（ad）。

# 无意义的命名和旧词新义

随着时间的推移，名字不断演变，获得了新的意义。

**Helper:** helper 是支持应用程序主要目标的函数。但是，定义应用程序的主要目标的标准是什么？应用程序中的一切都应该支持应用程序的主要目标。

在实践中，它们被集中在一个不自然的分组中，以提供一些复杂的、常用的操作的可重用性。它们往往会有特性依恋（[Feature Envy](https://sourcemaking.com/refactoring/smells/feature-envy)）问题，因为需要访问其他组件的内部数据才能工作。它们是找不到合适的命名时的借口。

**Base:** 在很久以前的 c# 中，命名为 *Base* 的类是一种约定，即在没有更好的名称时可以使用，从而被其它类继承。举个例子，*Automobile* 和 *Bicycle* 的父类是 *Base* 而不是 *Vehicle*。尽管微软建议避免使用这个名字（Cwalina，2009），但这个词还是渗入进了 Ruby 的世界，最显著的例子是 *ActiveRecord*。直到今天，我们仍然将 Base 视为开发人员找不到类名时起的类名。

![Base](https://alexoliveira.cc/images/posts/software-naming/base.png) 

Base 的变体包括 Common 和 Utils。例如， [JSON](https://github.com/flori/json/blob/65297fbae1e92e26fdde886fe156bac322977db2/lib/json/common.rb) Ruby gem *Common* 类具有  *parse*、 *generate*、 *load* 和 *jj* 方法，但这里的 common 真正的意思是什么？

**Tasks:** 在 JavaScript 社区中有一个调用异步函数（*tasks*）的浪潮。它从 task.js 开始，这个术语甚至在原始库不存在的时候也被使用。



你可能解释说，某个命名虽然没太有意义，但团队中的每个人都理解它的意思。

那就好。

但是，如果有某个新人加入这个团队，并认为这个很久以来就存在的命名是一坨垃圾时，你感觉如何？





我曾经参与过一个项目，其中一个类的名字是... 你们猜猜看...

**Atlanta（亚特兰大）**

是的， Atlanta！ 草他妈的 Atlante！没人知道，也没人告诉我为什么会叫这个名字！

## 沟通

> 现实存在于人类的思想中，而不是其他任何地方。
>
> “Reality exists in the human mind and nowhere else.” George Orwell  

我相信，善于沟通的做法是一种利他主义行为，我们在提高沟通技能上所付出的努力与我们对他人的关心程度有关。我们希望人们容易被（我们）理解，我们希望消除摩擦和阻碍。

其次，我们希望别人理解我们。通过认同 "接收方能收到消息是发送方的责任"，我们构建了一个共情的环境。这是双赢的局面。没有任何借口可以不刻意练习我们的沟通技巧——除非你生活在丛林中。

通过写作，我们可以提升阅读能力。而同理心的练习可能会让人筋疲力尽。但没办法，生活就是这样，熟能生巧。

# 参考文献

Cwalina, Krzysztof. 2009. *Framework Design Guidelines: Conventions, Idioms, and Patterns for Reusable .NET Libraries, Second Edition*. Boston: Pearson Education, Inc. 206.

Evans, Eric. 2003. *Domain-Driven Design: Tackling Complexity in the Heart of Software*. Boston: Addison-Wesley Professional.




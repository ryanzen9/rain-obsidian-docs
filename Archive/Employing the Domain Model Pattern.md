URL: https://learn.microsoft.com/en-us/archive/msdn-magazine/2009/brownfield/employing-the-domain-model-pattern
上次编辑时间: 2026年4月12日 17:09
创建时间: 2025年10月28日 16:30
标签: 好文记录
状态: 完成
目标追踪: 架构设计 (https://www.notion.so/290a89f7e47d8002bcf0ced466d98de7?pvs=21)

![](https://learn.microsoft.com/en-us/media/open-graph-image.png)

---

August 2009  2009 年 8 月

Volume 24 Number 08
第 24 卷 第 08 期

By [Udi Dahan](https://technet.microsoft.com/en-US/mt149362?author=udi%20dahan) | August 2009
Udi Dahan | 2009 年 8 月

Expand table  展开表格

This article discusses:  这篇文章讨论了：

- Domain model pattern  领域模型模式
- Scenarios for using the domain model pattern
使用领域模型模式的场景
- Domain events  领域事件
- Keeping the business in the domain
将业务保持在领域内

This article uses the following technologies:
本文使用以下技术：
Domain Model Pattern  领域模型模式

*In this article, we’ll go through the reasons to (and not to) employ the domain model pattern, the benefits it brings, as well as provide some practical tips on keeping the overall solution as simple as possible.*
*在本文中，我们将探讨采用领域模型模式的原因（以及不采用的原因），它带来的好处，并提供一些实用技巧，以使整体解决方案尽可能简单。*

## Contents  内容

What Is It?  它是什么？
Reasons Not to Use the Domain Model
不使用领域模型的原因
Technology  技术
Reasons to Use the Domain Model
使用领域模型的原因
Scenarios for Not Using the Domain Model
不使用领域模型的场景
Scenarios for Using the Domain Model
使用领域模型的场景
More Complex Interactions
更复杂的交互
Cross-Cutting Business Rules
跨领域业务规则
Domain Events and Their Callers
领域事件及其调用者
Explicit Domain Events  显式领域事件
Testability  可测试性
Commands and Queries  命令和查询
Keeping the Business in the Domain
将业务保持在领域内
Concurrency  并发性
Finding a Comprehensive Solution
寻找一个全面的解决方案
Works Cited  参考文献

If you had come to me a few years ago and asked me if I ever used the domain model pattern, I would have responded with an absolute "yes." I was sure of my understanding of the pattern. I had supporting technologies that made it work.
如果几年前你问我是否使用过领域模型模式，我会毫不犹豫地回答“是”。我确信自己理解这个模式。我还有支持这项技术的工作。

But, I would have been completely wrong.
但是，我完全错了。

My understanding has evolved over the years, and I've gained an appreciation for how an application can benefit from aligning itself with those same domain-driven principles.
多年来，我的理解发生了变化，我逐渐认识到应用程序如何从遵循这些相同的领域驱动原则中受益。

In this article, we'll go through why we'd even want to consider employing the domain model pattern (as well as why not), the benefits it is supposed to bring, how it interacts with other parts of an application, and the features we'd want to be provided by supporting technologies, and discuss some practical tips on keeping the overall solution as simple as possible.
在这篇文章中，我们将探讨为什么我们甚至想要考虑采用领域模型模式（以及为什么不想采用），它据称能带来的好处，它如何与应用程序的其他部分交互，以及支持技术应提供的功能，并讨论一些实用的技巧，以尽可能保持整体解决方案的简单性。

What Is It?  它是什么？

The author of the domain model pattern, Martin Fowler, provides this definition (Fowler, 2003):
领域模型模式的作者马丁·福勒（Martin Fowler）给出了这个定义（Fowler，2003）：

*An object model of the domain that incorporates both behavior and data.*
*一个既包含行为又包含数据的领域对象模型。*

To tell you the truth, this definition can be interpreted to fit almost any piece of code—a fairly good reason why I thought I was using the pattern when in fact I wasn't being true to its original intent.
说实话，这个定义几乎可以解释任何代码——这也是为什么我觉得我在使用这个模式，但实际上并没有遵循其初衷的一个相当好的理由。

Let's dig deeper.  让我们深入挖掘。

Reasons Not to Use the Domain Model
不使用领域模型的原因

In the text following the original description, I had originally blown past this innocuous passage, but it turns out that many important decisions hinge on understanding it.
在原始描述之后的文本中，我原本忽略了这段看似无害的文字，但事实证明，许多重要的决策都取决于理解它。

Since the behavior of the business is subject to a lot of change, it's important to be able to modify, build, and test this layer easily. As a result you'll want the minimum of coupling from the Domain Model to other layers in the system.
由于业务行为经常变化，因此能够轻松地修改、构建和测试这一层非常重要。因此，你希望从领域模型到系统其他层的耦合度最小。

So one reason not to make use of the domain model pattern is if the business your software is automating does not change much. That's not to say it does not change at all—but rather that the underlying rules dictating how business is done aren't very dynamic. While other technological and environmental factors may change, that is not the context of this pattern.
不使用领域模型模式的一个原因可能是，你正在自动化的业务变化不大。这并不是说它完全没有变化——而是说决定业务如何运作的基础规则并不非常动态。虽然其他技术和环境因素可能会变化，但这并不是这个模式的适用范围。

Some examples of this include supporting multiple databases (such as SQL Server and Oracle) or multiple UI technologies (Windows, Web, Mobile, and so on). If the behavior of the business hasn't changed, these do not justify the use of the domain model pattern. That is not to say one could not get a great deal of value from using technologies that support the pattern, but we need to be honest about which rules we break and why.
这方面的例子包括支持多个数据库（如 SQL Server 和 Oracle）或多个 UI 技术（Windows、Web、移动等）。如果业务行为没有变化，这些并不足以证明使用领域模型模式的合理性。这并不是说使用支持该模式的技术不能带来巨大价值，但我们需要诚实地面对我们打破哪些规则以及为什么。

Reasons to Use the Domain Model
使用领域模型的理由

In those cases where the behavior of the business is subject to a lot of change, having a domain model will decrease the total cost of those changes. Having all the behavior of the business that is likely to change encapsulated in a single part of our software decreases the amount of time we need to perform a change because it will all be performed in one place. By isolating that code as much as possible, we decrease the likelihood of changes in other places causing it to break, thus decreasing the time it takes to stabilize the system.
在这些情况下，如果业务行为经常变化，使用领域模型可以降低这些变化的总体成本。将所有可能变化的业务行为封装在我们软件的一个部分中，可以减少我们执行变更所需的时间，因为所有变更都将在一个地方进行。通过尽可能隔离这段代码，我们减少了其他地方的变更导致其崩溃的可能性，从而减少了系统稳定所需的时间。

Scenarios for Not Using the Domain Model
不使用领域模型的场景

This leads us to the No. 1 most common fallacy about employing domain models. I myself was guilty of making this false assumption for a number of years and see now where it led me astray.
这让我们来到了关于使用领域模型的第一个最常见的谬误。我自己就曾在这个错误假设上犯过错好几年，现在才明白它让我走错了路。

Fallacy: Any persistent object model is a domain model
谬误：任何持久化对象模型都是一个领域模型

First of all, a persistent object model does not inherently encapsulate all the behaviors of the business that are likely to change. Second, a persistent object model may include functionality that is not likely to change.
首先，持久化对象模型并不本质上封装所有可能变化的业务行为。其次，持久化对象模型可能包含一些不太可能变化的函数功能。

The nature of this fallacy is similar to stating that any screwdriver is a hammer. While you can (try to) hammer in nails with a screwdriver, you won't be very effective doing it that way. One could hardly say you were being true to the hammer pattern.
这种谬误的性质类似于说任何螺丝刀都是锤子。虽然你可以（尝试）用螺丝刀钉钉子，但那样做效率不会很高。你很难说你是在遵循锤子模式。

Bringing this back to concrete scenarios we all know and love, let's consider the ever-present requirement that a user's e-mail address should be unique.
让我们回到我们熟悉且喜爱的具体场景中，考虑一个普遍存在的需求：用户的电子邮件地址应该是唯一的。

For a while, I thought that the whole point of having a domain model was that requirements like this would be implemented there. However, when we consider the guidance that the domain model is about capturing those business behaviors that are subject to change, we can see that this requirement doesn't fit that mold. It is likely that this requirement will never change.
有一段时间，我以为领域模型的作用就是实现这类需求。然而，当我们考虑领域模型旨在捕捉那些可能发生变化业务行为的指导原则时，我们可以看到这个需求并不符合这种模式。这个需求很可能永远不会改变。

Therefore, choosing to implement such a requirement in the part of the system that is about encapsulating the volatile parts of the business makes little sense, may be difficult to implement, and might not perform that well. Bringing all e-mail addresses into memory would probably get you locked up by the performance police. Even having the domain model call some service, which calls the database, to see if the e-mail address is there is unnecessary. A unique constraint in the database would suffice.
因此，在系统封装业务易变部分的部分中实施这种需求，几乎没有意义，可能难以实现，而且性能可能不佳。将所有电子邮件地址都加载到内存中可能会让你被性能警察盯上。即使让领域模型调用某个服务，该服务再调用数据库来检查电子邮件地址是否存在也是不必要的。数据库中的唯一约束就足够了。

This pragmatic thinking is very much at the core of the domain model pattern and domain-driven design and is what will keep things simple even as we tackle requirements more involved than simple e-mail uniqueness.
这种务实思维正是领域模型模式和领域驱动设计的核心，也是我们在处理比简单电子邮件唯一性更复杂的需求时保持简洁的关键。

Scenarios for Using the Domain Model
使用领域模型模式的应用场景

Business rules that indicate when certain actions are allowed are good candidates for being implemented in a domain model.
表示某些操作何时允许的业务规则是实施领域模型的良好候选。

For example, in an e-commerce system a rule stating that a customer may have no more than $1,000 in unpaid orders would likely belong in the domain model. Notice that this rule involves multiple entities and would need to be evaluated in a variety of use cases.
例如，在一个电子商务系统中，一条规定客户未付款订单总额不得超过 1,000 美元的规则，很可能属于领域模型。请注意，这条规则涉及多个实体，并且需要在各种用例中进行评估。

Of course, in a given domain model we'd expect to see many of these kinds of rules, including cases where some rules override others. In our example above, if the user performing a change to an order is the account manager for the account the customer belongs to, then the previous rule does not apply.
当然，在一个给定的领域模型中，我们期望看到许多这类规则，包括某些规则会覆盖其他规则的情况。在我们的示例中，如果修改订单的用户是客户所属账户的账户经理，那么之前的规则就不适用。

It may appear unnecessary to spend time going through which rules need to apply in which use cases and quickly come up with a list of entities and relationships between them—eschewing the "big design up front" that agile practices rail against. However, the business rules and use cases are the very reasons we're applying the domain model pattern in the first place.
花时间逐一审视哪些规则适用于哪些使用场景，并迅速列出实体及其之间的关系，似乎没有必要——避免敏捷实践中所反对的“前期大设计”。然而，业务规则和使用场景正是我们应用领域模型模式的首要原因。

When solving these kinds of problems in the past, I wouldn't have thought twice and would have quickly designed a Customer class with a collection of Order objects. But our rules so far indicate only a single property on Customer instead—UnpaidOrdersAmount. We could go through several rules and never actually run into something that clearly pointed to a collection of Orders. In which case, the agile maxim "you aren't gonna need it" (YAGNI) should prevent us from creating that collection.
过去解决这类问题时，我通常不会多想，会迅速设计一个包含订单对象的客户类。但目前的规则表明，客户类只有一个属性——未支付订单金额。我们可能需要审视几条规则，却从未遇到明确指向订单集合的情况。在这种情况下，敏捷格言“你现在不需要它”（YAGNI）应阻止我们创建该集合。

When looking at how to persist this graph of objects, we may find it expedient to add supporting objects and collections underneath. We need to clearly differentiate between implementation details and core business behaviors that are the responsibility of the domain model.
当考虑如何持久化这个对象图时，我们可能会发现添加底层支撑对象和集合是明智之举。我们需要明确区分实现细节和领域模型负责的核心业务行为。

More Complex Interactions
更复杂的交互

Consider the requirement that when a customer has made more than $10,000 worth of purchases with our company, they become a "preferred" customer. When a customer becomes a preferred customer, the system should send them an e-mail notifying them of the benefits of our preferred customer program.
考虑这样一个需求：当客户在我们公司累计消费超过 10,000 美元时，他们将成为“优先”客户。当客户成为优先客户时，系统应该向他们发送一封电子邮件，通知他们优先客户计划的权益。

What makes this scenario different from the unique e-mail address requirement described previously is that this interaction does necessarily involve the domain model. One option is to implement this logic in the code that calls the domain model as follows:
使这个场景与之前描述的独特电子邮件地址要求不同的地方在于，这种交互不一定涉及领域模型。一个选项是在调用领域模型的代码中实现这种逻辑，如下所示：

Copy  复制

```
public void SubmitOrder(OrderData data) { bool wasPreferredBefore = customer.IsPreferred; // call the domain model for regular order submit logic if (customer.IsPreferred && !wasPreferredBefore) // send email }
```

One pitfall that the sample code avoids is that of checking the amount that constitutes when a customer becomes preferred. That logic is appropriately entrusted to the domain model.
示例代码避免了一个陷阱，即检查客户何时成为优选客户的金额构成。这种逻辑被适当地委托给了领域模型。

Unfortunately, we can see that the sample code is liable to become bloated as more rules are added to the system that needs to be evaluated when orders are submitted. Even if we were to move this code into the domain model, we'd still be left with the following issues.
不幸的是，我们可以看到，随着需要评估的规则越来越多，示例代码可能会变得臃肿。即使我们将这段代码移入领域模型，我们仍然会面临以下问题。

Cross-Cutting Business Rules
跨领域业务规则

There may be other use cases that result in the customer becoming preferred. We wouldn't want to have to duplicate that logic in multiple places (whether it's in the domain model or not), especially because refactoring to an extracted method would still require capturing the customer's original preferred state.
可能会有其他使用场景导致客户成为优先客户。我们不想在多个地方重复这个逻辑（无论它是否在领域模型中），特别是重构为提取方法仍然需要捕获客户的原始优先状态。

We may even need to go so far as to include some kind of interception/aspect-oriented programming (AOP) method to avoid the duplication.
我们甚至可能需要包括某种拦截/面向切面编程（AOP）方法来避免重复。

It looks like we'd better rethink our approach before we cut ourselves on Occam's razor. Looking at our requirements again may give us some direction.
看来我们最好在用奥卡姆剃刀伤到自己之前重新思考我们的方法。再次审视我们的需求可能会给我们一些方向。

When a customer has become a [something] the system should [do something].
当客户成为[某种状态]时，系统应该[做某事]。

We seem to be missing a good way of representing this requirement pattern, although this does sound like something that an event-based model could handle well. That way, if we're required to do more in the "should do something" part, we could easily implement that as an additional event handler.
我们似乎缺少一个很好的方式来表示这种需求模式，尽管这听起来像是事件模型可以很好地处理的事情。那样的话，如果我们需要在“应该做某事”的部分做更多事情，我们可以轻松地将其实现为附加的事件处理器。

Domain Events and Their Callers
领域事件及其调用者

Domain events are the way we explicitly represent the first part of the requirement described:
领域事件是我们明确表示需求描述第一部分的方式：

When a [something] has become a [something] ...
当一个[something]变成了[something]...

While we can implement these events on the entities themselves, it may be advantageous to have them be accessible at the level of the whole domain. Let's compare how the service layer behaves in either case:
虽然我们可以在实体本身上实现这些事件，但将它们在领域级别上实现可能更有优势。让我们比较一下在两种情况下服务层的行为：

Copy  复制

```
public void SubmitOrder(OrderData data) { var customer = GetCustomer(data.CustomerId); var sendEmail = delegate { /* send email */ }; customer.BecamePreferred += sendEmail; // call the domain model for the rest of the regular order submit logic customer.BecamePreferred -= sendEmail; // to avoid leaking memory }
```

While it's nice not having to check the state before and after the call, we've traded that complexity with that of subscribing and removing subscriptions from the domain event. Also, code that calls the domain model in any use case shouldn't have to know if a customer can become preferred there. When the code is directly interacting with the customer, this isn't such a big deal. But consider that when submitting an order, we may bring the inventory of one of the order products below its replenishment threshold—we wouldn't want to handle that event in the code, too.
不用在调用前后检查状态确实不错，但我们用领域事件订阅和取消订阅的复杂性做了交换。此外，任何使用场景中调用领域模型的代码都不应该知道客户是否可以成为优先客户。当代码直接与客户交互时，这并不是什么大问题。但考虑一下，提交订单时，我们可能会使订单产品之一库存低于补货阈值——我们也不希望在代码中处理该事件。

It would be better if we could have each event be handled by a dedicated class that didn't deal with any specific use case but could be generically activated as needed in all use cases. Here's what such a class would look like:
如果每个事件都能由一个专门的类来处理，而这个类不处理任何特定的使用场景，但在所有使用场景中根据需要可以通用激活，那就更好了。这样一个类看起来是这样的：

Copy  复制

```
public class CustomerBecamePreferredHandler : Handles<CustomerBecamePreferred> { public void Handle(CustomerBecamePreferred args) { // send email to args.Customer } }
```

We'll talk about what kind of infrastructure will make this class magically get called when needed, but let's see what's left of the original submit order code:
我们会讨论什么样的基础设施能让这个类在需要时神奇地被调用，但让我们看看原始提交订单代码还剩下什么：

Copy  复制

```
public void SubmitOrder(OrderData data) { // call the domain model for regular order submit logic }
```

That's as clean and straightforward as one could hope—our code doesn't need to know anything about events.
这正如我们所期望的那样干净利落——我们的代码不需要知道任何关于事件的事情。

Explicit Domain Events  显式领域事件

In the CustomerBecamePreferredHandler class we see the reference to a type called CustomerBecamePreferred—an explicit representation in code of the occurrence mentioned in the requirement. This class can be as simple as this:
在 CustomerBecamePreferredHandler 类中，我们看到了一个名为 CustomerBecamePreferred 的类型引用——这是代码中对需求中提到的发生事件的显式表示。这个类可以简单到如下所示：

Copy  复制

```
public class CustomerBecamePreferred : IDomainEvent { public Customer Customer { get; set; } }
```

The next step is to have the ability for any class within our domain model to raise such an event, which is easily accomplished with the following static class that makes use of a container like Unity, Castle, or Spring.NET:
下一步是让我们的领域模型中的任何类都能够引发此类事件，这可以通过以下静态类轻松实现，该类使用 Unity、Castle 或 Spring.NET 等容器：

Copy  复制

```
public static class DomainEvents { public IContainer Container { get; set; } public static void Raise<T>(T args) where T : IDomainEvent { foreach(var handler in Container.ResolveAll<Handles<T>>()) handler.Handle(args); } }
```

Now, any class in our domain model can raise a domain event, with entity classes usually raising the events like so:
现在，我们领域模型中的任何类都可以引发领域事件，实体类通常以如下方式引发事件：

Copy  复制

```
public class Customer { public void DoSomething() { // regular logic (that also makes IsPreferred = true) DomainEvents.Raise(new CustomerBecamePreferred() { Customer = this }); } }
```

Testability  可测试性

While the DomainEvents class shown is functional, it can make unit testing a domain model somewhat cumbersome as we'd need to make use of a container to check that domain events were raised. Some additions to the DomainEvents class can sidestep the issue, as shown in **Figure 1**.
虽然展示的 DomainEvents 类是功能性的，但它会使领域模型的单元测试变得有些繁琐，因为我们需要使用容器来检查是否触发了领域事件。DomainEvents 类的一些补充可以规避这个问题，如图 1 所示。

## Figure 1 Additions to the DomainEvents Class
图 1 DomainEvents 类的添加

Copy  复制

```
public static class DomainEvents { [ThreadStatic] //so that each thread has its own callbacks private static List<Delegate> actions; public IContainer Container { get; set; } //as before //Registers a callback for the given domain event public static void Register<T>(Action<T> callback) where T : IDomainEvent { if (actions == null) actions = new List<Delegate>(); actions.Add(callback); } //Clears callbacks passed to Register on the current thread public static void ClearCallbacks () { actions = null; } //Raises the given domain event public static void Raise<T>(T args) where T : IDomainEvent { foreach(var handler in Container.ResolveAll<Handles<T>>()) handler.Handle(args); if (actions != null) foreach (var action in actions) if (action is Action<T>) ((Action<T>)action)(args); } }
```

Now a unit test could be entirely self-contained without needing a container, as **Figure 2** shows.
现在单元测试可以完全自包含，无需容器，如图 2 所示。

## Figure 2 Unit Test Without Container
图 2 无容器的单元测试

Copy  复制

```
public class UnitTest { public void DoSomethingShouldMakeCustomerPreferred() { var c = new Customer(); Customer preferred = null; DomainEvents.Register<CustomerBecamePreferred>( p => preferred = p.Customer ); c.DoSomething(); Assert(preferred == c && c.IsPreferred); } }
```

Commands and Queries  命令和查询

The use cases we've been examining so far have all dealt with changing data and the rules around them. Yet in many systems, users will also need to be able to view this data, as well as perform all sorts of searches, sorts, and filters.
我们到目前为止所检查的使用案例都涉及更改数据及其相关规则。然而，在许多系统中，用户还需要能够查看这些数据，以及执行各种搜索、排序和过滤。

I had originally thought that the same entity classes that were in the domain model should be used for showing data to the user. Over the years, I've been getting used to understanding that my original thinking often turns out to be wrong. The domain model is all about encapsulating data with business behaviors.
我原本以为，领域模型中的实体类应该用于向用户展示数据。多年来，我逐渐习惯于认识到我最初的设想往往是不正确的。领域模型的核心在于将数据与业务行为进行封装。

Showing user information involves no business behavior and is all about opening up that data. Even when we have certain security-related requirements around which users can see what information, that often can be represented as a mandatory filtering of the data.
向用户展示信息并不涉及业务行为，完全是关于开放数据。即使我们有关于哪些用户可以查看哪些信息的特定安全要求，这也通常可以表示为对数据的强制过滤。

While I was "successful" in the past in creating a single persistent object model that handled both commands and queries, it was often very difficult to scale it, as each part of the system tugged the model in a different direction.
虽然我在过去“成功”地创建了一个单一的持久化对象模型，该模型处理命令和查询，但它往往很难扩展，因为系统的每个部分都从不同的方向拉扯模型。

It turns out that developers often take on more strenuous requirements than the business actually needs. The decision to use the domain model entities for showing information to the user is just such an example.
结果发现，开发者往往承担了比业务实际需求更繁重的任务。使用领域模型实体向用户展示信息的决策就是一个这样的例子。

You see, in a multi-user system, changes made by one user don't necessarily have to be immediately visible to all other users. We all implicitly understand this when we introduce caching to improve performance—but the deeper questions remain: If you don't need the most up-to-date data, why go through the domain model that necessarily works on that data? If you don't need the behavior found on those domain model classes, why plough through them to get at their data?
你明白，在一个多用户系统中，一个用户所做的更改不一定需要立即对所有其他用户可见。当我们引入缓存来提高性能时，我们都隐含地理解这一点——但更深层次的问题仍然存在：如果你不需要最新的数据，为什么要通过必然依赖于这些数据的领域模型呢？如果你不需要那些领域模型类上的行为，为什么要费力去获取它们的数据呢？

For those old enough to remember, the best practices around using COM+ guided us to create separate components for read-only and for read-write logic. Here we are, a decade later, with new technologies like the Entity Framework, yet those same principles continue to hold.
对于足够年长以记住的人来说，使用 COM+的最佳实践指导我们为只读逻辑和读写逻辑创建单独的组件。十年过去了，我们有了新的技术，比如实体框架，但同样的原则仍然适用。

Getting data from a database and showing it to a user is a fairly trivial problem to solve these days. This can be as simple as using an ADO.NET data reader or data set.
从数据库获取数据并展示给用户在当今是一个相当简单的问题。这可以简单地使用 ADO.NET 数据读取器或数据集来实现。

**Figure 3** shows what our "new" architecture might look like.
图 3 展示了我们“新”架构可能的样子。

![](https://learn.microsoft.com/en-us/archive/msdn-magazine/2009/brownfield/images/ee236415.fig03(en-us,msdn.10).gif)

Figure 3 **Model for Getting Data from a Database**
图 3 从数据库获取数据的模型

One thing that is different in this model from common approaches based on two-way data binding, is that the structure that is used to show the data isn't used for changes. This makes things like change-tracking not entirely necessary.
在这个模型中，与基于双向数据绑定的常见方法不同之处在于，用于显示数据的结构并不用于数据变更。这使得诸如变更跟踪等功能并非完全必要。

In this architecture, data flows up the right side from the database to the user in the form of queries and down the left side from the user back to the database in the form of commands. Choosing to go to a fully separate database used for those queries is a compelling option in terms of performance and scalability, as reads don't interfere with writes in the database (including which pages of data are kept in memory in the database), yet an explicit synchronization mechanism between the two is required. Options for this include ADO.NET Sync Services, SQL Server Integration Services (SSIS), and publish/subscribe messaging. Choosing one of these options is beyond the scope of this article.
在这种架构中，数据以查询的形式从数据库流向右侧的用户，并以命令的形式从用户流向左侧的数据库。选择使用一个完全独立的数据库来处理这些查询，在性能和可扩展性方面是一个有吸引力的选项，因为读取操作不会干扰数据库的写入操作（包括数据库内存中保留的数据页），但需要在两者之间实现显式的同步机制。这些选项包括 ADO.NET Sync Services、SQL Server Integration Services (SSIS) 以及发布/订阅消息。选择其中一种选项超出了本文的范围。

Keeping the Business in the Domain
将业务保持在领域内

One of the challenges facing developers when designing a domain model is how to ensure that business logic doesn't bleed out of the domain model. There is no silver-bullet solution to this, but one style of working does manage to find a delicate balance between concurrency, correctness, and domain encapsulation that can even be tested for with static analysis tools like FxCop.
开发者在设计领域模型时面临的一个挑战是如何确保业务逻辑不会从领域模型中泄露出来。对于这个问题，没有一劳永逸的解决方案，但有一种工作方式能够在并发性、正确性和领域封装之间找到微妙的平衡，甚至可以使用 FxCop 等静态分析工具进行测试。

Here is an example of the kind of code we wouldn't want to see interacting with a domain model:
这里是一个我们不希望与领域模型交互的代码示例：

Copy  复制

```
public void SubmitOrder(OrderData data) { var customer = GetCustomer(data.CustomerId); var shoppingCart = GetShoppingCart(data.CartId); if (customer.UnpaidOrdersAmount + shoppingCart.Total > Max) // fail (no discussion of exceptions vs returns codes here) else customer.Purchase(shoppingCart); }
```

Although this code is quite object-oriented, we can see that a certain amount of business logic is being performed here rather than in the domain model. A preferable approach would be this:
尽管这段代码非常面向对象，但我们可以看到，相当一部分业务逻辑是在这里执行的，而不是在领域模型中。更可取的方法是：

Copy  复制

```
public void SubmitOrder(OrderData data) { var customer = GetCustomer(data.CustomerId); var shoppingCart = GetShoppingCart(data.CartId); customer.Purchase(shoppingCart); }
```

In the case of the new order exceeding the limit of unpaid orders, that would be represented by a domain event, handled by a separate class as demonstrated previously. The purchase method would not cause any data changes in that case, resulting in a technically successful transaction without any business effect.
在新订单超过未支付订单限额的情况下，这将被表示为一个领域事件，并由前面演示的单独类进行处理。在这种情况下，购买方法不会导致任何数据变化，从而实现技术上成功的交易，而没有任何业务影响。

When inspecting the difference between the two code samples, we can see that calling only a single method on the domain model necessarily means that all business logic has to be encapsulated there. The more focused API of the domain model often further improves testability.
在检查两个代码示例的差异时，我们可以看到，仅调用领域模型中的单个方法意味着所有业务逻辑都必须封装在那里。领域模型的更专注的 API 通常进一步提高了可测试性。

While this is a good step in the right direction, it does open up some questions about concurrency.
虽然这是一个正确的方向上的好步骤，但它确实引发了一些关于并发性的问题。

Concurrency  并发性

You see, in between the time where we get the customer and the time we ask it to perform the purchase, another transaction can come in and change the customer in such a way that its unpaid order amount is updated. That may cause our transaction to perform the purchase (based on previously retrieved data), although it doesn't comply with the updated state.
你看，在我们获取客户和请求其执行购买操作之间的时间，另一个事务可能会进来并改变客户，使其未支付订单金额得到更新。这可能会导致我们的交易根据先前检索的数据执行购买（尽管它不符合更新后的状态）。

The simplest way to solve this issue is for us to cause the customer record to be locked when we originally read it—performed by indicating a transaction isolation level of at least repeatable-read (or serializable—which is the default) as follows:
解决这个问题最简单的方法是在我们最初读取客户记录时就锁定它——通过指定至少可重复读（或串行化，这是默认值）的事务隔离级别来实现：

Copy  复制

```
public void SubmitOrder(OrderData data) { using (var scope = new TransactionScope( TransactionScopeOption.Required, new TransactionOptions() { IsolationLevel = IsolationLevel.RepeatableRead } )) { // regular code } }
```

Although this does take a slightly more expensive lock than the read-committed isolation level some high-performance environments have settled on, performance can be maintained at similar levels when the entities involved in a given use case are eagerly loaded and are connected by indexed columns. This is often largely offset by the much simpler applicative coding model, because no code is required for identifying or resolving concurrency issues. When employing a separate database for the query portions of the system and all reads are offloaded from the OLTP database serving the domain model, performance and scalability can be almost identical to read-committed-based solutions.
尽管这确实比一些高性能环境采用的可提交隔离级别稍微贵一些的锁，但在特定用例中涉及的实体被急切加载并通过索引列连接时，性能可以保持在相似的水平。这通常在很大程度上被更简单的应用编码模型所抵消，因为不需要编写代码来识别或解决并发问题。当为系统的查询部分使用单独的数据库，并且所有读取操作都从为领域模型服务的 OLTP 数据库中卸载时，性能和可扩展性几乎可以与基于可提交的解决方案相同。

Finding a Comprehensive Solution
寻找一个全面的解决方案

The domain model pattern is indeed a powerful tool in the hands of a skilled craftsman. Like many other developers, the first time I picked up this tool, I over-used it and may even have abused it with less than stellar results. When designing a domain model, spend more time looking at the specifics found in various use cases rather than jumping directly into modeling entity relationships—especially be careful of setting up these relationships for the purposes of showing the user data. That is better served with simple and straightforward database querying, with possibly a thin layer of facade on top of it for some database-provider independence.
领域模型模式确实是一位熟练工匠手中的强大工具。和其他许多开发者一样，我第一次接触这个工具时，过度使用了它，甚至可能因为使用不当而得到了不尽如人意的结果。在设计领域模型时，应该花更多时间研究各种用例中的具体细节，而不是直接进入实体关系的建模——尤其是要小心为展示用户数据而设置这些关系。这更适合使用简单直接的数据库查询，可能再加上一层薄薄的面向数据库提供者的封装层。

When looking at how code outside the domain model interacts with it, look for the agile "simplest thing that could possibly work"—a single method call on a single object from the domain, even in the case when you're working on multiple objects. Domain events can help round out your solution for handling more complex interactions and technological integrations, without introducing any complications.
当观察领域模型外部的代码如何与其交互时，寻找敏捷的“可能起作用的最简单事物”——即使是在处理多个对象的情况下，也要寻找对领域模型中的单个对象进行单个方法调用。领域事件可以帮助完善你的解决方案，以处理更复杂的交互和技术集成，而不会引入任何复杂性。

When starting down this path, it took me some time to adjust my thinking, but the benefits of each pattern were quickly felt. When I began employing all of these patterns together, I found they provided a comprehensive solution to even the most demanding business domains, while keeping all the code in each part of the system small, focused, and testable—everything a developer could want.
刚开始走这条路时，我花了一些时间调整我的思维方式，但每个模式的好处很快就体会到了。当我开始一起使用所有这些模式时，我发现它们为最苛刻的业务领域提供了一个全面的解决方案，同时使系统每个部分的代码都保持小巧、专注和可测试——这是开发者所期望的一切。

Works Cited  参考文献

Fowler, M. *Patterns of Enterprise Application Architecture*, (Addison Wesley, 2003).
Fowler, M. 《企业应用架构模式》 (Addison Wesley, 2003).

**Udi Dahan** Recognized as an MVP, an IASA Master Architect, and Dr. Dobb's SOA Guru, Udi Dahan is The Software Simplist, an independent consultant, speaker, author, and trainer providing high-end services in service-oriented, scalable, and secure enterprise architecture and design. Contact Udi via his blog at [UdiDahan.com](https://udidahan.com/).
Udi Dahan 被评为 MVP、IASA 大师架构师，以及 Dr. Dobb's SOA 大师，他是 The Software Simplist，一位独立顾问、演讲者、作者和培训师，提供高端服务于面向服务的、可扩展的和安全的 企业架构和设计。通过 UdiDahan.com 博客联系 Udi。
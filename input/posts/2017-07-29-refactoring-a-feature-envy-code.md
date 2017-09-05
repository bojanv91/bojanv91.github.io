Tags: [code smells, refactoring]
RedirectFrom: 2017/07/refactoring-a-feature-envy-code/
Lead: Towards less coupled code, adhering to the Tell Don't Ask principle
Title: Refactoring a feature envy code
---

In a design review meeting a colleague asked: *Why sometimes we directly manipulate dependent objects fields, and sometimes we put the manipulation logic behind methods in those objects? What are pros/cons in both approaches?* This was a great question, because it opened a productive discussion. Checking the [code smells taxonomy](https://blog.codinghorror.com/code-smells/), and analyzing the code under review deeper, we identified it belongs to the **feature envy** code smells category. And this is how we refactored it.<!--excerpt--> 

**What is feature envy code?**
*Feature envy* is a code smell describing when an object accesses fields of another object to execute some operation, instead of just telling the object what to do.

Let's analyze the following code segment, and try to refactor it. 
For better context, it addresses the requirement: *An active user can pay a pending order. For reporting purposes the order tracks when and who paid it.*

## Original code segment (simplified)

Here you can see the original code segment greatly simplified, so it's easier to follow.

```C#
public class PayOrderRequest
{
    public Guid OrderId { get; set; }
}

public class PayOrderHandler : BaseCommandHandler<PayOrderRequest>
{
    public void Handle(PayOrderRequest request)
    {
        var currentUser = Session.Load<User>(LoggedInUserId);
        var order = Session.Load<Order>(request.OrderId);

        // checks if the order can be paid
        if (order.Status == OrderStatus.Pending && currentUser.Status == UserStatus.Active) 
        {
            // pay the order, and record related information
            order.Status = OrderStatus.Completed;
            order.PaidByUserId = currentUser.Id;
            order.PaidOnUtc = DateTime.UtcNow;
        }
        else
        {
            throw new CoreException($"{currentUser.Name} cannot pay this order.");
        }
        
        Session.Store(order);
    }
}
```

The core problem with this code is that it breaks encapsulation. The command handler depends too much on the ``order`` internals, and forms tight coupling. The ``order`` leaked it's domain logic to the command handler, thus it became anemic data-object.

Besides breaking encapsulation, it also makes the paying order functionality  hard to unit test. The command handler depends on the database via the session object, and to the logged in user provider. Writing a unit test for this, means we need to write mocks for both services. If we refactor it, fixing the encapsulation, we'll see that writing unit tests will be an easier task to do. Besides, why create mocks until deemed necessary?

Ultimately, the command handler should only coordinate the workflow, and the order object should only deal with the domain logic. They should not mix responsibilities between themselves. But this is not the case we have here.

Our question is, can the command handler **tell** the ``order`` what to do, encapsulating the logic, instead of asking it for too many details? Let's find out.



## Step 1 - hold precondition result in an inline variable

Let's introduce ``canOrderByPaid`` boolean variable which will hold the result of the precondition for paying an order.

```c#
// Original
if (order.Status == OrderStatus.Pending && currentUser.Status == UserStatus.Active) 
{ ... }
  

// Refactored (step 1)
bool canOrderByPaid = order.Status == OrderStatus.Pending && currentUser.Status == UserStatus.Active;
if (canOrderByPaid)
{ ... }
```


It's better to create a inline variable to describe more complex condition checks, and use it in the if-condition statement, than having bloated if-condition statement with a comment above, describing what it does.



## Step 2 - encapsulate the precondition in the ``Order`` 

Now it makes sense to encapsulate the precondition for paying an order, in the order object itself.

```c#
// Original
if (order.Status == OrderStatus.Pending && currentUser.Status == UserStatus.Active) 
{ ... }
  

// Refactored (step 1)
bool canOrderByPaid = order.Status == OrderStatus.Pending && currentUser.Status == UserStatus.Active;
if (canOrderByPaid)
{ ... }


// Refactored (step 2)
if (order.CanBePaidBy(currentUser)) 
{ ... }
```

The order class:

```c#
public class Order
{
    /* Code removed for clarity */

    public bool CanBePaidBy(User user) 
        => Status == OrderStatus.Pending && user.Status == UserStatus.Active;

    /* Code removed for clarity */
}
```

With this change, we eliminated coupling from the command handler to the fields of order and user classes (e.g. ``order.Status``, ``user.Status``). Imagine in more complex cases, how much direct coupling will be reduced only by following good encapsulation.



## Step 3 - encapsulate the actual operation

Following the same way how we encapsulated the paying precondition, we'll encapsulate the paying operation.

```c#
// Refactored (step 2)
if (order.CanBePaidBy(currentUser))
{
    order.Status = OrderStatus.Completed;
    order.PaidByUserId = currentUser.Id;
    order.PaidOnUtc = DateTime.UtcNow;
}
else
    throw new CoreException($"{currentUser.Name} cannot pay this order.");


// Refactored (step 3)
if (order.CanBePaidBy(currentUser))
    order.Pay(currentUser);
else
    throw new CoreException($"User {currentUser.Name} cannot pay this order.");
```

The order class:

```c#
public class Order
{
    /* Code removed for clarity */

    public void Pay(User payer) 
    {
        Status = OrderStatus.Completed;
        PaidByUserId = payer.Id;
        PaidOnUtc = DateTime.UtcNow;
    }

    /* Code removed for clarity */
}
```



## Step 4 - encapsulate further

Seeing the refactoring changes in previous steps, this final refactoring change comes natural.

```c#
// Refactored (step 3)
if (order.CanBePaidBy(currentUser))
    order.Pay(currentUser);
else
    throw new CoreException($"User {currentUser.Name} cannot pay this order.");


// Refactored (step 4)
order.Pay(currentUser);
```

The order class:

```c#
public class Order
{
    /* Code removed for clarity */

    public void Pay(User payer) 
    {
        if (!CanBePaidBy(payer))
            throw new CoreException($"User {payer.Name} cannot pay this order.");

        Status = OrderStatus.Completed;
        PaidByUserId = payer.Id;
        PaidOnUtc = DateTime.UtcNow;
    }

    /* Code removed for clarity */
}
```



## The final refactored code

This is how the final refactored code looks like. Compare it to the original one in the beginning of this session. What key difference can you identify?

```c#
public class PayOrderRequest
{
    public Guid OrderId { get; set; }
}

public class PayOrderHandler : BaseCommandHandler<PayOrderRequest>
{
    public void Handle(PayOrderRequest request)
    {
        var currentUser = Session.Load<User>(LoggedInUserId);
        var order = Session.Load<Order>(request.OrderId);
        
        order.Pay(currentUser);
        
        Session.Store(order);
    }
}
```

And the ``Order`` class, encapsulating the domain logic:

```c#
public class Order
{
    /* Code removed for clarity */
  
    public OrderStatus Status { get; private set; }
    public Guid PaidByUserId { get; private set; }
    public DateTime PaidOnUtc { get; private set; }

    public void Pay(User payer) 
    {
        if (!CanBePaidBy(payer))
            throw new CoreException($"User {payer.Name} cannot pay this order.");

        order.Status = OrderStatus.Completed;
        order.PaidByUserId = payer.Id;
        order.PaidOnUtc = DateTime.UtcNow;
    }

    public bool CanBePaidBy(User user) 
        => Status == OrderStatus.Pending && user.Status == UserStatus.Active;

    /* Code removed for clarity */
}
```

You can see we moved the domain logic out of the command handler, and we put it to the ``Order`` entity, where it belongs.



## Conclusion

The benefits we achieved from this refactoring session are:

* The command handler **stops asking** for details (and internals). Now it **tells** what other objects should do. Does not care about the details anymore. With this, we fulfilled the **Tell Don't Ask Principle**.
* The direct **coupling** from the command handler to the fields in user and order classes is **eliminated**. Less coupling, more sanity.
* The ``Order`` class is not a bag of public set properties anymore. It's not anemic, it's not a data-class. Now it has cohesive responsibility, **encapsulating the actions it can perform**.
* **Domain logic** for paying **is not leaked** in other classes anymore. Now it's located in the responsible class itself. So, only by looking at the ``Order`` class we can understand what it can do. No need to open other related classes. 
* Paying functionality can be easily **unit tested**, without having to deal with special mocking techniques. Less mocks, better tests.



**Rule of thumb:** Where ever you see a method uses fields of another class extensively to perform some action, consider moving the action's logic into *that* class itself.


Tags: [code smells, refactoring]
RedirectFrom: 2017/07/refactoring-a-feature-envy-code/
Lead: A step-by-step session to less coupled code, adhering to the Tell Don't Ask principle
Title: Refactoring a feature envy code
---

Recently, in a design review meeting the following question was raised: *When refactoring, why sometimes do we put an if-condition statement in a temporary variable, and sometimes in a method in a dependent class?*
This is a wonderful question! It opened a discussion spanning across multiple topics, including coupling, domain leaking, encapsulation, and the tell don't ask principle. 

Ultimately, checking the [code smells taxonomy](https://blog.codinghorror.com/code-smells/), we identified the code segment in question is a smell called *feature envy*.<!--excerpt-->

**Feature envy** is a code smell describing when an object accesses fields of another object in order to perform some action, rather than telling the object to do the action itself.

Let's examine the following [simplified] code segment, and try to refactor it. 
For better context, it addresses the requirement: An active user can pay a pending order. For reporting purposes the order tracks when and who paid it.

## v1 - [simplified] original code segment

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
    
            // checks if the order can be paid by the currently logged in user
            if (order.Status == OrderStatus.Pending && currentUser.Status == UserStatus.Active) 
            {
                // pay the order, and record related information
                order.Status = OrderStatus.Completed;
                order.PaidByUserId = currentUser.Id;
                order.PaidOnUtc = DateTime.UtcNow;
    
                Session.Store(order);
            }
            else
            {
                throw new CoreException($"{currentUser.Name} cannot pay this order.");
            }
        }
    }

The problem with this code, and the reason it is considered a code smell, is because it breaks encapsulation. 

The domain logic for performing the paying action on the order object, is leaked out of the order object, into the command handler method. The command handler should only coordinate the workflow, and the order object should be responsible for dealing with the business logic.

Besides breaking encapsulation, it also makes the paying order functionality  harder to unit test. This method depends on the database via the session object, and to the provider of the currently logged in user object. Writing a unit test for this, means that we need to write mocks for both services. If we refactor it, fixing the encapsulation, we'll see that writing unit tests will be a much easier task to do.

Let's observe how can we improve this piece of code, one refactoring step at a time.

## v2 - introduce temporary inline variable for the precondition

Let's introduce ``canOrderByPaid`` boolean variable which will hold the result of the precondition for paying an order.


    /* Code removed for clarity */

    bool canOrderByPaid = order.Status == OrderStatus.Pending && currentUser.Status == UserStatus.Active;
    if (canOrderByPaid)
    {
        order.Status = OrderStatus.Completed;
        order.PaidByUserId = currentUser.Id;
        order.PaidOnUtc = DateTime.UtcNow;
    
        Session.Store(order);
    }
    else
    {
        throw new CoreException($"{currentUser.Name} cannot pay this order.");
    }
    
    /* Code removed for clarity */


It's better to create a temporary inline variable to describe more complex condition checks, and use it in the if-condition statement, rather than having bloated if-condition statement with a comment above, describing what it does.

## v3 - encapsulate the precondition in the order object

Now it makes sense to encapsulate the the precondition for paying an order, in the order object itself.

    if (order.CanBePaidBy(currentUser)) 
    {
        order.Status = OrderStatus.Completed;
        order.PaidByUserId = currentUser.Id;
        order.PaidOnUtc = DateTime.UtcNow;
        Session.Store(order);
    }
    else
    {
        throw new CoreException($"{currentUser.Name} cannot pay this order.");
    }

The order class:

    public class Order
    {
        /* Code removed for clarity */
    
        public bool CanBePaidBy(User user) 
            => Status == OrderStatus.Pending && user.Status == UserStatus.Active;
    
        /* Code removed for clarity */
    }

With this change, we eliminated coupling from the command handler to the properties of order and user classes (e.g. ``order.Status``, ``user.Status``). Imagine in more complex cases, how much direct coupling will be reduced only by following good encapsulation.

## v4 - encapsulate the actual action too

Following the same way how we encapsulated the paying precondition, we'll encapsulate the actual paying action too.

    if (order.CanBePaidBy(currentUser))
    {
        order.Pay(currentUser);
    
        Session.Store(order);
    }
    else
        throw new CoreException($"User {currentUser.Name} cannot pay this order.");

The order class:

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

## v5 - natural final changes

Observing the refactoring changes in previous steps, this final refactoring change comes natural.

    order.Pay(currentUser);

    Session.Store(order);

The order class:

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

## The resulting refactored code

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

The order class:

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

## Summary

We made it to the end. Let's summarize the benefits we achieved out of this refactoring session:

* The command handler **stops asking** for details, now it **tells** the order what to do, providing necessary dependencies, and does not care about the details it does. **Tell Don't Ask Principle** fulfilled.
* The **high coupling** from the command handler to the fields in user and order classes is **eliminated**. Less coupling, more sanity.
* The ``Order`` class is not a bag of public set properties anymore, called a data class. Now it has cohesive behavior, **encapsulating the actions it can perform**.
* **Domain logic** for paying **is not leaked** in other classes anymore. Now it's located in the responsible class itself. So, only by seeing the ``Order`` class we can understand what it can do. No need to open other related classes. Other classes now may be only "users" of this class, calling methods to perform some action/behavior.
* Paying functionality now can be easily **unit tested**, without having to deal with special mocking techniques. Less mocks, better tests.

**Rule of thumb:** Where ever you see a method uses fields of another class extensively, in order to perform some action, consider moving the action in *that* class itself.

---

If there is interest out there, I might write more about practical refactoring topics and software design from my everyday life.


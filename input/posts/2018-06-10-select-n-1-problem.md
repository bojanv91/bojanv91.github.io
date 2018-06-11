Tags: [Software Design]
Title: Select N+1 Problem
---

Select N+1 is a data-access performance problem. Any code that iterates thru a collection of elements, and executes additional query for each element, has this problem. Thought, this behavior is avoidable.<!--excerpt-->

Let's look at this sample code:

![1528580795195](/images/data-model.png)

```c#
var books = LoadBooks();        // 1st query
foreach (var book in books)
{
    var bookReviews = LoadBookReviews(book.Id);    // 2nd query, called N times
    // ...
}
```

`LoadBooks()` makes a database query to load books. Then, for each book `LoadBookReviews(book.Id)` makes a query to load reviews for the given book. Loading books is one query, plus N additional queries for loading book reviews, results in N+1 database queries, where N is the number of books. 

If there are 100 loaded books, the above code will call 101 database queries, thus causing performance problems. Imagine if multiple users open the same page at the same time... 

**How does this look in SQL?**

```sql
-- 1st query
SELECT * FROM Book ...

-- 2nd query, N times
SELECT * FROM BookReview WHERE BookId = 1    -- BookId = @BookId
SELECT * FROM BookReview WHERE BookId = 2
SELECT * FROM BookReview WHERE BookId = 3
...
```

## Solution: Load necessary data before iterating through it

One solution to N+1 problem is to load all necessary data before iterating through it. This way we won't need to execute additional database queries to load child data in for-loops, since we will have that data loaded before.

```c#
var books = LoadBooks();        // 1st query
var reviews = LoadBookReviewsFor(books.Select(x => x.Id)); // 2nd query, called once
foreach (var book in books)
{
    var bookReviews = reviews.FirstOrDefault(x => x.BookId == book.Id);
    // ...
}
```

`LoadBooks()` does the same thing as before, where `LoadBookReviewsFor(..book ids..)` queries the database to load all book reviews for a given list of books, instead for a single book.

In this case, no matter how many books or reviews are there, the code will always call only 2 database queries; thus fixing the N+1 problem.

**How does this look in SQL?**

```sql
-- 1st query
SELECT * FROM Book ...

-- 2nd query
SELECT * FROM BookReview WHERE BookId IN (1, 2, 3, ...)   -- BookId IN @BookIds
```

## Example in Entity Framework with navigation properties and lazy loading

Bad:

```c#
var books = dbContext.Books.ToList();    // 1st query
foreach (var book in books)
{
    var bookReviews = book.Reviews.ToList();    // 2nd query, called N times via lazy loading
    // ...
}
```

Good: refactor to eager loading, using the **Include** feature.

```c#
var books = dbContext.Books.Include(x => x.Reviews).ToList();    // 1st query, batched
foreach (var book in books)
{
    var bookReviews = book.Reviews.ToList();    // in-memory, no additional queries
    // ...
}
```

By using `.Include(x => x.Reviews)` in Entity Framework (or `.Fetch(x => x.Reviews)` in NHibernate) we eagerly load related book reviews in a single database query. This approach solves the Select N+1 problem, but opens doors to loading too many objects in memory, especially when we need to load many related collections; in such cases, plain old SQL approach is preferred.

## Example in Entity Framework without navigation properties

Bad:

```c#
var books = dbContext.Books.ToList();    // 1st query
foreach (var book in books)
{
    // 2nd query, called N times explicitelly 
    var bookReviews = dbContext.BookReviews.Where(x => x.BookId == book.Id).ToList();
    // ...
}
```

Good: **load necessary data before iterating through it**

```c#
var books = dbContext.Books.ToList();            // 1st query
var bookIds = books.Select(x => x.Id).ToArray();
var allReviews = dbContext.BookReviews           // 2nd query
    .Where(x => bookIds.Contains(x.BookId))
    .ToList();

foreach (var book in books)
{
    var bookReviews = allReviews.Where(x => x.BookId == book.Id); // no additional query
    // ...
}
```

## Example in Marten document database

Bad:

```c#
var books = session.Query<Book>().ToList();    // 1st query
foreach (var book in books)
{
    // 2nd query, called N times explicitelly 
    var bookReviews = session.Query<BookReview>()
        .Where(x => x.BookId == book.Id)
        .ToList(); 
    // ...
}
```

Good: same approach, **load necessary data before iterating through it**

```c#
var books = session.Query<Book>().ToList();		// 1st query
var bookIds = books.Select(x => x.Id).ToArray();
var allReviews = session.Query<BookReview>()    // 2nd query
    .Where(x => x.BookId.IsOneOf(bookIds))
    .ToList();								

foreach (var book in books)
{
    var bookReviews = allReviews.Where(x => x.BookId == book.Id); // no additional query
    // ...
}
```

## Summary 

Select N+1 is a performance problem that can really slow down your application. The more "N" results the application has, the less performant it will be when this problem is not detected and fixed.

Here we've seen how to detect such problem and how to fix it. The common approach follows a straightforward practice: **load necessary data before iterating through it.**

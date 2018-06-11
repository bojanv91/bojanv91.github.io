Tags: [Software Design]
RedirectFrom: 2015/09/towards-good-enough-code-re-factoring-business-rule-check-specification-pattern/
Title: Refactoring business rules with the Specification Pattern
---

The other day, one of my colleges asked me for code review on a specific part of code and I said let's dig a little deeper into the options that we have. In this article, I demonstrate the re-factoring steps in detail that we've taken and eventually how we employed the `Specification Pattern` <!--excerpt-->. Have in mind that, I choose a very basic example in order to keep things simple and avoid confusion that can be arouse from domain complexity.<!--excerpt-->

Here is the original code:  
  	
	//..

	var newCompany = new Company(message.Name, message.CountryId);
	
	// Query all companies from database 
	var companies = _companyRepository.Query().ToList();
	// Check if the newly created company is unique
	if (companies.Any(x => x.Name == newCompany.Name && x.CountryId == newCompany.CountryId))
		throw new Exception("A company with the same name and country already exists");

	session.Save(newCompany);
	//..

Here, we can see a few problems. First, all companies are queried from the database, and that can create performance issues. Another problem is too much operations happening in the `If` check line; thus, the lengthy line is making the code harder to read. And, the final problem is very plain practice of `Exception` throwing. Although, I like expressing explicit guard checks, that code can be better. Let's tackle these problems, one by one, in a few steps along this article and provide some improvement suggestions.

Also, I provide here the `tl;dr;` version of the code:

	//..

	var newCompany = new Company(message.Name, message.CountryId);
	
	var spec = new UniqueCompanySpecification(_companyRepository);
	if (spec.IsSatisfiedBy(newCompany) == false)
		throw new CompanyAlreadyExistsException();

	session.Save(newCompany);
	//..

# How we get there?

## Step 1 - Solve The Query Performance Issues

	var numberOfSameCompanies = _companyRepository.Query()
		.Where(x => x.Name == newCompany.Name && x.CountryId == newCompany.CountryId)
		.Count();
	if (numberOfSameCompanies > 0)
		throw new Exception("A company with the same name and country already exists");

The query above retrieves the number of companies satisfying the given `where` condition. Performance issues have been solved.

## Step 2 - Make The `if` Condition Check Explicit 
	
	var numberOfSameCompanies = _companyRepository.Query()
		.Where(x => x.Name == newCompany.Name && x.CountryId == newCompany.CountryId)
		.Count();
	var doesCompanyAlreadyExists = numberOfSameCompanies > 0;
	if (doesCompanyAlreadyExists)
		throw new Exception("A company with the same name and country already exists");

By setting some explicit conditions, we gain clear understanding of what the code does.

## Step 3 - Make The Business Rule Violation Explicit 

Original:

	throw new Exception("A company with the same name and country already exists");

Re-factored to:

	throw new CompanyAlreadyExistsException();

And the implementation of the exception:

	public class CompanyAlreadyExistsException : Exception
	{
	    CompanyAlreadyExistsException () 
	      :base("A company with the same name and country already exists")
	    { 
		}
	}

Now, it looks better. Anyway, we have still room for improvements.

## Step 4 - Encapsulate The Business Rule Check By Employing 'The Specification Pattern'

The 'Specification Pattern' is a tactical design pattern presented in Eric Evans’ book Domain Driven Design. The `Specification Pattern` is a way of encapsulating business rule(s) and testing it against a candidate object to see if that object satisfies all requirements expressed in a specification. This pattern fits very good with the Single-Responsibility-Principle (SRP), which states that one class should have only one reason to change. Furthermore, this specification object can be easily unit tested and reused.  
  
Here, you can see how it is used:

	var spec = new UniqueCompanySpecification(_companyRepository);
	if (spec.IsSatisfiedBy(newCompany) == false)
		throw new CompanyAlreadyExistsException();

And the implementation details:
	
	public class UniqueCompanySpecification : ISpecification<Company>
	{
		readonly ICompanyRepository _companyRepository;

		public UniqueCompanySpecification(ICompanyRepository companyRepository)
		{
			_companyRepository = companyRepository;
		}

		public bool IsSatisfiedBy(Company candidate)
		{
			var numberOfSameCompanies = _companyRepository.Query()
				.Where(x => x.Name == newCompany.Name && x.CountryId == newCompany.CountryId)
				.Count();
			bool isUnique = numberOfSameCompanies == 0;
			return isUnique;
		}
	}

	public interface ISpecification<T>
	{
		bool IsSatisfiedBy(T candidate);
	} 

After all re-factoring steps, the final code is as following:

	//..

	var newCompany = new Company(message.Name, message.CountryId);
	
	var spec = new UniqueCompanySpecification(_companyRepository);
	if (spec.IsSatisfiedBy(newCompany) == false)
		throw new CompanyAlreadyExistsException();

	session.Save(newCompany);
	//..

# Summary

In this article, I've shown a re-factoring process and usage of the Specification Pattern in order to satisfy an explicit business rule.     
The re-factoring steps we took:  

1. Solve the query performance issues
2. Make the `if` condition check explicit
3. Make the business rule violation explicit
4. Encapsulate the business rule check by employing the Specification Pattern

The Specification Pattern lets you decouple the design of requirements, fulfillment, and validation. It also allows you to make your system definitions more clear and declarative, but be careful not to fall into temptation to over-use it.

**References:**

- [Specification Pattern by Eric Evans and Martin Fowler](http://martinfowler.com/apsupp/spec.pdf)
- [https://en.wikipedia.org/wiki/Specification_pattern](https://en.wikipedia.org/wiki/Specification_pattern)
- [Book: Domain Driven Design, Tackling Complexity In The Hearth of Software - by Eric Evans](http://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)

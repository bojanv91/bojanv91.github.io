---
layout: post
title: Opinionated Request/Response objects with Metadata for every-day use with ASP.NET Web API
---

In this article, I will try to explain how I design and use `Request / Response` classes in my default web API design approach.

Working with web apps, I can see some usage patterns emerge in how we structure the `Response` classes. The most common response structures can be segregated in 3 forms: single, paged list and errors response. Besides the actual data, now days we also need additional context information or with other words - metadata. Let's dive in. <!--excerpt-->

> [Request/Response design pattern](http://www.servicedesignpatterns.com/ClientServiceInteractions/RequestResponse) is the most common pattern used in client-server interactions. It is used when the client must have an immediate response or wants the service to complete a task without delay. Request/Response begins when the client establishes a connection to the service. Once a connection has been established, the client sends its request and waits for a response. The service processes the request as soon as it is received and returns a response over the same connection.

Wikipedia explanation about metadata:

> Metadata is "data about data".

[TOC]

## Request / Response classes

### Request

Request data comes in two ways in HTTP APIs: form body or query string.
Luckily ASP.NET model binders are smart enough to understand how de-serialize form body or query string data to .NET CLR classes.

### Response: Single

In this example, user's intent is to create new person by specifying first and last name.

**POST Request - URI: api/people**

	{
		"firstName": "Pece",
        "lastName": "Deteto"
	}

**POST Response - HTTP Status Code: 200**

    {
        "metadata" {
        	"message": "Person is created."
        }
        "personId": 1,
    }

As a response the server returns the newly created person ID and metadata containing a success message. Returning such response makes things easier for the client apps developers. Why we are returning the person's ID is a common understanding, but why we are returning a message text stating the success of the operation is more interesting. 
We keep all success and error messages on the server, easily i18n-ed so the client apps don't need manage them at all. Let's say you have Android, iPhone, Windows Phone and Web App as consumers of your API. You want consistency in the operation messages (among other things), and you don't want to manage them on all platforms. API's messages are the source of truth. 
Why I am embedding `message` into `metadata` field? Why the `metadata` field? - It's simple, I don't know what the future requirements may give to us, so I make this assumption that in future requirement additional fields will be needed besides a `message` . Maybe you are making use of some parts of Hypermedia and you'll need to send the fields label names with their data types to your smart client, which can compose it's UI from the metadata. Maybe. This way I am sure that I reduce the risk of breaking changes.

Now let's look into the `metadata` usage for paged list items.

### Response: Paged list

**GET Request - URI: api/people**

	{
    	"pageNumber": 1,
        "pageSize": 10,
    	"sortBy": "firstName",
        "sortDirection": "asc",
	}

This request ultimately is sent as a query string:

	http://DOMAIN/api/people?pageNumber=1&pageSize=10&sortBy=firstName&sortDirection=asc

**GET Response - HTTP Status Code: 200**

    {
        "metadata" {
        	"message": "View list of people.",
            "pageNumber": 1,
            "pageSize: 10,
            "returnedCount": 10,
            "totalCount": 150,
            "pagesCount": 15,
            "prevPageNumber": null,
            "nextPageNumber": 2
        }
        "people": [
        	{
            	"personId": 1,
                "firstName": "Pece",
                "lastName": "Deteto"
            },
            {
            	"personId": 2,
                "firstName": "Sandra",
                "lastName": "Chupeto"
            },
            ...
        ]
    }

I'll leave it to yourself, dear reader, how smart your client apps can be with this just enough metadata.

### Response: Error

**POST Request - URI: api/people**

	{
        "lastName": "De"
	}

**POST Response - HTTP Status Code: 400**

	{
		"errors": [
			{
				"propertyName": "FirstName",
				"errorMessage": "First name is required.",
				"attemptedValue: ""
			},
            {
				"propertyName": "LastName",
				"errorMessage": "Last name must be greater than 4 characters.",
				"attemptedValue: "De"
			}
		]
	}

### Web API HTTP Status Codes

As it can be concluded from the above examples - most commonly I map the outcome from application operations to status codes like this:

- 200 when the operation completes successfully
- 400 when input validation or business logic error occurs
- 401 when unauthorized operation is requested
- 404 when the requested resource is not found
- 5** - when something bad happens on the server-side

---

## Implementation

### The WebAPI Controller

    [RoutePrefix("api/people")]
    public class PeopleController : BaseApiController
    {
        [HttpGet]
        [Route("{personId}")]
        public GetPerson.Response GetPerson(int personId)
        {
            GetPerson.Request request = new GetPerson.Request { PersonId = personId };
            GetPerson.Response response = CommandDispatcher.Dispatch(request);
            return response;
        }

		[HttpGet]
        [Route("")]
        public ListPeoplePaged.Response ListPeoplePaged(ListPeoplePaged.Request request)
        {
            return CommandDispatcher.Dispatch(request);
        }

        [HttpPost]
        [Route("")]
        public CreatePerson.Response CreatePerson(CreatePerson.Request request)
        {
            return CommandDispatcher.Dispatch(request);
        }
    }

*In the example above I make use of Command and Query pattern. I'll explain how I use it in my next article.*

### The Request/Response classes

    //Features/People/CreatePerson.cs
    public class CreatePerson
    {
        public class Request : CommandRequest<Response>
        {
            public string FirstName { get; set; }
            public string LastName { get; set; }
        }

        public class Response : CommandResponse
        {
            public int PersonId { get; set; }
        }
    }

	//Features/People/ListPeoplePaged.cs
    public class ListPeoplePaged
    {
        public class Request : CommandRequest<Response>
        {
            public int PageNumber { get; set; }
            public int PageSize { get; set; }
            public string SortBy { get; set; }
            public string SortDirection { get; set; }
        }

        public class Response : CommandResponsePaged
        {
            public IEnumerable<Item> People  { get; set; }

            public class Item
            {
            	public int PersonId { get; set; }
                public string FirstName { get; set; }
                public string LastName { get; set; }
            }
        }
    }

I like my feature classes cohesive and focused.
You can generate your API documentation with [Swashbuckle](https://www.nuget.org/packages/Swashbuckle) and inspect how to use your APIs in your client-side code.

### The Front-End Client with jQuery

The following is small piece of code service method demonstrating how you can work with our Web API with jQuery.

	//HTML
	<form id="#createUserForm">
		<input id="PersonId" name="PersonId" type="hidden" />
		<div>
			<label for="FirstName">First name</label>
			<input id="FirstName" name="FirstName" type="text" />
			<span id="ErrorPlaceholderFor_FirstName" class="hidden"></span>
		</div>
		<div>
			<label for="LastName">Last name</label>
			<input id="LastName" name="LastName" type="text" />
			<span id="ErrorPlaceholderFor_LastName" class="hidden"></span>
		</div>
	</form>
	
	//SERVICE DEFINITION
	postDataAsync(uri, data) {
		var deferred = $.Deferred();
		
		$.post(uri, data)
			.done((result) => {
				deferred.resolve(result.responseJSON);		//contains 'metadata' and other fields
			})
			.fail((result) => {
				deferred.reject(result.responseJSON);
			});
			
		return deferred.promise();
	}
	
	//USAGE FOR CREATE USER UI
	$("#createUserForm").on("submit", () => {
		var $thisForm = $(this);
		var data = $thisForm.serialize();
		
		postDataAsync("/api/people", data)
			.done((response) => {
				displaySuccessWith(response.metadata.message);
				$thisForm.find("#PersonId").val(response.personId);
			})
			
			.fail((response) => {
				var errorSummary = "";
				response.errors.forEach((error) => {
					var $errorPlaceholder = $thisForm.find("#ErrorPlaceholderFor_" + error.propertyName);
					$errorPlaceholder.html(error.errorMessage);
					$errorPlaceholder.removeClass("hidden");
					$errorPlaceholder.addClass("shown");
					
					errorSummary += errorMessage + "<br />";
				});
				displayErrorWith(errorSummary);
			});
	});

In the next article I will dig deeper in how I use Command and Query pattern in my applications.


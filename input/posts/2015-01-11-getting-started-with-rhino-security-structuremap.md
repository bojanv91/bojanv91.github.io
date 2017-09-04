Tags: [security]
RedirectFrom: 2015/01/getting-started-with-rhino-security-structuremap/
Lead: A guide for setting up Rhino Security library
Title: Getting Started with Rhino Security and Structure Map
---

In this posting I will show you how to configure Rhino Security infrastructure to work with StructureMap IoC container and provide to you database schema (for MSSQL and [FluentMigrator](http://bojanv91.github.io/2014/12/database-development-guidance/)) that you will need in order Rhino Security to get working. I've struggled some time before I got everything working, so here are my results. :) <!--excerpt-->

> [Rhino Security](https://github.com/ayende/rhino-security) is a security framework to provide row level security for NHibernate. Rhino Security is perfect for people who want to set up user and group security in their NHibernate domain models. 
> 
>> [Rhino Security GitHub repository](https://github.com/ayende/rhino-security)  

More details about the architecture and how Rhino Security works behind the scenes can be found [here](http://ayende.com/blog/2958/a-vision-of-enterprise-platform-security-infrastructure), [here](http://ayende.com/blog/3109/rhino-security-overview-part-i) and [here](http://ayende.com/blog/3113/rhino-security-part-ii-discussing-the-implementation).

# Action Plan

- Installing NuGet packages
- Configuring StructureMap container and registering Rhino.Security into NHibernate 
- Implementing CommonServiceLocator provider for StructureMap
- User entity that implements Rhino.Security.IUser interface
- Preparing the database schema
- Usage DEMO (code samples [https://github.com/bojanv91/RhinoSecurityWithStructureMap](https://github.com/bojanv91/RhinoSecurityWithStructureMap)) 

![Getting started with Rhino Security and StructureMap](/images/2015-01-11-getting-started-with-rhino-security-structuremap/rhino-01.png)
 
## Installing NuGet packages

	Install-Package Rhino.Security			
	Install-Package NHibernate
	Install-Package FluentNHibernate
	Install-Package StructureMap
	Install-Package CommonServiceLocator

FluentNHibernate provides fluent mapping interface for mapping our domain model entities to table structures via NHibernate.  
CommonServiceLocator provides abstraction over IoC containers and service locators and contains a shared interface for service location. Rhino Security makes use of it, that is why can be used with any IoC container. 

## Configuring StructureMap container and registering Rhino.Security into NHibernate 

In the following code snippet I have all configuration stuff in one class, called the Bootstrapper. It's purpose is to provide functionality for booting up our application, the starting point. Explanations about what does what are put in comments. If something is still unclear do write me a comment, I'll happily update that part. 

	using FluentNHibernate.Cfg;
	using FluentNHibernate.Cfg.Db;
	using NHibernate;
	using NHibernate.Cfg;
	using NHibernate.Context;
	using Rhino.Security.Interfaces;
	using Rhino.Security.Services;

	namespace RhinoSecurityWithStructureMap
	{
	    public class Bootstrapper
	    {
	        public static void Bootstrap(string connectionString)
	        {
	            var container = new StructureMap.Container();
	            container.Configure(cfg =>
	                {
	                    //NHibernate configurations 
	                    cfg.For<ISessionFactory>().Singleton().Use(() => CreateSessionFactory(connectionString));
	                    cfg.For<ISession>().Use(context => GetSession(context));
	
	                    //Rhino Security configurations 
	                    cfg.For<IAuthorizationService>().Use<AuthorizationService>();
	                    cfg.For<IAuthorizationRepository>().Use<AuthorizationRepository>();
	                    cfg.For<IPermissionsBuilderService>().Use<PermissionsBuilderService>();
	                    cfg.For<IPermissionsService>().Use<PermissionsService>();
	                });
	
	            //Setting up StuctureMapServiceLocator as a CommonServiceLocator that Rhino.Security will use for DI
	            Microsoft.Practices.ServiceLocation.ServiceLocator
	                .SetLocatorProvider(() => new StructureMapServiceLocator(container));
	        }
	
	        private static ISessionFactory CreateSessionFactory(string connectionString)
	        {
	            FluentConfiguration fluentConfig = Fluently.Configure()
	                .Database(MsSqlConfiguration.MsSql2012.ConnectionString(connectionString))          //specifying connection string for Microsoft SQL Server 2012 
	                .Mappings(m => m.FluentMappings.AddFromAssemblyOf<Bootstrapper>())                  //specifying in which assembly NHibernate should look for entity mappings
	                .CurrentSessionContext(typeof(ThreadStaticSessionContext).AssemblyQualifiedName)    //specifying the session context lifecycle to be initialized per thread
	                .ExposeConfiguration(cfg =>
	                {
	                    Rhino.Security.Security.Configure<User>(cfg, Rhino.Security.SecurityTableStructure.Prefix);
	                });
	
	            return fluentConfig.BuildSessionFactory();
	        }
	
	        private static ISession GetSession(StructureMap.IContext context)
	        {
	            var sessionFactory = context.GetInstance<ISessionFactory>();
	            return sessionFactory.GetCurrentSession();
	        }
	    }
	}

## Implementing CommonServiceLocator provider for StructureMap

The code is pretty much straightforward. We just implement Microsoft.Practices.ServiceLocation.IServiceLocator interface with the common code that is provided to us by StructureMap IContainer interface, basically this class acts as a wrapper.

    public class StructureMapServiceLocator : Microsoft.Practices.ServiceLocation.IServiceLocator
    {
        private readonly IContainer _container;

        public StructureMapServiceLocator(IContainer container)
        {
            _container = container;
        }

        public IEnumerable<TService> GetAllInstances<TService>()
        {
            return _container.GetAllInstances<TService>();
        }

        public IEnumerable<object> GetAllInstances(Type serviceType)
        {
            return (IEnumerable<object>)_container.GetAllInstances(serviceType);
        }

        public TService GetInstance<TService>(string key)
        {
            return _container.GetInstance<TService>(key);
        }

        public TService GetInstance<TService>()
        {
            return _container.GetInstance<TService>();
        }

        public object GetInstance(Type serviceType, string key)
        {
            return _container.GetInstance(serviceType, key);
        }

        public object GetInstance(Type serviceType)
        {
            return _container.GetInstance(serviceType);
        }

        public object GetService(Type serviceType)
        {
            return _container.GetInstance(serviceType);
        }
    }


## User entity implements Rhino.Security.IUser interface

In our domain model we commonly have entity which represents the actual user. Rhino.Security must know which entity is the user entity in order to work. So our User entity must implement Rhino.Security.IUser interface, more precisely only SecurityInfo property from the interface must be implemented.

	public class User : Rhino.Security.IUser
	{
		public virtual int Id { get; protected set; }	    
		public virtual string Username { get; set; }
	    public virtual string PasswordHashed { get; set; }
	
	    public Rhino.Security.SecurityInfo SecurityInfo
	    {
	        get
	        {
	            return new Rhino.Security.SecurityInfo(Username, Id);
	        }
	    }
	}

## Preparing the database schema

Schema files can be found in the following links:

- [SQL dump](https://github.com/bojanv91/RhinoSecurityWithStructureMap/blob/master/RhinoSecurityWithStructureMap/DatabaseScripts/rhino_security_and_basic_user.sql.sql)
- [Fluent Migrator schema](https://github.com/bojanv91/RhinoSecurityWithStructureMap/blob/master/RhinoSecurityWithStructureMap/DatabaseScripts/rhino_security_and_basic_user.cs) ([blogpost about how to use Fluent Migrator](http://bojanv91.github.io/2014/12/database-development-guidance/))

![Rhino database schema](/images/2015-01-11-getting-started-with-rhino-security-structuremap/rhino-02.png)

## Usage DEMO

The full code sample can be found in following github repository: [https://github.com/bojanv91/RhinoSecurityWithStructureMap](https://github.com/bojanv91/RhinoSecurityWithStructureMap). Here I provide excerpt code snippets from the actual test code.

Setting up user groups, operations and permissions:

	var _authorizationRepository = ServiceLocator.Current.GetInstance<IAuthorizationRepository>();
	var _authorizationService = ServiceLocator.Current.GetInstance<IAuthorizationService>();
	var _permissionsBuilderService = ServiceLocator.Current.GetInstance<IPermissionsBuilderService>();
	var _permissionService = ServiceLocator.Current.GetInstance<IPermissionsService>();

    using (var transaction = _session.BeginTransaction())
    {
        //creating user group 'Admin'
        _authorizationRepository.CreateUsersGroup("Admin");

        //creating operations
        _authorizationRepository.CreateOperation("/Content");
        _authorizationRepository.CreateOperation("/Content/Create");
        _authorizationRepository.CreateOperation("/Content/View");
        _authorizationRepository.CreateOperation("/Content/Delete");

        transaction.Commit();
    }

    using (var transaction = _session.BeginTransaction())
    {
        //adding the LoggedInUser to the 'Admin' users group
        _authorizationRepository.AssociateUserWith(_loggedInUser, "Admin");

        //Building 'Allow' permissions for the LoggedInUser, 
        //by default if not defined as allowed, the operation is denied
        //For the sake of this example, we say the the users that are in 'Admin' users group can
        //create and view content, but cannot delete content. 
        _permissionsBuilderService.Allow("/Content/Create").For("Admin").OnEverything().DefaultLevel().Save();
        _permissionsBuilderService.Allow("/Content/View").For("Admin").OnEverything().DefaultLevel().Save();

        //We can explicitly define 'Deny' permission, but as the default behaviour denies everything 
        //that is not defined as 'Allow', I am not going to define it. You don't trust me? 
        //That's why we have tests ;) 
        //_permissionsBuilderService.Deny("/Content/Delete").For("Admin").OnEverything().DefaultLevel().Save();

        transaction.Commit();
    }

Test code demonstrating the usage:

    public class RhinoTests : IUseFixture<TestFixture>
    {
        private readonly IAuthorizationService _authorizationService;
        private readonly User _loggedInUser;

        public RhinoTests() 
        {
            _authorizationService = ServiceLocator.Current.GetInstance<IAuthorizationService>();
            _loggedInUser = TestFixture._loggedInUser;
        }

        [Fact]
        public void it_should_allow_content_creation()
        {
            bool result = _authorizationService.IsAllowed(_loggedInUser, "/Content/Create");
            Assert.True(result);
        }

        [Fact]
        public void it_should_allow_content_viewing()
        {
            bool result = _authorizationService.IsAllowed(_loggedInUser, "/Content/View");
            Assert.True(result);
        }

        [Fact]
        public void it_should_deny_content_deletition()
        {
            bool result = _authorizationService.IsAllowed(_loggedInUser, "/Content/Delete");
            Assert.False(result);
        }

        public void SetFixture(TestFixture data) { }
    }

---

Happy coding folks! Having questions or concerns? Shoot me a tweet -> @bojanv91
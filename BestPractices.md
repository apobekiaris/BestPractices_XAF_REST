# Best Practices

## Non-Secured Endpoints

### Server Time

To check the server's current time across different time zones, a simple `GET` request can be used. This request doesn't require the `XafApplication` context. Note, that the [AuthorizeAttribute](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.authorization.authorizeattribute?view=aspnetcore-7.0) can restrict this operation to authenticated users.

   ```cs
   [HttpGet("api/Custom/ServerTime/{timezone}")]
   //[Authorize]
   public ActionResult<string> GetServerTime(string timezone) {
       try {
           TimeZoneInfo tz = TimeZoneInfo.FindSystemTimeZoneById(timezone);
           DateTime serverTime = TimeZoneInfo.ConvertTimeFromUtc(DateTime.UtcNow, tz);
           return Ok($"Server time in {timezone}: {serverTime}");
       }
       catch (TimeZoneNotFoundException) {
           return BadRequest($"Invalid timezone: {timezone}");
       }
   }
   ```

### Clear Logs

To clear logs on the server, use a `POST` request. The [AuthorizeAttribute](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.authorization.authorizeattribute?view=aspnetcore-7.0) can restrict this operation to authenticated users.

  ```cs
  [HttpPost(nameof(ClearLogs))][Authorize]
  [SwaggerOperation("Clears logs older than today")]
  public IActionResult ClearLogs() {
      try {
          var logDirectory = @"C:\path\to\your\logs";
          var di = new DirectoryInfo(logDirectory);
          foreach (var file in di.GetFiles()) {
              if (file.CreationTime < DateTime.Today) {
                  file.Delete();
              }
          }
          return Ok(new { status =  "Logs older than today have been deleted successfully"  });
      }
      catch (Exception e){
          return BadRequest(e);
      }
  }
  ```

### Database Connection Health Check

To check the health of the database connection, use a `GET` request. The `INonSecuredObjectSpaceFactory` can work outside of the Xaf security context for this operation.

  ```cs
  [ApiController]
  [Route("api/[controller]")]
  public class CustomController : ControllerBase {
      private readonly INonSecuredObjectSpaceFactory _nonSecuredObjectSpaceFactory;
      public CustomController(INonSecuredObjectSpaceFactory nonSecuredObjectSpaceFactory) => _nonSecuredObjectSpaceFactory = nonSecuredObjectSpaceFactory;
  
      [HttpGet(nameof(DbConnectionHealthCheck))]
      [SwaggerOperation("returns the current database connection health")]
      public IActionResult DbConnectionHealthCheck() {
          try {
              using var objectSpace = _nonSecuredObjectSpaceFactory.    CreateNonSecuredObjectSpace(typeof(ApplicationUser));
              return Ok(new { status =  "Healthy"  });
          }
          catch (Exception e) {
              return StatusCode(500,e.Message);
          }      
      }
    }
  ```

## Secured Endpoints

### Current User Identifier

To get the current user identifier, use a `GET` request. This operation uses `ISecurityProvider`.
  
  ```cs
  [ApiController]
  [Route("api/[controller]")]
  public class CustomController : ControllerBase {
      private readonly ISecurityProvider _securityProvider;
      public CustomController(ISecurityProvider securityProvider) => _securityProvider = securityProvider;
  
      [HttpGet()]
      [SwaggerOperation("returns the current user identifier")]
      public IActionResult GetUserId() 
          => Ok(_securityProvider.GetSecurity().UserId);
  }
  ```

### Employee Details

To retrieve persistent Business entities with serialization by projecting to anonymous DTO objects, use `GET` requests. This operation uses `IObjectSpaceFactory` and respects the Xaf application security configuration.

  ```cs
  [ApiController]
  [Route("api/[controller]")]
  public class CustomController : ControllerBase {
      private readonly IObjectSpaceFactory _objectSpaceFactory;
      public CustomController(IObjectSpaceFactory objectSpaceFactory) => _objectSpaceFactory = objectSpaceFactory;
  
      [HttpGet(nameof(Employee)+"/{id}")]
      [SwaggerOperation("returns an Employee by its id")]
      public ActionResult GetEmployee(int id) {
          using var objectSpace = _objectSpaceFactory.CreateObjectSpace(typeof     (Employee));
          var employee = objectSpace.GetObjectByKey<Employee>(id);
          return employee == null ? NotFound($"Employee ({id}) not found.") : Ok   (new  {employee.EmployeeId,employee.DepartmentName});
      } 
  
      [HttpGet(nameof(Employee)+"/{department}")]
      [SwaggerOperation("returns all Employees of a department")]
      public ActionResult GetEmployees(string department) {
      using var objectSpace = _objectSpaceFactory.CreateObjectSpace(typeof     (Employee));
          return Ok(objectSpace.GetObjectsQuery<Employee>()
          .Select(employee => new { employee.EmployeeId, employee.     DepartmentName })
              .Where(employee => employee.DepartmentName == department));
      }
  }
  ```
  
  > Xaf does a similar projection automatically for Create, Read, Delete, Update if your Employee is register to be part of the OData model in the Startup.cs.

  ```cs
  services.AddXafWebApi(options => options.BusinessObject<Employee>());
  ```

  > If you want to intercept those default endpoints see [Execute Custom Operations on Endpoint Requests](https://docs.devexpress.com/eXpressAppFramework/403850/backend-web-api-service/execute-custom-operations).
  
  > Advanced: [Use OData serialization of a business object in your custom Web API methods](https://supportcenter.devexpress.com/internal/ticket/details/T1041495)

### Stream Employee Photo

To stream the `Employee` Photo (`bytes`), use a `GET` request.

  ```cs
  [HttpGet("EmployeePhoto/{employeeId}")]
  public FileStreamResult AuthorPhoto(int employeeId) {
      using var objectSpace = _objectSpaceFactory.CreateObjectSpace(typeof(Employee));
      return File(new MemoryStream(objectSpace.GetObjectByKey<Employee>(employeeId).  Photo), "application/octet-stream");
  }
  ```
  
### Create a New ApplicationUser

To create a new `ApplicationUser`, use a `POST` request. This operation uses the `ISecurityProvider` to query for `create permissions` and the `IObjectSpaceFactory` to query for `duplicates`.
  
  ```cs
  [ApiController]
  [Route("api/[controller]")]
  public class CustomController : ControllerBase {
      private readonly IObjectSpaceFactory _objectSpaceFactory;
      private readonly ISecurityProvider _securityProvider;
      public CustomController(IObjectSpaceFactory objectSpaceFactory,ISecurityProvider securityProvider) {
          _objectSpaceFactory = objectSpaceFactory;
          _securityProvider=securityProvider;
      }

      [HttpPost(nameof(CreateUser)+"/{userName}")]
      [SwaggerOperation("Create a new user from a username")]
      public IActionResult CreateUser(string userName) {
          var strategy = (SecurityStrategy)_securityProvider.GetSecurity();
          if (!strategy.CanCreate(typeof(ApplicationUser)))
              return Forbid("You do not have permissions to create a user");
        using var objectSpace = _objectSpaceFactory.CreateObjectSpace(typeof  (ApplicationUser));
        if (objectSpace.GetObjectsQuery<ApplicationUser>().Any(user => user.  UserName == userName))
              return ValidationProblem("Username exists");
          var applicationUser = objectSpace.CreateObject<ApplicationUser>();
          applicationUser.UserName = userName;
          var random = new Random();
        var password = new string(Enumerable.Range(0, 5).Select(_ => (char)  random.Next(33, 127)).ToArray());
          applicationUser.SetPassword(password);
          objectSpace.CommitChanges();
          return Ok(password);
      }
    }
  ```


  **DOES IT MAKE SENSE**
  ![](images/400.png)


# Best Practices

## Non-Secured Endpoints

### Server Time

To check the server's current time across different time zones, a simple `GET` request can be used. This request doesn't require the `XafApplication` context. Note, that the [AuthorizeAttribute](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.authorization.authorizeattribute?view=aspnetcore-7.0) can restrict this operation to authenticated users.

   ```cs
   [HttpGet("api/Custom/ServerTime/{timezone}")]
//    [Authorize]
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
  [HttpPost(nameof(ClearLogs))]
//   [Authorize]
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
      [Authorize]
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
      [Authorize]
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
      [Authorize]
      public ActionResult GetEmployee(int id) {
          using var objectSpace = _objectSpaceFactory.CreateObjectSpace(typeof     (Employee));
          var employee = objectSpace.GetObjectByKey<Employee>(id);
          return employee == null ? NotFound($"Employee ({id}) not found.") : Ok   (new  {employee.EmployeeId,employee.DepartmentName});
      } 
  
      [HttpGet(nameof(Employee)+"/{department}")]
      [SwaggerOperation("returns all Employees of a department")]
      [Authorize]
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
  [Authorize]
  public FileStreamResult EmployeePhoto(int employeeId) {
      using var objectSpace = _objectSpaceFactory.CreateObjectSpace(typeof(Employee));
      var bytes=objectSpace.GetObjectByKey<Employee>(employeeId).Photo;
      return File(new MemoryStream(bytes), "application/octet-stream");
  }
  ```
  
### Create a New Employee

To create a new `Employee`, use a `POST` request. This operation uses the `ISecurityProvider` to query for `create permissions` and the `IObjectSpaceFactory` to query for `duplicates`.
  
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

      [HttpPost(nameof(CreateUserEmployee)+"/{email}")]
      [SwaggerOperation("Create a new user from a email")]
      [Authorize]
      public IActionResult CreateUserEmployee(string email) {
          var strategy = (SecurityStrategy)_securityProvider.GetSecurity();
          if (!strategy.CanCreate(typeof(Employee)))
              return Forbid("You do not have permissions to create an employee!");
          using var objectSpace = _objectSpaceFactory.CreateObjectSpace(typeof      (Employee));
          if (objectSpace.FirstOrDefault<Employee>( e => e.Email == email)!=null)
              return ValidationProblem("Email is already registered!");
          var employee = objectSpace.CreateObject<Employee>();
          employee.Email = email;
          objectSpace.CommitChanges();
          return Ok();
      }
    }
  ```

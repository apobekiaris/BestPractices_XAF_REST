Goes below https://docs.devexpress.com/eXpressAppFramework/403715/backend-web-api-service/use-odata-to-send-requests/use-http-client-to-access-web-api#get-a-jwt-authentication-token

## OAuth2 Code Grant Flow with `ITokenAcquisition`

Similarly, if you use `OAuth2` to obtain the access token follow these steps:

### 1. Register the `ITokenAcquisition` Service

In your application's startup code, register the `ITokenAcquisition` service by adding the following code to the `ConfigureServices` method:

```csharp
services.AddMicrosoftIdentityWebApi(Configuration, configSectionName: "Authentication:AzureAd", jwtBearerScheme: "AzureAd")
    .EnableTokenAcquisitionToCallDownstreamApi()
    .AddInMemoryTokenCaches(); // Add this line to register the ITokenAcquisition service
```


### 2. Acquire the Access Token

Inject the `ITokenAcquisition` service into your context constructor or use the `XafApplication.ServiceProvider` to resolve it:

```csharp
private readonly ITokenAcquisition _tokenAcquisition;

public YourContextConstructor(ITokenAcquisition tokenAcquisition)
{
    _tokenAcquisition = tokenAcquisition;
}
```

or

```csharp
_tokenAcquisition = application.ServiceProvider.GetRequiredService<ITokenAcquisition>();
```

Then, use the `GetAccessTokenForUserAsync` method of the `ITokenAcquisition` service to acquire the access token:

```csharp
var scopes = new[] { "scope1", "scope2" }; // Replace with the required scopes
var oAuthToken = await _tokenAcquisition.GetAccessTokenForUserAsync(scopes);
```


### 3. Create and Send the Request

Create an instance of `HttpRequestMessage` and set the authorization header using the obtained access token:

```csharp
string endPointAddress = "https://your-endpoint.com/api/endpoint"; 
using var httpRequestMessage = new HttpRequestMessage(HttpMethod.Get, endPointAddress);
httpRequestMessage.Headers.Authorization = new AuthenticationHeaderValue("Bearer", oAuthToken);

using var responseMessage = await _httpClient.SendAsync(httpRequestMessage);
```

By following these steps, you can obtain an access token using the OAuth2 code grant flow and use it to make authenticated requests to your API.

Please note that the above example assumes you have already completed the OAuth2 authorization process and obtained the authorization code. The code snippet focuses on exchanging the authorization code for an access token using the `ITokenAcquisition` service.

Feel free to adjust the code and replace the placeholder values with your actual configuration and endpoint details.
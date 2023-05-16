# OAuth2 code grant flow

This example demonstrates how to authenticate using OAuth2 with the authorization code grant flow

```cs
    HttpClient httpClient = new HttpClient();

    // Obtain an access token using the authorization code grant flow.
    string authorizationEndpoint = "https://your-authentication-endpoint.com/oauth2/authorize";
    string tokenEndpoint = "https://your-authentication-endpoint.com/oauth2/token";
    string clientId = "your-client-id";
    string clientSecret = "your-client-secret";
    string redirectUri = "https://your-redirect-uri.com";
    string authorizationCode = "your-authorization-code";

    // Exchange the authorization code for an access token.
    var tokenRequestContent = new FormUrlEncodedContent(new[] {
        new KeyValuePair<string, string>("grant_type", "authorization_code"),
        new KeyValuePair<string, string>("code", authorizationCode),
        new KeyValuePair<string, string>("client_id", clientId),
        new KeyValuePair<string, string>("client_secret", clientSecret),
        new KeyValuePair<string, string>("redirect_uri", redirectUri)
    });
    var tokenResponse = await httpClient.PostAsync(tokenEndpoint, tokenRequestContent);
    var tokenContent = await tokenResponse.Content.ReadAsStringAsync();

    // Parse the access token from the token response.
    var token = JObject.Parse(tokenContent)["access_token"].Value<string>();

    // Set the access token in the authorization header.
    httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
```


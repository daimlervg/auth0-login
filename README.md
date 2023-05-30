# Auth0 Quickstart Sample

This example shows how to add login/logout using Auth0 and extract user profile information from claims.

## Requirements

- [.NET SDK](https://dotnet.microsoft.com/download) (.NET Core 3.1 or .NET 5.0+)

## To run this project

1. Ensure that you have replaced the `appsettings.json` file with the values for your Auth0 account. (Domain and ClientId)

2. Run the application from the command line or you can skip the following step if you are running the solution from VS Studio:

```bash
dotnet run
```

3. Go to `http://localhost:3000` in your web browser to view the website.

## Run this project with Docker (Optional Step)

In order to run the example with Docker you need to have [Docker](https://docker.com/products/docker-desktop) installed.

To build the Docker image and run the project inside a container, run the following command in a terminal, depending on your operating system:

```
# Mac
sh exec.sh

# Windows (using Powershell)
.\exec.ps1
```

## Important Snippets

### 1. Register the Auth0 SDK in program.cs 

To enable authentication in your ASP.NET Core application, use the middleware provided by the SDK. Go to the Program.cs file and call builder.Services.AddAuth0WebAppAuthentication() to configure the Auth0 ASP.NET Core SDK.
Ensure to configure the Domain and ClientId, these are required fields to ensure the SDK knows which Auth0 tenant and application it should use.


```csharp
var builder = WebApplication.CreateBuilder(args);
// Cookie configuration for HTTP to support cookies with SameSite=None
builder.Services.ConfigureSameSiteNoneCookies();

// Cookie configuration for HTTPS
//  builder.Services.Configure<CookiePolicyOptions>(options =>
//  {
//     options.MinimumSameSitePolicy = SameSiteMode.None;
//  });
builder.Services.AddAuth0WebAppAuthentication(options =>
{
    options.Domain = builder.Configuration["Auth0:Domain"];
    options.ClientId = builder.Configuration["Auth0:ClientId"];
});
builder.Services.AddControllersWithViews();
var app = builder.Build();
```
Note: Get your domain and clientId from your Auth0 account -> Applications/Settings

### 2. Register the Authentication middleware

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    ...
    app.UseAuthentication();
    app.UseAuthorization();
    ...
}
```
### 3. Login (AccountController)

To add the Login, call ChallengeAsync and pass "Auth0" (Auth0Constants.AuthenticationScheme) as the authentication scheme. This will invoke the OIDC authentication handler that our SDK registers internally.
After the OIDC middleware signs the user in, the user is also automatically signed in to the cookie middleware. This allows the user to be authenticated on subsequent requests.

```csharp
public async Task Login(string returnUrl = "/")
{
    var authenticationProperties = new LoginAuthenticationPropertiesBuilder()
        .WithRedirectUri(returnUrl)
        .Build();

    await HttpContext.ChallengeAsync(Auth0Constants.AuthenticationScheme, authenticationProperties);
}

```

### 4. User Profile

The SDK extracts the user's information from the ID Token and makes them available as the User.Claims property on the controller.

You can create a custom user profile page for displaying a user's name, email address, and profile image, by passing the corresponding information to the view from inside your controller.

```csharp
[Authorize]
public IActionResult Profile()
{
    return View(new UserProfileViewModel()
    {
        Name = User.Claims.FirstOrDefault(c => c.Type == ClaimTypes.Name)?.Value,
        EmailAddress = User.Claims.FirstOrDefault(c => c.Type == ClaimTypes.Email)?.Value,
        ProfileImage = User.Claims.FirstOrDefault(c => c.Type == "picture")?.Value
    });
}
```
You can check user profile: http://localhost:3000/Account/Profile
### 5. Logout

To add Logout, you need to sign the user out of both the Auth0 middleware as well as the cookie middleware.

Auth0 should redirect the user after a logout.
Note that the resulting absolute Uri must be whitelisted in the **Allowed Logout URLs** settings for the client.
In This case the allowed logout url will be: http://localhost:3000/Account/Login 

```csharp
[Authorize]
public async Task Logout()
{
    var authenticationProperties = new LogoutAuthenticationPropertiesBuilder()
        .WithRedirectUri(Url.Action("Index", "Home"))
        .Build();
        
    await HttpContext.SignOutAsync(Auth0Constants.AuthenticationScheme, authenticationProperties);
    await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
}
```

### 6. Claims User 

A Page displays all the claims associated the the current User. This is useful when debugging to see which claims are being populated from the Auth0 ID Token

http://localhost:3000/Account/Claims


## Configure Auth0

### 1. Configure Callback URLs (Application URIs -> Allowed Callback URLs)

The Callback URL of your application is the URL where Auth0 will redirect to after the user has authenticated in order for the SDK to complete the authentication process.

You will need to add this URL to the list of Allowed URLs for your application in your Application Settings, this URL will mostly take the format https://YOUR_APPLICATION_URL/callback.

In this case the URL will be ```http://localhost:3000.```

### 2. Install dependencies

```
Install-Package Auth0.AspNetCore.Authentication

https://github.com/auth0/auth0-aspnetcore-authentication
```

## Extra info

### 1. Custom Claums

Go to Auth0 Dashboard -> Action -> Library -> Create a new Custom Action with the following

Using the following approach you can use custom claims
```
/**
* Handler that will be called during the execution of a PostLogin flow.
*
* @param {Event} event - Details about the user and the context in which they are logging in.
* @param {PostLoginAPI} api - Interface whose methods can be used to change the behavior of the login.
*/
exports.onExecutePostLogin = async (event, api) => {
   const namespace = 'https://mysuperApp.com';
    if (event.authorization) {
      api.idToken.setCustomClaim(`${namespace}/roles`, event.authorization.roles);
      api.accessToken.setCustomClaim(`${namespace}/roles`, event.authorization.roles);
      api.idToken.setCustomClaim(`${namespace}/app_metadata`, event.user.app_metadata);
      api.accessToken.setCustomClaim(`${namespace}/app_metadata`, event.user.app_metadata);
    }
};


/**
* Handler that will be invoked when this action is resuming after an external redirect. If your
* onExecutePostLogin function does not perform a redirect, this function can be safely ignored.
*
* @param {Event} event - Details about the user and the context in which they are logging in.
* @param {PostLoginAPI} api - Interface whose methods can be used to change the behavior of the login.
*/
// exports.onContinuePostLogin = async (event, api) => {
// };
```
In the above example we have configured roles and app_metadata 

You also can test your code, save draft and deploy it.

### 2. App Metadata

You can add extra data using app metadata in (User Management -> Details -> app_metadata), 

this is an app_metadata example:
```json
{
  "permissions": [
    "catalogs_read",
    "catalogs_update"
  ]
}
```
You will get it in the user's claims:

```
Claim: https://mysuperApp.com/app_metadata	
value: {"permissions":["catalogs_read","catalogs_update"]}
```

### 3. Roles

You can add Roles using User Management -> Choose your user -> Roles and fill it with Roles.

In this example we are gonna add 

- Name: Admin 
- Description: Administrador del Sistema
- Assignment: Direct



### 3. Role-Based Access Control for APIs

You can enable role-based access control (RBAC) using the Auth0 Dashboard or the Management API. This enables the API Authorization Core feature set.

- Go to Dashboard > Applications > APIs and click the name of the API to view.
- Scroll to RBAC Settings and enable the Enable RBAC toggle.
- To include all permissions assigned to the user in the permissions claim of the access token, enable the Add Permissions in the Access Token toggle, and click Save. Including permissions in the access token allows you to make minimal calls to retrieve permissions, but increases token size.

The permissions will be included in the access token but wont be included in id token or userinfo

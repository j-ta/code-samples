#Token-based Auth in WebApi within an Existing Web App

Typical code samples show you how to add authentication to a new standalone WebApi project. What do you do if a WebApi has been added to a web application (web forms and/or MVC) that is using forms authentication? You can set up new API controllers to use token-based auth and then gradually migrate the older API controllers when appropriate. 

There are Microsoft libraries that can do this, though there have been enough changes over the years that it can be hard to figure out which one to use. They are also known imprecisely by terms like WIF (Windows Identity Foundation), Identity, Identity 2, and Identity 3 (seems to be in-progress). This describes an implementation using Owin. 

##1. Add the nuget packages

* Microsoft.AspNet.Identity.Core
* Microsoft.Owin
* Microsoft.Owin.Security
* Microsoft.Owin.Security.OAuth
* Owin

##2. Configure Owin auth
Create Startup.cs and add a MapOwinRoute to your route config to enable the TokenEndpointPath specified below.

```C#
 public void ConfigureAuth(IAppBuilder app)
        {
            // Configure the db context and user manager to use a single instance per request
            app.CreatePerOwinContext(ApplicationDbContext.Create);
            app.CreatePerOwinContext<ApplicationUserManager>(ApplicationUserManager.Create);

            // Enable the application to use a cookie to store information for the signed in user
            // and to use a cookie to temporarily store information about a user logging in with a third party login provider
            app.UseCookieAuthentication(new CookieAuthenticationOptions());
        
            // Configure the application for OAuth based flow
            PublicClientId = "self";
            OAuthOptions = new OAuthAuthorizationServerOptions
            {
                TokenEndpointPath = new PathString("/Token"),
                Provider = new ApplicationOAuthProvider(PublicClientId),
                AuthorizeEndpointPath = new PathString("/api/Account/ExternalLogin"),
                AccessTokenExpireTimeSpan = TimeSpan.FromDays(1),
                AllowInsecureHttp = false
            };

            // Enable the application to use bearer tokens to authenticate users
            app.UseOAuthBearerTokens(OAuthOptions);
        }

```

##3. Implement OAuthAuthorizationServerProvider
The ApplicationOAuthProvider above is your custom implementation of an OAuth provider. This is where you put your logic to validate credentals, add claims, and sign in. To implement the resource owner OAuth flow, override `GrantResourceOwnerCredentials`. 

For ease of backward-compatibility with the older roles-based identity model, you can add claims of `ClaimsType.Role`. This will allow you to re-use older methods on Identity like `IsInRole`.

##3. Add call to your new token endpoint
The POST request looks like the following:

```

Header:
Content-Type: application/x-www-form-urlencoded

Body:
grant_type: password
username: myUser
password: myPassword 

```

You will get back an `access_token` that you can then include in an `Authorize` header as "Bearer {access_token}".

#4. Set authorize attribute on Api
The end result is you can decorate any ApiController with the built-in authorize attribute, and it will automatically unpack the bearer token, set your claims principal, and authorize your class or method based upon the roles you set

```C#

//Sample attribute
[Authorize(Roles = "AdminUser")]
public string Get(int id)
{
    return "value";
}

``` 

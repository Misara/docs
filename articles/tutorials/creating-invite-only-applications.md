---
description: This scenario demonstrates an invite-only sign-up implementation using the Auth0 Management API to customize the process and the email flow.
toc: true
crews: crew-2
---

# Invite-only Applications

Self-service provisioning is a common concept for SaaS applications. Users can register and pay and then begin using the application.

Other types of applications (such as Google Apps and Office 365) may not allow single users to register for an application. Instead, customers may be organizations that pay upfront for a number of users and only want to allow those users access to your application. In these cases, an invite-only workflow can be used.

## Analystick scenario

Analystick is a multi-tenant SaaS solution offering analytics in the cloud. Their customers send them a list of users (with their given name, family name, and email address) that can access the application.

This functionality can be achieved using an Enterprise Connection where you federate with your customer using ADFS/SAML-P/…. This will allow your customer to authenticate users with their own Active Directory which specifies who is to be given access to the application.

The invite-only flow will be setup as follows:

1. The tenant administrator will create new users in their subscription from within the application. 
2. The application will call the Auth0 Management API to create these new users in a database connection. 
3. The application will send out activation emails to these users. 
4. When users click the activation link, they will be redirected to Auth0 where their email address will be set to validated. 
5. After validation, Auth0 will redirect users to the application where they will be presented with a password reset form. 
6. The application will update each user's password in Auth0, after which they will be able to authenticate.

![](/media/articles/invite-only/invite-only-overview.png)

### Setup

Users can be stored in a single database because Analystick will be signing up users from Contoso, Fabrikam and other companies with their corporate email addresses, making users unique for each customer.

![](/media/articles/invite-only/invite-only-connections.png)

To prevent users from signing up, select the **Disable Sign Ups** option on the connection to make sure users can only be created on the backend.

The Analystick application is an ASP.NET MVC web application hosted on `http://localhost:45000/`. You will need to create an application in the dashboard with the correct parameters:

 - **Name**: give your application a clear name as this will be used in the emails being sent out during the invite-only workflow
 - **Application Type**: this will be a regular web application.
 - **Allowed Callback URLs**: this should be the url of your application followed with `/signin-auth0` (a requirement of the Auth0.Owin NuGet package for .NET)

![](/media/articles/invite-only/invite-only-app.png)

Since this client application will access the [Auth0 Management API v2](/api/v2), you will need to authorize it:

* Go to the [API section](${manage_url}/#/apis) on the dashboard.
* Select _Auth0 Management API_.
* Click on the *Non-interactive Clients* tab.
* Look for the Client you created before and move the toggle to `Authorized`.
* In the *Scopes* list select `read:users`, `update:users`, `delete:users`, `create:users`, and `create:user_tickets`.
* Click **Update** to save the changes.

![Authorize Client](/media/articles/invite-only/invite-only-authorize-client.png)

### GitHub repository

A full working sample of the application can be found at [this GitHub repository](https://github.com/auth0-samples/auth0-invite-only-sample).

### User Management

Analystick has built a simple user interface in their admin backend to allow the import of up to 5 users.

![](/media/articles/invite-only/invite-only-new.png)

This admin interface uses the [Auth0.ManagementAPI SDK](https://www.nuget.org/packages/Auth0.ManagementApi/) to communicate with the Auth0 Management API v2:

```cs
public class UsersController : Controller
{
    private ManagementApiClient _client;

    private async Task<ManagementApiClient> GetApiClient()
    {
        if (_client == null)
        {
            var token = await (new ApiTokenCache()).GetToken();
            _client = new ManagementApiClient(
                token,
                ConfigurationManager.AppSettings["auth0:Domain"]);
        }

        return _client;
    }

    public async Task<ActionResult> Index()
    {
        var client = await GetApiClient();
        return View((await client.Users.GetAllAsync(connection: ConfigurationManager.AppSettings["auth0:Connection"]))
            .Select(u => new UserModel {
                UserId = u.UserId,
                GivenName = u.FirstName,
                FamilyName = u.LastName,
                Email = u.Email }
            ).ToList());
    }

    ...

    [HttpPost]
    public async Task<ActionResult> New(IEnumerable<UserModel> users)
    {
        if (users != null)
        {
            foreach (var user in users.Where(u => !String.IsNullOrEmpty(u.Email)))
            {
                var randomPassword = Guid.NewGuid().ToString();
                var metadata = new
                {
                    activation_pending = true
                };

                var client = await GetApiClient();
                var profile = await client.Users.CreateAsync(new Auth0.ManagementApi.Models.UserCreateRequest
                {
                    Email = user.Email,
                    Password = randomPassword,
                    Connection = ConfigurationManager.AppSettings["auth0:Connection"],
                    EmailVerified = true,
                    FirstName = user.GivenName,
                    LastName = user.FamilyName,
                    AppMetadata = metadata
                });

                var userToken = JWT.JsonWebToken.Encode(
                    new { id = profile.UserId, email = profile.Email },
                    ConfigurationManager.AppSettings["analystick:signingKey"],
                        JwtHashAlgorithm.HS256);

                var verificationUrlTicket = await client.Tickets.CreateEmailVerificationTicketAsync(
                    new Auth0.ManagementApi.Models.EmailVerificationTicketRequest
                    {
                        UserId = profile.UserId,

                        ResultUrl = Url.Action("Activate", "Account", new { area = "", userToken }, Request.Url.Scheme)
                    }
                );

                var body = "Hello {0}, " +
                    "Great that you're using our application. " +
                    "Please click <a href='{1}'>ACTIVATE</a> to activate your account." +
                    "The Analystick team!";

                var fullName = String.Format("{0} {1}", user.GivenName, user.FamilyName).Trim();
                var mail = new MailMessage("app@auth0.com", user.Email, "Hello there!",
                    String.Format(body, fullName, verificationUrlTicket.Value));
                mail.IsBodyHtml = true;

                var mailClient = new SmtpClient();
                mailClient.Send(mail);
            }
        }

        return RedirectToAction("Index");
    }

    [HttpPost]
    public async Task<ActionResult> Delete(string id)
    {
        if (!String.IsNullOrEmpty(id))
        {
            var client = await GetApiClient();
            await client.Users.DeleteAsync(id);
        }
        return RedirectToAction("Index");
    }
}
```

The `Users.CreateAsync` method is called to create the user in the database connection. This method is called with 5 parameters:

 1. The user’s email address
 2. The user’s password (a new Guid is assigned as a random password to the user)
 3. The name of the connection in which to create the user
 4. The email verified parameter (set to false prior to the user clicking the activation link).
 5. A metadata object containing the given name and family name of the user, and an activation pending setting (used later to validate the user).

### Emails

Once the user is created, you will need to send the verification email. The email will contain a link to the account activation URL of the application (/activate/account) that contains the password reset form. To securely identify the user, a JWT token is appended to the URL.

To generate the `verificationUrl`, call the `Tickets.CreateEmailVerificationTicketAsync` method on the SDK, set the return URL to that of the password reset form, and append the token.

::: note
Since you do not want the default emails to be sent, you must disable the **Verification Email** and **Welcome Email** in the Auth0 dashboard.
:::

![](/media/articles/invite-only/invite-only-disable-email.png)

The backend will be sending out the email, so you will need to access an SMTP server. This example uses Mailtrap, but any SMTP server will do.

After signing up with an email server, add these SMTP settings to the `web.config`:

```xml
  <system.net>
    <mailSettings>
      <smtp from="myapp@auth0.com">
        <network userName="1234567" password="89101112" port="2525" host="mailtrap.io" />
      </smtp>
    </mailSettings>
  </system.net>
```

That is all that is required for user provisioning.

Now go back to the user overview to start importing users.

![](/media/articles/invite-only/invite-only-users.png)

Each user will receive a welcome email containing a link to activate their account.

![](/media/articles/invite-only/invite-only-activation-mail.png)

### User Activation

The link in the email template redirects to Auth0 for email verification.

After successful verification, Auth0 will redirect the user to the password reset form of the application with the user token included in the URL.

![](/media/articles/invite-only/invite-only-activation.png)

Once the user has entered their password, you should verify that the account has not been updated yet, update the user's password, and mark the user as active (`activation_pending = false`).

```cs
/// <summary>
/// GET Account/Activate?userToken=xxx
/// </summary>
/// <param name="userToken"></param>
/// <returns></returns>
public async Task<ActionResult> Activate(string userToken)
{
    dynamic metadata = JWT.JsonWebToken.DecodeToObject(userToken,
        ConfigurationManager.AppSettings["analystick:signingKey"]);
    var user = await GetUserProfile(metadata["id"]);
    if (user != null)
        return View(new UserActivationModel { Email = user.Email, UserToken = userToken });
    return View("ActivationError",
        new UserActivationErrorModel("Error activating user, could not find an exact match for this email address."));
}

/// <summary>
/// POST Account/Activate
/// </summary>
/// <param name="model"></param>
/// <returns></returns>
[HttpPost]
public async Task<ActionResult> Activate(UserActivationModel model)
{
    dynamic metadata = JWT.JsonWebToken.DecodeToObject(model.UserToken,
        ConfigurationManager.AppSettings["analystick:signingKey"], true);
    if (metadata == null)
    {
        return View("ActivationError",
            new UserActivationErrorModel("Unable to find the token."));
    }

    if (!ModelState.IsValid)
    {
        return View(model);
    }

    User user = await GetUserProfile(metadata["id"]);
    if (user != null)
    {
        if (user.AppMetadata["activation_pending"] != null && !((bool)user.AppMetadata["activation_pending"]))
            return View("ActivationError", new UserActivationErrorModel("Error activating user, the user is already active."));

        var client = await GetApiClient();
        await client.Users.UpdateAsync(user.UserId, new UserUpdateRequest {
            Password = model.Password
        });
        await client.Users.UpdateAsync(user.UserId, new UserUpdateRequest
        {
            AppMetadata = new { activation_pending = false }
        });

        return View("Activated");
    }

    return View("ActivationError",
        new UserActivationErrorModel("Error activating user, could not find an exact match for this email address."));
}
```

::: note
Always validate the token first to be sure you are updating the correct user.
:::

Next, display a confirmation page where the user can click a link to sign in.

You can customize the rendering of Lock. Since you don’t want users to sign up, hide the Sign Up button which is visible by default:

```javascript
var lock = new Auth0Lock(
    '@System.Configuration.ConfigurationManager.AppSettings["auth0:ClientId"]',
    '@System.Configuration.ConfigurationManager.AppSettings["auth0:Domain"]', {
        auth: {
            redirectUrl: window.location.origin + '/signin-auth0'
        },
        allowSignUp: false
    });

function showLock() {
    lock.show();
}
```

![](/media/articles/invite-only/invite-only-login.png)

As a final step, you can enforce user activation by configuring Auth0 at application startup to intercept every login. This allows you to modify a user’s identity before handing it over to the OWIN pipeline.

This example checks if a user is active, and if so, adds the **Member** role to the user:

```cs
public partial class Startup
{
   public void ConfigureAuth(IAppBuilder app)
   {
      ...

      var provider = new Auth0AuthenticationProvider
      {
         OnReturnEndpoint = context =>
         {
             ..
         },
         OnAuthenticated = context =>
         {
             if (context.User["activation_pending"] != null)
             {
                 var pending = context.User.Value<bool>("activation_pending");
                 if (!pending)
                 {
                     context.Identity.AddClaim(new Claim(ClaimTypes.Role, "Member"));
                 }
             }

            ...

             return Task.FromResult(0);
         }
      };

      app.UseAuth0Authentication(ConfigurationManager.AppSettings["auth0:ClientId"],
        ConfigurationManager.AppSettings["auth0:ClientSecret"], ConfigurationManager.AppSettings["auth0:Domain"],
         provider: provider);
   }
}
```

Now you can ensure that these pages will only be accessible to users by enforcing the presence of a **Member** claim:

```cs
[Authorize(Roles = "Member")]
public class ProfileController : Controller
{
    public ActionResult Index()
    {
        ...
    }
}
```

Once users have completed the entire flow, they will be able to access the member-only pages.

![](/media/articles/invite-only/invite-only-profile.png)

### Summary

This scenario has demonstrated an invite-only sign-up implementation using the Auth0 Management API to customize the sign-up process and the email flow.

For more information about the Management API, see the [Management API Explorer](/api/v2) to try the various endpoints.

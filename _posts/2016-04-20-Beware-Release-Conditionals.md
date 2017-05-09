---
layout: post
title: Why you should avoid preprocessor directives  
---

Preprocessor directives are directions to the compiler to change the code at build time, which might sound amazing but has some drawbacks.  But let's ignore that for a moment and think about good situations where this sounds less-bad.  Maybe you have special test code.  Maybe for developing locally you want to allow more relaxed passwords or a longer session timeout.  Or maybe you want to enforce SSL on the login method in production but not during development. 

# A Non-Hypothetical Scenario

Consider the code below from an ASP.NET MVC Controller:

``` csharp
#if (!DEBUG) 
    [RequireHttps] 
#endif
    public ActionResult Login(string returnUrl = "/")
    {
        return View();
    }
```

It looks like a good idea.  After all, why would you ever want to publish code and not have the login be secure?

Here are two situations I ran into:

## Situation 1: When developing locally and the Build Configuration gets switched into a release mode.
 ### Cause
 This can happen on occassion, especially if your team does not have a build server in place and the agreed upon release method is Web Deploy from a developer's machine (whoever decides to take responsibility for the pushed code).
 
 ### Effect
 When your developer tries to login to the site locally, they get redirected to `https://localhost/YourSite` and development stops immediately. 
 
 ### Reaction
 At best your developer knows what's up and can switch their build configuration right away.
 
 At worst your developer has no clue what's going on and thinks maybe they broke something.  This is especially since the security module was just overhauled.  Or maybe they just updated the nuget packages.  Or maybe they just upgraded Visual Studio or installed a new extension.  Still, it used to work, so she searches the internet for terms like `localhost redirects to https` or `avoid redirect to https owin cookie authentication`.  
 
 The first leads to results about how modern browsers such as Chrome or Firefox will try to be smart and force you to browse with https.  This is further enforced by a smart developer who decides to watch their network traffic and sees something like `Upgrade-Insecure-Requests:1` in the HTTP Request Headers.  Now the problem is chrome?  No.
 
 Eventually they decide to reload Visual Studio, switch branches, or revert all the current changes to their local code -- anything that might fix it.  And usually through the mysteries of Visual Studio this works, though the root cause is still unknown.   

## Situation 2: When running integration tests on a build server or other dedicated test machine.

### Cause
  After you have your build server and integration tests, you want to put them into an automated build environment, have the code magically deployed and integration tests ran.  All this followed by pretty emails telling you exactly what failed and who did it.
  
  You want to test the release version of your code -- not the debug version. 
  
  Let's say you decide that `localhost` will suffice.  Since you have other things running on port `80`, you choose another one, say `8080`.
  
### Effect  
  Now you run your tests -- this is what happens:
  
  ![Login Redirection]({{ site.url }}/images/2016/login redirects.png)
  
  Let's examine this for a second.
  
  **Request 1: Go the page -- OWIN Cookie Authentication**
  
   `Found 302.  Location: http://localhost:8080/User/Login?RedirectUrl=`
  
  **Request 2: You get redirected to https.**
  
   `Found 302.  Location https://localhost/User/Login?RedirectUrl=`
  
  **Request 3: Attempt to view the  secure login page over SSL**
  
   `Not Found 404`
  
### Reaction
  No good.  But this is a controlled environment.  So you check IIS and the SSL Settings.  You look through all the `web.config` transformations for any URL Rewrites.  You check IIS for URL Rewrites.  Of course, everything looks good.
  
  You repeat the same web searches from above and, of course, you don't find anything of value.  Additionally, the searches in general tend to be somewhat useless because most people are trying to do the opposite with web queries like `how can I redirect http to https in (OWIN / IIS / UrlRewrite / MVC / Apache / etc)`.  No dice.
  
# Futher Complications

In this scenario, the root cause was not something that could even be stepped through in debugging.  And, of course, it never happened when debugging to begin with.  Go figure. 

All total, the inclussion of this as a directive cost the company around $2,000 in lost development time, if not more.

# The Solution

Don't use pre-processor directives or any other build hacks that change the nature of the compiled code. (Compiler optomizations not withstanding)

Just don't.  Please.

## Some alternatives to preprocessor directives

### AppSettings 
Set a value and have your code consume that at runtime.


``` xml
<appSettings>
    <add name="RequireHttpsForLogin" value="true" />
</appSettings>
```


### Validation checks
If you want certain things to happen only in Production, include an Environment flag in your settings, then check that and respond appropriately.  

``` csharp
var environment = (string)Reader.GetValue("Environment", typeof(string))
if(enviroment != Environments.Local){
    // do something
}
```

### Rethink your design
What are you really trying to accomplish?  Who's job is that?

In this situation, we considered creating a new FilterAttribute that would check the environment and redirect only in certain situations, but ultimately decided that SSL vs no SSL should be handled at the server and not inside the application since the entire application needs to be secure in QA and Production environments.  

  
  
  
   
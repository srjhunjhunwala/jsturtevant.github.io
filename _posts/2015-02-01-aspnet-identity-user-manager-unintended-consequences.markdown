---
layout: post
title: ASP.NET Identity User Manager and Unintended Consequences
date: "2015-02-01 09:00"
categories:
  - asp.net
  - csharp
---

Using the ASP.NET Identity ```UserManager``` to do some simple user claim manipulation I made the following function call:

{% highlight csharp %}
await UserManager.RemoveClaimAsync(userId, claimToDelete);
{% endhighlight %}

What do you think this piece of code would do?  Remove the claim that is specified?  It might.  It might not.  If you start to dig a little deeper you might be surprised...

## Reasonable Assumptions
I read a great [article by Scott Wlaschin](http://fsharpforfunandprofit.com/posts/is-your-language-unreasonable/?imm_mid=0cbe3f&cmp=em-prog-na-na-newsltr_20150131) about language design choices which make the language easy to reason about.  He leaves us with a simple definition:

> "reasoning about the code" means that you can draw conclusions using only the information that you have right in front of you, rather than having to delve into other parts of the codebase.  

If we look at the ```UserManager.RemoveClaimAsync()``` call above, we might *reason* that the code should simply remove the claim. In most cases it does but not always and leaves us wondering we couldn't reason about such a simple piece of code and delving deeper into the codebase.

## The problem
Everything was working fine, until one day I ran the ```UserManager.RemoveClaimAsync()``` function and found the claim was not being removed from the database.  

I was immediately confused by result and began looking everywhere for the error. Not finding any answers, I finally opened [SQL Server Profiler](https://msdn.microsoft.com/en-us/library/ms181091.aspx) to see the database calls that were being made.  I was in for a surprise.  

I found that the SQL ```DELETE``` statement for removing the claim was never being called but that the *one call to the ```UserManager``` was making* ***four database calls***.  

It was then that I realized the call to ```UserManager.RemoveClaimAsync()``` was actually failing and I was not checking the result that is returned.  I should have been doing the following:

{% highlight csharp %}
IdentityResult result = await UserManager.RemoveClaimAsync(userId, claimToDelete);
if (!result.Succeeded)
{
  foreach (var error in result.Errors)
  {
    // do something with the errors
    Trace.WriteLine(error);
  }
}
{% endhighlight %}

If I had handled the error correctly I would have immediately found the problem (someone updated a user to have the same email address as another user outside the application).  But what about the four database calls that resulted in the failed removal of the claim?

## Unintended Consequences
If you have the default MVC 5 template configuration you will find the following code in the IdentityConfig.cs file:

{% highlight csharp %}
public static ApplicationUserManager Create(IdentityFactoryOptions<ApplicationUserManager> options, IOwinContext context)
{
  var manager = new ApplicationUserManager(new UserStore<ApplicationUser>(context.Get<ApplicationDbContext>()));
  // Configure validation logic for usernames
  manager.UserValidator = new UserValidator<ApplicationUser>(manager)
  {
    AllowOnlyAlphanumericUserNames = false,
    RequireUniqueEmail = true
  };

  // more configuration...
}
{% endhighlight %}

The line ```RequireUniqueEmail = true``` looks innocuous but in reality is causing the error and failed claim deletion we uncovered above.

That single line tells the ```UserManager``` to validate the user on almost every call.  So for our simple call to the ```UserManager``` we are actually *validating* the user *then* removing the claim.  

*In the case of a failure*, the result is **4 database calls** and a failed removal of the claims.  *When successful*, the claim is removed but it results in **6 database calls including an** ```UPDATE``` statement to the ```AspNetUsers``` table.  Why an update statement to the ```AspNetUsers```table when removing a claim?  Your guess is as good as mine.

You can see all of the database calls by either by using [SQL Server Profiler](https://msdn.microsoft.com/en-us/library/ff650699.aspx) or adding a trace statement to your ```ApplicationDbContext```. Be forewarned you might not like what you find:

{% highlight csharp %}
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
  public ApplicationDbContext()
  : base("DefaultConnection", throwIfV1Schema: false)
  {
    // Log all the database calls when in Debug.
    this.Database.Log = (message) => Debug.Write(message);
  }
}
{% endhighlight %}

## Conclusion
There is a lot more happening behind the scene with the ```UserManger``` than can be *reasoned* about at first glance.  When making a simple call to delete a claim we find that there is user validation and user update calls being made.

The immediate lesson I learned was that I need to check the return result from all the calls of the ```UserManager```.  It adds a lot of noise to the code but can be factored out into a helper class.  

This experience has started to make me ponder if a language should help a programmer by make sure they use the return result from a function.  I will be paying attention to any times where I might have over looked the return result of a function.  Any thoughts?

I must also call to question the efficiency of the ```UserManager```.  There is a lot of chatter happening with the database to do even the simplest database calls.  For now, I think it is providing enough value by making it easy to generate email code,  do user authentication, and generate second factor codes that I will have to deal with it.  

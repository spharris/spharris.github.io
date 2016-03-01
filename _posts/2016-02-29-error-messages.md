---
layout: post
title: Your API Needs Good Error Messages
---
Not so long ago, any kind of invalid field input -- a `String` instead of an `Integer`, for example -- sent into the [Genability API](http://developer.genability.com/documentation/) would result in the following error message:

{% highlight html %}
<html>
    <head>
        <title>Apache Tomcat/7.0.52 (Ubuntu) - Error report</title>
    </head>
    <body>
        <h1>HTTP Status 400 - </h1>
        <HR size="1" noshade="noshade">
        <p>
            <b>type</b> Status report
        </p>
        <p>
            <b>message</b>
            <u></u>
        </p>
        <p>
            <b>description</b>
            <u>The request sent by the client was syntactically incorrect.</u>
        </p>
        <HR size="1" noshade="noshade">
        <h3>Apache Tomcat/7.0.52 (Ubuntu)</h3>
    </body>
</html>
{% endhighlight %}

For those that don't recognize it, this is Apache Tomcat's default 400 page. It certainly tells you what happened ("The request sent by the client was syntactically incorrect."), but it doesn't give you any clue about the root of the problem or how to solve it. This is a terrible error message for an API.

As you might imagine, this caused a lot of confusion on the part of our users. At least once or twice every week, we would get a support email from someone who used incorrect capitalization on `fromDateTime` or tried to send in an `Account` status of `INCTIVE` instead of `INACTIVE`.

To make matters worse, these kinds of errors were often tricky to diagnose. The only way to see which field was incorrect was to get people to send in the requests that they were making. Then, once they would finally send in their request, seeing the difference between `INCTIVE` and `INACTIVE` could take longer than anyone would care to admit.

A better error message in this situation could have solved both problems: users would have been able to understand what was happening and help themselves and, if they weren't able to solve the issue on their own, we would have known exactly where to start looking.

## Error Messages are Critical for API Usability
Everybody knows that good documentation is part of building a usable API. Our [API documentation](http://developer.genability.com/documentation/), I think, falls into that category. However, documentation can't possibly cover every single situation that a customer might find themselves in. What if a user tries to send in `account_id` (incorrect) instead of `accountId` (correct)? Unless they're going over the documentation with a magnifying glass, they almost certainly won't see their mistake. And with the crappy error message above, they're stuck. This is where better error messages come in.

In a lot of ways, error messages are just another form of documentation: they tell users how they should use your API. Sometimes,though, they can be even better than documentation. Error messages provide immediate feedback about what the user did wrong and how they can fix it. This immediate feedback leads to shorter integration times and less customer support.

### Time = Money
It's important to understand that the key non-functional measurement of a successful API is the speed at which a developer can go from zero to a working implementation. This is true in a couple of ways.

First and foremost, this means that your customers (and, subsequently, you) get to revenue more quickly. If your error messages help point users in the right direction instead of giving them bad or no information, they'll be able to correct their mistakes and get their software working in a more timely fashion.

Secondly, faster integration times mean fewer chances for people to say, "This is too hard (or expensive). Let's just stick with what we were doing before." This is especially important in cases where you are taking customers from your competitors. As we all know, people have very strong status quo biases; they will naturally look for any reason to avoid changing something that's already "good enough." Long, difficult integration periods caused (in part) by unhelpful or confusing are exactly the kinds of things that lead to lost customers.

Better error messages also mean less spent supporting customers and more time improving your API. When we changed our invalid input message to be more informative, we got to reallocate the time spent on investigating and answering simple support tickets to getting actual work done. It was an improvement all around.

## Creating Better Error Messages
Now we know that error messages are an important part of any good API design. So, how do you make yours better? Two things are key to any kind of error message:

1. Show the user what went wrong.
2. Show the user how to fix their mistake.

### Show the User What Went Wrong
If you try to send in an `account_id` property to the Genability API these days, here's what you'll get back:

{% highlight json %}
{
  "status": "error",
  "count": 1,
  "type": "Error",
  "results": [
    {
      "code": "InvalidError",
      "message": "The specified property is not valid for this type.",
      "objectName": "Profile",
      "propertyName": "account_id"
    }
  ]
}
{% endhighlight %}

This is much better. This error message tells users exactly what happened: they sent in a `Profile` property (`account_id`) that's invalid for that object. Now, they actually have a shot at figuring out how to get rid of this message. They can visit the [Profile](http://developer.genability.com/documentation/api-reference/account-api/usage-profile/#usage-profile) documentation page, see that the correct value is `accountId`, and go on their merry way.

### Show the User How to Fix Their Mistake
Sometimes, even more information is necessary. Not every error is a simple type error; sometimes input parameters need to fall within a certain range or need to have a value within some distance of another input value. In these cases, we not only want to tell the user what went wrong, but also what they can do to fix their mistake.

A simple example of this kind of thing is date ranges. Lots of things can happen over a period of time. Maybe you want to see all of your tweets for the last month or, in the Genability API, maybe you want to calculate your electricity cost for the last year. These kinds of requests generally have a `from` and a `to`, and the `to` obviously needs to come after the `from`. What if the user switches them?

One option would be to just say what went wrong:

{% highlight json %}
{
  "status": "error",
  "count": 1,
  "type": "Error",
  "results": [
    {
      "code": "InvalidArgument",
      "message": "Please enter a valid date range.",
      "objectName": "GetTariffCostRestRequest",
      "propertyName": "toDateTime"
    }
  ]
}
{% endhighlight %}

That's fine, and it's certainly better than our original Tomcat error page. With this error message, however, you're forcing the user to go back and examine their request to see what went wrong. If their code isn't well organized, or they just swapped their positions of `from` and `to` vertically when building their request, this could take a while for them to diagnose. We can do better.

Instead of just telling them what went wrong, show them:

{% highlight json %}
{
  "status": "error",
  "count": 1,
  "type": "Error",
  "results": [
    {
      "code": "InvalidArgument",
      "message": "The fromDateTime must be before the toDateTime. Your fromDateTime (\"2016-01-01\") is later than your toDateTime (\"2015-01-01\").",
      "objectName": "GetTariffCostRestRequest",
      "propertyName": "toDateTime"
    }
  ]
}
{% endhighlight %}

Now we're talking. The user is able to see not only what went wrong (an invalid date range, with the invalid dates included), but also exactly what they can do to fix it. They can easily see that they've swapped their variables around and that they can just switch them back to have a valid request.

## Conclusion
Error messages are a critical component of any API design. Not only do they make your users' lives easier, but they can lead directly to quicker revenue generation and more of your time spent working on the things you actually care about. Remember that next time you're tempted to just write, "Something went wrong, sorry!" in response to an error.

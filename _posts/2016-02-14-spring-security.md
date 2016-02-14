---
layout: post
title: TIL about Spring Security
---
Today, I was implementing [JWT](https://jwt.io) authentication for our REST API. Since the API is a Spring MVC application, a big part of implementing a new authentication methodology was getting up-to-speed on [Spring Security](http://projects.spring.io/spring-security/). The [documentation](http://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/) is excellent and was very helpful in that regard, but I thought it would be useful to explain a few of the concepts in my own words.

## Authentication Flow
Assuming that a user has already entered their credentials, the normal flow of Spring authentication and authorization looks something like this:

1. Create an `AuthenticationToken`.
2. Have an `AuthenticationProvider` decide whether that token is valid or not.
3. If the token is valid, populate the `SecurityContext` with the details of your authenticated user.
4. A `SecurityInterceptor` inspects the populated `SecurityContext` and, based on the permissions that have been granted to the user, decides whether the request can access the protected resource.

Let's take a look at each item in a little bit more detail.

### Create an `AuthenticationToken`
One of the items in the servlet filter chain is an authentication processing filter. This filter starts off the authentication process. It recognizes that the user is trying to authenticate and creates an [authentication token](http://docs.spring.io/autorepo/docs/spring-security/4.0.3.RELEASE/apidocs/org/springframework/security/authentication/AbstractAuthenticationToken.html). The type of token depends on the kind of authentication method that you're trying to implement. For JWT authentication, I ended up calling it a `JwtAuthenticationToken`.

If you are implementing [HTTP Basic](https://en.wikipedia.org/wiki/Basic_access_authentication) authentication, you might use a [UsernamePasswordAuthenticationToken](http://docs.spring.io/autorepo/docs/spring-security/4.0.3.RELEASE/apidocs/org/springframework/security/authentication/UsernamePasswordAuthenticationToken.html). Spring has a bunch of different options built in, so you probably won't have to make one yourself. The key idea of this step is that the token will be different depending on the kind of authentication mechanism(s) that you want to implement.

### Have an `AuthenticationProvider` Validate the Token
Once it has created an auth token, the authentication processing filter passes the token off to the [AuthenticationManager](http://docs.spring.io/autorepo/docs/spring-security/4.0.3.RELEASE/apidocs/org/springframework/security/authentication/AuthenticationManager.html). It's the authentication manager that decides how you're actually going to authenticate. It does this by checking the token against its list of [AuthenticationProvider](http://docs.spring.io/autorepo/docs/spring-security/4.0.3.RELEASE/apidocs/org/springframework/security/authentication/AuthenticationProvider.html)s and seeing which one will accept the kind of token that your filter created. If you have multiple kinds of authentication, you will have multiple kinds of tokens as well as multiple `AuthenticationProvider` implementations, one for each kind of token.

The most important aspect of `AuthenticationProvider` implementations is that the `authenticate` method can only have three outcomes:

1. The `Provider` doesn't understand this type of token. In this case, the provider should return `null`. This is a kind of "non-judgemental" authorizaton failure. It indicates to the `AuthencationManager` that it should try and find another `AuthenticationProvider` that understands this token.
2. The token is invalid. This could happen if, for example, the password for the given user is incorrect. In this case, the provider should throw some type of [AuthenticationException](http://docs.spring.io/autorepo/docs/spring-security/4.0.3.RELEASE/apidocs/org/springframework/security/core/AuthenticationException.html).
3. The token is valid. The `Provider` returns a populated `Authentication` object.

Note that if the provider does not understand a particular type of token -- a `JwtAuthenticationToken` is sent to a `UsernamePasswordAuthenticationProvider`, for example -- the authentication attempt is **not** rejected. The end result of such a request would be a `SecurityContext` that is not populated.

### Populate the Security Context
It's the `AuthenticationProvider` that actually decides whether the user's credentials are valid or not. If they are, it returns an [Authentication](http://docs.spring.io/spring-security/site/docs/4.0.1.RELEASE/apidocs/org/springframework/security/core/Authentication.html) object populated with the details about the authenticated user (the "`Principal`" in Spring). The `Authentication` object includes simple things like the username or email, but also details about which kinds of resources the principal is allowed to access. If the credentials are not valid, the `AuthenticationProvider` throws an exception. Assuming that the credentials are valid, the [SecurityContext](http://docs.spring.io/spring-security/site/docs/4.0.1.RELEASE/apidocs/org/springframework/security/core/context/SecurityContext.html) (which holds all of the security-related information about the current request) is populated with the `Authentication` information.

### Authorize the Request
Now that the `SecurityContext` is populated, the final step in the Spring Security authentication and authorization flow is to decide whether the user is actually authorized to access the protected resource. This is the job of the [SecurityInterceptor](http://docs.spring.io/spring-security/site/docs/3.2.0.RELEASE/apidocs/org/springframework/security/access/intercept/AbstractSecurityInterceptor.html), which will likely be one of the final filters in your servlet filter chain. The `SecurityInterceptor` is configured with information about what authorities are required to access each protected resource. When it is invoked, it will inspect the roles present on the `Principal` populated in the `SecurityContext`. If it has the required role, access to the resource is granted. Otherwise, the request is rejected.

## Access for Anonymous Users
What if you have resources that you don't want to require people to log in for? Access for anonymous users can be granted through a two-step process. First, you include a special filter, called the [AnonymousAuthenticationFilter](http://docs.spring.io/autorepo/docs/spring-security/4.0.3.RELEASE/apidocs/org/springframework/security/web/authentication/AnonymousAuthenticationFilter.html), in your filter chain. This filter, which you put after your standard authentication handling filter (the first step described [above](#create-an-authenticationtoken)), populates the `SecurityContext` with a special `AuthenticationToken`, an `AnonymousAuthenticationToken` if no previous step in the filter chain has populated it. The anonymous token has two properties: a `principal` of `anonymousUser` and a single role of `ROLE_ANONYMOUS`.

During the authorization phase of the request, when it is run through the `SecurityInterceptor` filter, `ROLE_ANONYMOUS` can be checked against the required roles for a particular resource.

## Summary
Those are the very basics of request authentication using Spring Security: create a token, authorize the token, populate the `SecurityContext`, and authorize the request.

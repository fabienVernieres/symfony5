# Custom Authenticator authenticate() Method

We're currently converting our old Guard authenticator into the new authenticator
system. And... these two systems share a lot of methods, like `supports()`,
`onAuthenticationSuccess()` and `onAuthenticationFailure()`.

The *big* difference is down in the new `authenticate()` method. In the old Guard
system, we split up authentication into a few methods. We had `getCredentials()`,
where we grab some information, `getUser()`, where we found the `User` object, and
`checkCredentials()`, where we checked the password. All three of these things are
now combined into the `authenticate()` method... with a few nice bonuses. For example,
as you'll see in a second, it's no longer *our* responsibility to check the password.
That now happens *automatically*.

## The Passport Object

Our job in `authenticate()` is simple: to return a `Passport`. Go ahead and add a
`Passport` return type. That's actually needed in Symfony 6. It wasn't added
automatically due to a deprecation layer and the fact that the return type changed
from `PassportInterface` to `Passport` in Symfony 5.4.

*Anyways*, this method returns a `Passport` here... so do it: `return new Passport()`.
By the way, if you're new to the new custom authenticator system and want to learn
more, check out our [Symfony 5 Security tutorial](https://symfonycasts.com/screencast/symfony5-security)
where we talk all about this. I'll go through the basics now, but the details live
over there.

Before we fill in the `Passport`, grab all the info from the `Request` that we
need... paste... then set each of these as variables:
`$email =`, `$password =`... and let's worry about the CSRF token later.

The first argument to the `Passport()` is a `new UserBadge()`. What you pass here
is the *user identifier*. In our system, we're logging in via the email, so pass
`$email`!

And... if you want, you can stop right here. If you pass only one argument to
`UserBadge`, Symfony will use your "user provider" from `security.yaml` to
find that user. We're using an `entity` provider, which tells Symfony to try to
query for the `User` object in the database via the `email` property.

## Optional Custom User Query

In the old system, we did this all manually by querying the `UserRepository`.
That is *not* needed anymore... but sometimes... if you have custom logic, you will
*still* want to find the user manually.

*If* you have this use-case, pass a `function()` as a second argument that accepts
a `$userIdentifier` argument. With this setup, when the authentication system
needs the User object, it will call our function and pass us the "user identifier"...
which will be whatever we passed to the first argument. So, the email.

Our job is to use that to return the user. Start with
`$user = $this->entityManager->getRepository(User::class)`

And yea, I could have injected the `UserRepository` instead of the entity manager.
That would be better... but this is fine. Then `->findOneBy(['email' => $userIdentifier])`.

If we did *not* find a user, we need to `throw` a `new UserNotFoundException()`.
*Then*, `return $user`.

First `Passport` argument is done!

## PasswordCredentials

For the second argument, down here, change my bad semicolon to a comma - then say
`new PasswordCredentials()` and pass this the submitted `$password`.

That's all we need to do here. Yup, we do *not* need to actually *check* the password.
We pass a `PasswordCredentials()`... and then there's another system that's
responsible for checking the submitted password against the hashed password in
the database. How cool is that?

## Extra Badges

Finally, the `Passport` accepts an array of "badges", which are extra "stuff"
that you want to add... usually to activate other features.

We only need to pass *one*: a `new CsrfTokenBadge()`. This is because our login
form is protected by a CSRF token. Previously, we checked it manually. Lame!

But we don't need to do that anymore. Pass the string `authenticate` to the first
argument... which just needs to match the string used when we generate the token
in the template: `login.html.twig`. If I search for `csrf_token`... there it is!
use the same string we use here.

For the second argument, pass the submitted CSRF token:
`$request->request->get('_csrf_token')`, which you can also see in the login form.

And... done! *Just* by passing the badge, the CSRF token will be validated.

Oh, and while we don't *need* to do this, I'm also going to pass a
`new RememberMeBadge()`. If you use the "Remember Me" system, then you and need to
pass this badge. It tells the system that you opt "into" having a remember me token
set. You *still* need to have a "Remember Me" checkbox here... for it to work.
Or, to *always* enable it, add `->enable()` on the badge.

And, of course, none of this will work unless you activate the `remember_me`
system under your firewall, which I don't actually have yet. It's still safe
to add that badge... but there won't be any system to process it and add the
cookie.

## Deleting Old Methods!

Anyways, we're done! If that felt overwhelming, back up and watch our Symfony
Security tutorial to get more context.

The cool thing is that we don't need `getCredentials()`, `getUser()`,
`checkCredentials()`, *or* `getPassword()` anymore. All we need is
`authenticate()`, `onAuthenticationSuccess()`, `onAuthenticationFailure()`, and
`getLoginUrl()`. We can even celebrate up here by removing a *bunch* of old use
statements. Yay!

Oh, and look at the constructor here. We no longer need `CsrfTokenManagerInterface`
or `UserPasswordHasherInterface`: both of those checks are done somewhere *else*
no *for* us. And that gives us two *more* use statements to delete.

## Activating the New Security System

At this point, our one custom authenticator *has* been moved to the new authenticator
system. This mean that, in `security.yaml`, we are ready to switch to the new system!
Say `enable_authenticator_manager: true`.

And these custom authenticators aren't under a `guard` key anymore. Instead,
add `custom_authenticator` and add this directly below that.

Okay, moment of truth! We just *completely* switched to the new system. Will
it work? Head back to the homepage, reload and... it does! And check out those
deprecations! It went from around 45 to 4. Woh!

And *some* of these relate to one *more* security changes. Next: let's update
to the new `password_hasher` & check out a new command for debugging security
firewalls.
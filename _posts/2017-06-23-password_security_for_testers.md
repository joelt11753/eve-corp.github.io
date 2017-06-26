---
layout: page
title: Password Security for Testers
keywords: passwords, application security, white-box testers
---

This article is targeted towards white-box testers or SDETS, but can be used by anybody as an introduction to current best-practices for storing passwords.

Below are basic guidelines I follow to develop a secure login portal, whereby our application (and not a third-party, such as Google or Facebook) is responsible for storing usernames and passwords.

## Use SSL
The first is SSL, or Secure Socket Layer.  This is the 'S' on 'HTTPS'.  This one is pretty straight forward -- encrypt data while it's in-motion (from the client to the server) so that a man-in-the-middle can't just grab your login information.

## Do not re-populate password fields
If something is wrong with the login information, you should not return the password to the client to re-populate the login form.  The downside is that the user has to re-enter their password, which is kind of annoying.  "But it's going over SSL, so what's the problem?", you say.

<img src="/images/password-postback.png" alt="password as a postback" />

See the problem?  Because we've provided the value to the page, even though it's blanked out when it renders, it's still plain-text in the code-behind.  This means that if you get up and leave, somebody with access could use this to view your password.

## Use the 'password' type for input fields

As in the example above, HTML 5 introduced these great input types which automatically take care of masking passwords for you.  Some browsers have even added some extra security around these fields to warn the user if your password is about to be transported over an open (non-SSL) network.  Additionally, some browsers have also implemented features to let the user temporarily view what they've typed to help check for errors.

## Never ever store passwords in plain text

It's that simple. Don't store un-encrypted passwords ever.  You don't want some shmuck with access to the database to have access to your password. There are ways around any reason why you would possibly want to do this.  

## Never ever store passwords in logs

Same as above: don't do it.  Note that even using replacing characters with astricks in log files is weak because it reveals one component of the password: its length.  Use a standard placeholder, such as "[password]" instead if you must have something.

## Don't encrypt, hash!

While from the outside, encryption and hashing look similar -- they both result in giberish -- they are quite different with the key point being that *encryption is reversible*.  You don't want that.

Now, there are some exceptions to this, where you really do need to encrypt rather than hash.  For example, if you are building an automation tool and you *must* to store the password to log in to an alternate system, then you need encryption.  All the same rules otherwise apply.  (Note that this should still be avoided where possible by use of Active Directory, API Tokens, etc.)

For your standard application, you never need to retrieve that password, so it gets hashed before going into the database.

## Use a strong hashing algorithm

Do an internet search for a strong hashing algorithm *by today's standards* and use one of them.  Because computers get stronger, previously secure algorithms are no longer secure, and for that reason alone I am not making any recommendations as to what is "good".  If you have access to the code, find the place where password's are validated, and check to see that it's using a library of *strong hashing algorithm*.

## Hash passwords with a unique salt

A salt is something that's added to a user's password to prevent the use of rainbow tables. Rainbow tables are pre-computed lists of passwords and their hashes, so a hacker just needs to match hash with hash, and there's your password.  The use of a salt makes this significantly harder if not computationally prohibitive.

It's normal to store the hash and salt in the same record in the database.

## There should be no restrictions on characters

Anything in a password should be valid, including periods, spaces, and unicode puppy dog faces &#x1f436;

## There should be no restrictions on length

Because the password's going to be hashed, it'll be reduced down to a managable length to fit into the password field in the database.

## There should be a minimum length

This is to protect your users and make it harder for somebody to randomly guess.  Again, as computers get more powerful, the "acceptable" minimum length changes.  I would say 12 is a good minimum.

## Use a smart password-strength library

Something like [zxcvbn](https://github.com/dropbox/zxcvbn) will be a much better indicator of password strength for your users than anything you could come up with.  As in... while requiring, for example, 2 lower case, 2 upper case, 2 numbers, and 2 punctuation -- there are much better passwords that are far easier for a human to remember.  And the easier it is for a human to remember, the less-likely that it will be written down.  (See the link to the zxcvbn demo for more insight.)

## Admins should not be allowed to set user passwords

If an admin is setting a password, that means that too many people know the password already.  If an admin needs to *reset* a password on behalf of the user, they should just click a button, and the user should receive an email with a reset-code to reset their own password.

## Passwords shouldn't be sent through email

This is generally well-known, but still broken a lot.  Even though lots of work has been done across the internet to help us be more secure, in most cases, email is not considered a secure medium.

## Password reset links should expire

Because email is not secure, a password reset link should expire.  Even if your application does use temporary passwords sent over email, they should still expire for the same reason.  I find 15 minutes to be a fair timeline, especially since sometimes email can be slow, and people get distracted, etc.

## Users should be prompted for "something they know" before being provided with a reset code

Typically called "security questions", the "something they know" is in leiu of a password, and the answers to those questions should be treated and stored with as much care as their normal password (ie. salted & hashed).

You might think "oh what if I want to show them their answers.  Maybe it should be encrypted rather than hashed!"  Just don't.   If they don't know their answers, they can enter in a new one (as long as they provide their password at the same time).  You don't want somebody walking up to their machine to be able to view their security question answers.


**What else is on your mind?**

## Further Reading:

* [Play with zxcvbn](https://dl.dropboxusercontent.com/u/209/zxcvbn/test/index.html)
* [Safely Storing User Passwords: Hashing vs. Encrypting](http://www.darkreading.com/safely-storing-user-passwords-hashing-vs-encrypting/a/d-id/1269374)


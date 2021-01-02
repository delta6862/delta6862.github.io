![thumbnail]

## Overview
 - [Introduction](#introduction)
 - [The bug](#the-bug)
 - [Discovery](#discovery)
 - [Reporting](#reporting)
 - [Exploitation and impact](#exploitation-and-impact)
 - [Closing notes](#closing-notes)

## Introduction
This post is about a bug found in Magister which allowed a malicious actor to store arbitrary javascript code on the site. 
This issue has since been patched and been made free to publicly disclose.
If you're unfamiliar with Magister as a platform I've briefly explained it in a different blog post which can be 
found [here](https://delta6862.github.io/library/Bug_Bounties/Magister/Password_reset/#magister).

## The bug
Magister has a feature called "ELO bronnen" where users can store files, folders, and most crucially for this bug; URLs. 
The URL must start with `http(s)` to prevent users from using this feature for unintended purposes. 

However this policy was enforced client side, 
thus making it possible for any user to bypass the filter entirely by sidestepping the front-end and talking directly to the server.
Without the filter in place, a user can specify the protocol to be `javascript:` which then executes the rest of the URL as javascript.

## Discovery
I was initially looking at this type of vulnerability in magister's internal mailing system. While this did show some interesting behavior it
seemed to be set up properly so I eventually went back to work. 

A little while later I had to access ELO bronnen for its intended purpose and
noticed it also had a feature to store URLs. Initially, I assumed that, like the mailing system, it was set up securely but took a look at it anyway.
However, after a bit of poking around, I found this bug. 

At first, I considered this to be a bit of a joke vulnerability, 
I was the only one who could access it and, because Magister opens the link in a new tab,
I also presumed the javascript would be executed in a blank context.
However, when testing it for completeness sake I noticed that the new tab kept its parent context 
until it navigated to a new location. 
This means the injected javascript gets executed within the context of Magister allowing it to access sensitive information and perform user actions.

## Reporting
After finding this I opened a new responsible disclosure report on [Zerocopter](https://www.zerocopter.com/). The images below includes both the original
report and the discussion that followed. 
It's worth pointing out this report was handled a lot faster than my previous report. I'd attribute this to both
the vulnerability being more well defined than the previous one and possibly both Magister and Zerocopter trusting the issue to be legitimate based
off previous experience with my reporting.

![comment-1]
![comment-2]
![comment-3]
![comment-4]

## Exploitation and impact
As with the "account take over via improper password reset implementation" bug, the goal is to establish persistent access to all accounts on a given subdomain.
There were a few hurdles with this:
- From an unprivileged users perspective, this is effectively self XSS as only the original user can access their own file storage and an unprivileged user can't create a link on the shared file system.
- The session cookies are HTTPOnly, which means we can't steal the session cookies and send them to a collection server.
- Magister uses JSON web tokens for API authentication, which have a set lifetime.

The first two proved to be the biggest challenge. The web token is stored in the session storage which we can access freely with our XSS.
However its lifetime is around 30 minutes to an hour, this gives us near-full access for a limited time window. Session cookies, on the other
hand, are valid for as long as the user is active. This means we can keep a session cookie alive for as long as we want simply by making regular
requests with it. In short; we need to turn our JSON web token into a session cookie to achieve persistence. 

To do this we can once again take advantage of the password reset functionality. Using the web token we can get the current email address of the user,
 send it to our collection server, and then change it to an email address we control. Once this is done we request a password reset on the specified account and retrieve the reset code from our inbox. Using this reset code we change the password, log in and get a valid session cookie for that account. We then change the email back to the original email address and keep our session alive. The victim will be unable to log in to their account and, thinking they forgot their password, reset it and be none the wiser. In the meantime, we have our valid session cookie which we can use to request a web token for that account whenever we please.

Finally, we need to turn this self XSS into an actual XSS, this turned out to be easier than expected. Certain privileged users may access specific
parts of other users private file storage. An unprivileged user could create a link containing javascript code that achieves persistence and then redirects
to a valid site. This user could then ask a privileged user to visit that link under false pretenses. With the new access, this unprivileged user could
use the higher privileged users accout to create a similar link under false pretenses in the shared file space, making it accessible to every user
on the subdomain. Each user simply has to click it once for their account to be permanently compromised. On a high traffic item, such as an exam schedule,
this could be done in a matter of weeks.

Now for the fun part, "what to do with it", I had several fun conversations with friends about what we would do with full access to everyone's account.
After some simpler ideas such as organizing a mass secret Santa under the name of the principle one friend came up with the following scenario;

At the start of the school year, we use our access to create a completely fictitious individual and add them to the final year of high school. This
person will obviously never show up, never hand in work, and never make an exam. However, as Magister handles both the grading and the absences of a student,
We can simply change this persons grades and make it appear as if they're attending, effectively creating a 'ghost' in the class who will eventually pass
their final year of high school and have a short, but bright, future ahead of them.

## Closing notes
- After the vulnerability was fixed I reached out to Magister asking if they wished to play an active role in the reporting process. 
Initially, the idea was to have a phone call regarding the subject, however, this was later scrapped in favor of email.
After a short correspondence, their information security officer asked me to include the following statement in the blog post.

```
Reaction Iddink Group:

"As a provider of educational information systems we understand the importance of protecting the privacy and security of our, often vulnerable, users. Unfortunately, it is still possible that some vulnerabilities remain despite our best efforts.

If you discover a vulnerability, we encourage you to adhere to ‘responsible disclosure’ and inform us before releasing any information of the vulnerability to the public. We are committed to working together with our community to protect our users and we will address any reported vulnerability as quickly as possible.

Within Iddink Group we are pleased with how this vulnerability was reported and that we were able to improve the security of our systems before the outside world was informed."
```
- I urge both blue and red team to check each endpoint individually. Just because a certain feature was implemented correctly on one endpoint doesn't mean this is the case everywhere on the
application. Legacy code can cause a lot of oversights if not properly maintained.
- My point of contact at Iddink group kindly pointed out that the final exploit mentioned in [Exploitation and impact](#exploitation-and-impact) is not possible with this vulnerability. Evidently users can only be added with Magister desktop, which requires 2fa. This means an attacker cannot achieve persistence with the password reset option.

[thumbnail]: thumbnail.png
[comment-1]: comment-page-1.png
[comment-2]: comment-page-2.png
[comment-3]: comment-page-3.png
[comment-4]: comment-page-4.png

---
tags:
  - AUTH0
source: https://medium.com/@bap_16778/alternatives-to-auth0-we-are-most-excited-about-in-2024-49696e984072
---




# Alternatives to Auth0 we are most excited about in 2024

![](https://miro.medium.com/v2/resize:fit:700/0*QMKr8q928q6hv8U6.gif) 
Hey friends 👋
Auth0 and its parent company, Okta, are what developers think of when managing user authentication and authorisation. Both do,  *in general* , a great job at implementing secure access for our favourite apps!
They are also the tools to help us work with the temperamental 0Auth2. 😅
![](https://miro.medium.com/v2/resize:fit:700/1*j9WdKU8Vq4UScdvYuTdE8g.png) 
Though these options deserve to be considered, many decent alternatives are on the market.
In the open-source world, we are particularly excited about the below 5 alternatives!
Let’s dive straight into it 👇


# 

Ory is a big name in the industry. It maintains advanced open-source security software solving authentication, authorisation, access control, application network security, and delegation. Ory Kratos is an open-source repository that is Auth0-like. It focuses on the identity, user management and authentication system for the Cloud.
![](https://miro.medium.com/v2/resize:fit:700/1*svSkQNaltzAjFrNwI7wQZw.png) 
 **The main features you can expect are:** 
- Registration, login, and account management flows for passkeys, biometric authentication, social sign-in, SSO, and multi-factor authentication
- Pre-built login, registration, and account management pages and components
- OAuth2 and OpenID Connect provider for single sign-on, API access, and machine-to-machine authorization
- Low-latency permission checks based on Google’s Zanzibar model and with built-in support for the Ory Permission Language



# Hanko

Hanko is an elegant project focused on an open-source authentication and user management solution. Their approach focuses on moving the login beyond passwords. Passkeys are the future, and this was well articulated in the blog post  [ *The Beginning of the End of the password*  ](https://blog.google/technology/safety-security/the-beginning-of-the-end-of-the-password/), an article written by Google.
Since passkeys were announced only very recently, the ecosystem of devices, browsers, and operating systems is getting ready to move beyond passwords.
![](https://miro.medium.com/v2/resize:fit:700/1*phq9UeJ3a-KiH7nnbXlrRg.png) 
 **Hanko has been preparing for this shift, and as of present, it provides:** 
- Fast integration with Hanko Elements web components (login box and user profile)
- API-first, small footprint, cloud-native
- Availability for self-hosting and on Hanko Cloud.

The easiest way to get started with hank is with docker-compose:
1️⃣ Clone this repository:  `git clone  [https://github.com/teamhanko/hanko.git](https://github.com/teamhanko/hanko.git) ` 
2️⃣ In the newly created hanko folder, run:\
 `docker compose -f deploy/docker-compose/quickstart.yaml -p "hanko-quickstart" up --build` 
3️⃣ After the services are running, the login page can be viewed at  `localhost:8888` . To receive emails without your own SMTP server, Hanko added mailslurper which will be available at  `localhost:8080` 


# Cerbos

Cerbos is an authorisation layer. It enables you to define context-aware access control rules for your application resources in YAML policies, which are managed and deployed via your Git-ops infrastructure. You can set up a self-hosted Cerbos Policy Decision Point.
 **With Cerbos you can:** 
- Define authorisation logic in a collaborative IDE and testing environment
- Collaborate with colleagues to author and share policies in private playgrounds
- Deploy with a fully hosted CI/CD pipeline
- Build special policy bundles for client-side or in-browser authorisation

Here is how Cerbos works with your application (a more advanced explanation can also be found  [here](https://www.cerbos.dev/how-it-works)  ) : 👇
![](https://miro.medium.com/v2/resize:fit:700/1*SNY-T66_NRk3icdkjz81pg.png) 
To try out Cerbos, you can get started with their very fun  [Cerbforce tutorial](https://docs.cerbos.dev/cerbos/latest/tutorial/00_intro.html) 


# Zitadel

Zitadel is an open-source user management tool quickly set up like Auth0. Zitadel is built with a complex multi-tenancy architecture in mind, and it provides solutions to handle B2B customers and partners.
 **Zitadel is built with the following structure in mind:** 
- API-first approach
- Multi-tenancy authentication and access management
- Strong audit trail due to event sourcing as storage pattern
- Actions to react on events with custom code
- Self-service for end-users, business customers, and administrators
- CockroachDB or a Postgres database as storage option

![](https://miro.medium.com/v2/resize:fit:700/1*LhxXIjhCyDm1DVQjvnaSXA.png) 
You should consider Zitadel if you are interested in leveraging the below features:
If the information above makes you think Zitadel is a good fit for you, you can get started by checking out their guide  [here](https://zitadel.com/docs/guides/start/quickstart) 


# SuperToken

SuperTokens is another open-source authentication solution.
 **The main features you can find on the platform are the following:** 
- Various login types: Email / password, Passwordless (OTP or Magic link based), Social / OAuth 2.0
- Access control
- Session management
- User management
- Self-hosted / managed cloud

Their architecture is unique because your backend API layer would sit in the middle of your front end and SuperTokens’. This enables easy customisations to the auth logic and allows for a secure session solution.
![](https://miro.medium.com/v2/resize:fit:700/1*W6qqOvKZjliA2TsPy-z3GQ.png) 
To start with SuperTokens, you can use their practical guide to pick the login type you want and get started  [here](https://supertokens.com/docs/guides) 
That’s it for this one. ☝️
As you have learned in this article, there are many exciting alternatives to Auth0.
You should look into each alternative and determine what service best suits your current or future needs.
In the meantime, I invite you to consider supporting these projects by starring them.
 *(We are not affiliated with them. We just think that great projects deserve great recognition.)* 
See you next week,
Your Dev.to buddy 💚

If you want to join the self-proclaimed “coolest” server in open source 😝, you should join our [ discord server.](https://discord.com/invite/ChAuP3SC5H/?utm_source=medium&utm_campaign=alternative_to_auth0)  We are here to help you on your journey in open source. 🫶
 *Originally published at *  [ *https://dev.to*  ](https://dev.to/quine/alternatives-to-auth0-we-are-most-excited-about-in-2024-2b07) * on February 16, 2024.* 
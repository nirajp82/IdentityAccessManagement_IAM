An Illustrated Guide to OAuth and OpenID Connect
### Reference: https://developer.okta.com/blog/2019/10/21/illustrated-guide-to-oauth-and-oidc
<img width="1600" height="800" alt="image" src="https://github.com/user-attachments/assets/69e72685-6015-4592-aa9f-47be2462cbfd" />
In the “stone age” days of the Internet, sharing information between services was easy. You simply gave your username and password for one service to another so they could login to your account and grab whatever information they wanted!

<img width="1000" height="643" alt="image" src="https://github.com/user-attachments/assets/7070ad89-5952-40fa-9002-c2a65727ff47" />
Yikes! You should never be required to share your username and password, your credentials, to another service. There’s no guarantee that an organization will keep your credentials safe, or guarantee their service won’t access more of your personal information than necessary. It might sound crazy, but some applications still try to get away with this!

Today we have an agreed-upon standard to securely allow one service to access data from another. Unfortunately, these standards use a lot of jargon and terminology that make them more difficult to understand. The goal of this post is to explain how these standards work using simplified illustrations.

You can think of this post as the worst children’s book ever. You’re welcome.

<img width="800" height="456" alt="image" src="https://github.com/user-attachments/assets/c9649202-b878-487d-b391-0bcc2bf1ddad" />

By the way, this content is also available as a video!

https://www.youtube.com/watch?v=t18YB3xDfXI

Ladies and Gentlemen, Introducing OAuth 2.0
OAuth 2.0 is a security standard where you give one application permission to access your data in another application. The steps to grant permission, or consent, are often referred to as authorization or even delegated authorization. You authorize one application to access your data, or use features in another application on your behalf, without giving them your password. Sweet!

As an example, let’s say you’ve discovered a web site named “Terrible Pun of the Day” and create an account to have it send an awful pun joke as a text message every day to your phone. You love it so much, you want to share this site with everyone you’ve ever met online. Who wouldn’t want to read a bad pun every day, am I right?
<img width="800" height="671" alt="image" src="https://github.com/user-attachments/assets/8b2c7f19-dad6-4d90-8f93-bc93529673cd" />

However, writing an email to every person in your contacts list sounds like a lot of work. And, if you’re like me, you’ll go to great lengths to avoid anything that smells like work. Good thing “Terrible Pun of the Day” has a feature to invite your friends! You can grant “Terrible Pun of the Day” access to your email contacts and send out emails for you! OAuth for the win!

<img width="800" height="1120" alt="image" src="https://github.com/user-attachments/assets/e2423550-6e90-4f0b-a0ba-d806bfaefe9f" />

Pick your email provider
Redirect to your email provider and login if needed
Give “Terrible Pun of the Day” permission to access to your contacts
Redirect back to “Terrible Pun of the Day”
In case you change your mind, applications that use OAuth to grant access also provide a way to revoke access. Should you decide later you no longer want your contacts shared, you can go to your email provider and remove “Terrible Pun of the Day” as an authorized application.

## Let the OAuth Flow
You’ve just stepped through what is commonly referred to as an OAuth flow. The OAuth flow in this example is made of visible steps to grant consent, as well as some invisible steps where the two services agree on a secure way of exchanging information. The previous “Terrible Pun of the Day” example uses the most common OAuth 2.0 flow, known as the “authorization code” flow.

Before we dive into more details on what OAuth is doing, let’s map some of the OAuth terminologies.

You’ve just stepped through what is commonly referred to as an OAuth flow. The OAuth flow in this example is made of visible steps to grant consent, as well as some invisible steps where the two services agree on a secure way of exchanging information. The previous “Terrible Pun of the Day” example uses the most common OAuth 2.0 flow, known as the “authorization code” flow.

Before we dive into more details on what OAuth is doing, let’s map some of the OAuth terminologies.
<img width="1007" height="830" alt="image" src="https://github.com/user-attachments/assets/ef0c6ae2-01d3-4573-a1ea-0c182df4e3e4" />

<img width="998" height="770" alt="image" src="https://github.com/user-attachments/assets/37355500-0fb1-4dca-857e-ceede10517e3" />

Note: Sometimes the “Authorization Server” and the “Resource Server” are the same server. However, there are cases where they will not be the same server or even part of the same organization. For example, the “Authorization Server” might be a third-party service the “Resource Server” trusts.

Now that we have some of the OAuth 2.0 vocabulary handy, let’s revisit the example with a closer look at what’s going on throughout the OAuth flow.

<img width="800" height="1928" alt="image" src="https://github.com/user-attachments/assets/65bf10a6-9a8d-4932-9c4a-6ee7d5dfa060" />

You, the Resource Owner, want to allow “Terrible Pun of the Day,” the Client, to access your contacts so they can send invitations to all your friends.
The Client redirects your browser to the Authorization Server and includes with the request the Client ID, Redirect URI, Response Type, and one or more Scopes it needs.
The Authorization Server verifies who you are, and if necessary prompts for a login.
The Authorization Server presents you with a Consent form based on the Scopes requested by the Client. You grant (or deny) permission.
The Authorization Server redirects back to Client using the Redirect URI along with an Authorization Code.
The Client contacts the Authorization Server directly (does not use the Resource Owner’s browser) and securely sends its Client ID, Client Secret, and the Authorization Code.
The Authorization Server verifies the data and responds with an Access Token.
The Client can now use the Access Token to send requests to the Resource Server for your contacts.


### Client ID and Secret
Long before you gave “Terrible Pun of the Day” permission to access your contacts, the Client and the Authorization Server established a working relationship. The Authorization Server generated a Client ID and Client Secret, sometimes called the App ID and App Secret, and gave them to the Client to use for all future OAuth exchanges.

Client receives the Client ID and Client Secret from the Authorization Server
<img width="800" height="568" alt="image" src="https://github.com/user-attachments/assets/f4c25b72-31fe-4365-b103-155909a1681d" />

As the name implies, the Client Secret must be kept secret so that only the Client and Authorization Server know what it is. This is how the Authorization Server can verify the Client.

That’s Not All Folks… Please Welcome OpenID Connect
OAuth 2.0 is designed only for authorization, for granting access to data and features from one application to another. OpenID Connect (OIDC) is a thin layer that sits on top of OAuth 2.0 that adds login and profile information about the person who is logged in. Establishing a login session is often referred to as authentication, and information about the person logged in (i.e. the Resource Owner) is called identity. When an Authorization Server supports OIDC, it is sometimes called an identity provider, since it provides information about the Resource Owner back to the Client.

OpenID Connect enables scenarios where one login can be used across multiple applications, also known as single sign-on (SSO). For example, an application could support SSO with social networking services such as Facebook or Twitter so that users can choose to leverage a login they already have and are comfortable using.
<img width="800" height="840" alt="image" src="https://github.com/user-attachments/assets/d5de6f45-073e-44f9-80e8-e75ae31fb1ad" />

FleaMail SSO

The OpenID Connect flow looks the same as OAuth. The only differences are, in the initial request, a specific scope of openid is used, and in the final exchange the Client receives both an Access Token and an ID Token.

Terrible Pun of the Day OIDC Example

As with the OAuth flow, the OpenID Connect Access Token is a value the Client doesn’t understand. As far as the Client is concerned, the Access Token is just a string of gibberish to pass with any request to the Resource Server, and the Resource Server knows if the token is valid. The ID Token, however, is very different.

Jot This Down: An ID Token is a JWT
An ID Token is a specifically formatted string of characters known as a JSON Web Token, or JWT. JWTs are sometimes pronounced “jots.” A JWT may look like gibberish to you and me, but the Client can extract information embedded in the JWT such as your ID, name, when you logged in, the ID Token expiration, and if anything has tried to tamper with the JWT. The data inside the ID Token are called claims.

Terrible Pun of the Day Examines an ID Token

With OIDC, there’s also a standard way the Client can request additional identity information from the Authorization Server, such as their email address, using the Access Token.

Learn More About OAuth and OIDC
That’s OAuth and OIDC in a nutshell! Ready to dig deeper? Here are some additional resources to help you learn more about OAuth 2.0 and OpenID Connect!

What the Heck is OAuth?
Nobody Cares About OAuth or OpenID Connect
Implement the OAuth 2.0 Authorization Code with PKCE Flow
What is the OAuth 2.0 Grant Type?
OAuth 2.0 From the Command Line
Build a Secure Node.js App with SQL Server
As always, feel free to leave comments below. To make sure you stay up to date with our latest developer guides and tips follow us on Twitter and subscribe to our YouTube Channel!

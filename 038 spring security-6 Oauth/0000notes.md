# Notes

>Note: First Authentication happens and then authorization!!

![alt text](image.png)

![alt text](image-1.png)

Already seen in Oauth in HLD!!In 8th step we can have JWT or Opaque token or any other type of token!!

![alt text](image-2.png)

OAuth is used to acess protected data!! OIDC can be used to veryify the user too!!

OpenID Connect is built on top of OAuth 2.0 and adds an identity layer. OAuth 2.0 is primarily for authorization, providing access tokens to access resources. OpenID Connect adds an identity token that contains user authentication information, allowing client apps to verify the user's identity. So, OAuth 2.0 handles access control, while OpenID Connect handles user authentication and identity.

## OIDC
OAuth 2.0 is designed only for authorization. It is used for granting access to data and features from one application to another. In OAuth, the client is given a token which it uses to access the data on the resource server, but it doesn’t get to know anything about the user.

OAuth 2.0 works by having a client application request permission from a user to access resources on a resource server. Once authorized, the client receives an access token, which it uses to make API calls to the resource server. This token represents the granted permissions but does not include user identity details, ensuring the client accesses data without knowing who the user is. This separation enhances security and privacy by limiting the client’s knowledge to authorization scope only.

You may have seen that when you try to login to an app, then the app can prompt you to authenticate using your Facebook or Google account. In this case, the app is probably using OpenID Connect. OpenID Connect allows a range of clients, including web-based, mobile, and JavaScript clients, to request and receive information about authenticated sessions and end-users.


### Local user authentication
Let’s suppose we are not using OpenID Connect or any Identity Provider (discussed in the next section) for authentication. In that case, our application will have to maintain a local database in which we will store user information and credentials. This seems like an easy solution but there are a few problems:

- If our organization has lots of applications, then maintaining a database for each application is a tedious task. It requires resources and staff to manage.
- Users find the task of signing up as cumbersome. First, they need to remember so many passwords, and second, it is likely that they’ll use the same password everywhere. If that password gets compromised, all the applications that the user uses with that password will also become compromised

### Identity Providers
To solve the problem of local authentication, Identity Providers came in the picture. As the name suggests, Identity Providers take care of your authentication needs while you focus on your main business. They provide the identity of the user so that organizations can directly onboard the user without asking for any details.

It is a win-win situation for both the user and the organization. The organization is not required to store the personal data of its user, and the users are saved from creating an account each time they need to use some application.

![alt text](image-42.png)

## OpenId Connect Terminologies

1. Identity token
While discussing OAuth, we discussed the authorization code and access token. In the case of OpenId Connect, there is one more token that we can request. This token is called the identity token, which encodes the user’s authentication information.

    In contrast to access tokens, which are only intended to be understood by the resource server, ID tokens are intended to be understood by the client application. The ID token contains the user information in JSON format. The JSON is wrapped into a JWT.

    When a client receives the identity token, it should validate it first. The client must validate the following fields:

    - iss - Client must validate that the issuer of this token is the Authorization Server.
    - aud - Client must validate that the token is meant for the client itself.
    - exp - Client must validate that the token is not expired.

ex:

```json
{
  "iss": "https://server.example.com",
  "sub": "24400320",
  "aud": "s6BhdRkqt3",
  "nonce": "n-0S6_WzA2Mj",
  "exp": 1311281970,
  "iat": 1311280970,
  "auth_time": 1311280969,
  "acr": "urn:mace:incommon:iap:silver"
}
```
Let’s look at what these values mean:

- iat - The iat claim identifies the time at which the JWT was issued. This claim can be used to determine the age of the JWT. Its value must be a number containing a NumericDate value.

- auth_time - Time when the End-User authentication occurred. Its value is a JSON number representing the number of seconds from 1970–01–01T0:0:0Z as measured in UTC until the date/time.
- nonce - ID token requests may come with a nonce request parameter to protect from replay attacks. When the request parameter is included, the server will embed a nonce claim in the issued ID token with the same value of the request parameter.

>note: The Identity token contains only basic information about the user. To get the complete user information, the client must send the access token (please note access token should be sent not identity token) to UserInfo endpoint.

![alt text](image-43.png)

![alt text](image-44.png)

![alt text](image-45.png)

### Authorization Code Flow for Authentication

The Authorization code flow for OpenID Connect is similar to the Authorization Code Flow that we discussed in the OAuth 2.0 . The only difference is the change in the value of the scope field. It must contain openid as one of the values, followed by other scope values based on what type of user data the client wants.


- What would happen if the client does not provide an openid in the scope field while sending a request to the authorization server?

    The answer is that in this case, the flow will work as a normal authorization flow. The client app will not get access to the user information as it will not receive the identity token.

- Can user information be fetched from the UserInfo endpoint by sending the access token in the request even if openid was not provided in the scope field when an access token was requested?

    The answer is NO. When we send a request to the token endpoint to fetch the access token, then we must send openid in the scope field. We must also send other scope values like email or address if we want to get this information. The access token that is returned is based on the scope values that were sent with the request. When we hit the UserInfo endpoint, then only that user information is returned which the access token is authorized to get.

    Let’s say that while sending a request to the token endpoint, the scope value is “openid email”. The client sends this request and gets an access token. If the client sends this access token to the UserInfo endpoint, it will get only email information. It will not get an address or any other information.

![alt text](image-46.png)

![alt text](image-47.png)

If the scope does not include 'openid', the flow works as a normal OAuth 2.0 authorization flow. In this case, the client app will not receive an identity token and cannot access user information through the UserInfo endpoint.
### Implicit Code Flow for Authentication
This flow is also similar to the Implicit grant type discussed in the OAuth chapter. This flow is used for single-page JavaScript apps or those apps which do not have a backend.

In Implicit flow, the response_type field can either take token or id_token or token_id_token as value. This leads to some interesting cases depending upon what is provided in the scope field.

![alt text](image-48.png)

![alt text](image-49.png)

![alt text](image-50.png)

### Hybrid Code Flow for Authentication

In Authorization flow, we first get authorization token from authorization endpoint and then get the access token and identity token from the token endpoint. This takes some time as two server calls are needed.

In the implicit flow, we get the access token and identity token from the authorization endpoint. This is faster but is not secure.

In the hybrid flow, the client gets immediate access to the identity token from the authorization endpoint itself. The client also gets the authorization code from the authorization endpoint. Later, it fetches the access token from the token endpoint which can be used to get further user info.

#### Hybrid flow type#
In hybrid flows, the response_type field should contain code as one of the values and token or id_token or token+id_token as the other value.

![alt text](image-51.png)

![alt text](image-52.png)

![alt text](image-53.png)

In the hybrid code flow, tokens can be issued by both the authorization endpoint and the token endpoint because they serve different purposes and have different security considerations. The authorization endpoint issues tokens immediately to the client for faster access, but these tokens may have fewer claims or different security properties. The token endpoint issues tokens after the client exchanges the authorization code, allowing for more secure and complete tokens. This separation helps balance speed and security. For example, the access token from the authorization endpoint and the token endpoint may differ due to different lifetimes or security scopes.


OpenID Connect (OIDC) builds on OAuth 2.0 by adding authentication on top of authorization. While OAuth 2.0 handles authorization (granting access to resources), OIDC adds the identity layer, allowing the client to authenticate the user and obtain user information via the identity token and UserInfo endpoint.

OAuth 2.0 was designed primarily for authorization, meaning it grants access to resources without verifying the user's identity. It focuses on allowing third-party applications to access user data with permission. Authentication, which is about verifying who the user is, was not part of OAuth's original purpose. OpenID Connect was introduced on top of OAuth 2.0 to add this authentication layer, enabling clients to confirm the user's identity.

![alt text](image-3.png)

![alt text](image-4.png)

### Auth0 Registration:

![alt text](image-5.png)

![alt text](image-6.png)


![alt text](image-7.png)

After registration, we get Client id and Secret

![alt text](image-8.png)

![alt text](image-9.png)

![alt text](image-10.png)

### SecurityConfig.java

![alt text](image-11.png)

Basic Controller class, just for testing

![alt text](image-12.png)

Notice one thing that, I have not created any User in our Springboot app.

![alt text](image-13.png)

![alt text](image-14.png)


![alt text](image-15.png)


![alt text](image-16.png)


What? With just pom.xml, application.properties and SecurityConfig.java changes, we able to run the complete OAuth2 flow.

Answer is Yes, Springboot Security framework provides the compete functionality of OAUTH2 protocol, we don’t have to code anything.


But here is the twist:

When I tried to access the "/users" API through postman (not through browser), it takes me back to Login page, why?

![alt text](image-17.png)

Because, Springboot assume that, Oauth2 login will be done on a browser, so by-default it creates SESSION.

![alt text](image-18.png)

![alt text](image-19.png)

![alt text](image-20.png)

just rightmost image 

![alt text](image-21.png)

![alt text](image-22.png)

![alt text](image-23.png)

![alt text](image-24.png)

![alt text](image-25.png)


![alt text](image-26.png)

![alt text](image-27.png)

![alt text](image-28.png)


![alt text](image-29.png)


![alt text](image-30.png)


![alt text](image-31.png)


![alt text](image-32.png)

OAuth2LoginAuthenticationFilter parent class has one method "onAuthenticationSuccess()" at last,  which by default do some cleanup task once authentication process completes, I have overwrite that method and set the ID_TOKEN in the response body.

![alt text](image-33.png)

![alt text](image-34.png)

Start the application

![alt text](image-35.png)


After Authorizing from the Authorization server:

![alt text](image-36.png)


Now, only 1 task left, now Client will pass this TOKEN with every request and we have to verify it.


![alt text](image-37.png)

Created New Filter to Validate the token

![alt text](image-38.png)

Utility Class, just to validate the token

![alt text](image-39.png)

![alt text](image-40.png)

![alt text](image-41.png)
















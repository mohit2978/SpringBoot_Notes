# Notes

![alt text](image.png)

![alt text](image-1.png)

![alt text](image-2.png)

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
















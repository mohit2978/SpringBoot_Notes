# Notes

- We have already covered different type of Authentications.
- And with that, we have also covered, how at Security Filter layer itself we can do authorization.

![alt text](image.png)

- But in large scale applications, where we might have 100s of APIs, in that scenarios managing the roles at Security Filter might become difficult and cause scalable issue.

That’s where, Annotation based "Role Based Authorization" comes into the picture. And these annotations are used within our Controller class.

![alt text](image-1.png)

See we know Authentication object reaches controller so using that we will so Authorization

![alt text](image-2.png)


##  Dynamic User creation(Already seen)

![alt text](image-3.png)

One User can have multiple permissions

---

![alt text](image-4.png)

---

![alt text](image-5.png)

---
![alt text](image-6.png)

---

![alt text](image-7.png)

Now, we should be able to create user, and in this one User can have 1 role like ROLE_ADMIN, ROLE_USER etc.
But we can give many permissions like ORDER_READ, SALES_DELETE etc.…  (more granular level permissions)

![alt text](image-8.png)

![alt text](image-9.png)

![alt text](image-10.png)

Above we have created user, who has both the permissions ROLE_USER and ORDER_READ, so API is successfully executed.

![alt text](image-11.png)

Now, when we try to access /sales API, which above user do not have permission, request has thrown exception.

![alt text](image-12.png)

![alt text](image-13.png)


### SecurityExpressionRoot.java

![alt text](image-15.png)
![alt text](image-14.png)

Now, one question comes to mind is, how this Authorization methods invokes before invocation of the Controller method.

Its because of INTERCEPTORS

![alt text](image-16.png)

@PreAuthorize Annotation is intercepted by 

AuthorizationManagerBeforeMethodInterceptor

What and how Interceptors works and how they are different from filters? 
We have already covered both the topics in depth. 

![alt text](image-17.png)


![alt text](image-18.png)

![alt text](image-19.png)

![alt text](image-20.png)

Let's see with an example

Created 2 Users

![alt text](image-21.png)

![alt text](image-22.png)

![alt text](image-23.png)

![alt text](image-24.png)

OrderDto is typecasted to `returnObject` and before returning we are checking the order returning is ours or not.

Now, try to invoke this api with "b_users" credentials

![alt text](image-25.png)

Now, try to invoke this api with "a_users" credentials

![alt text](image-26.png)





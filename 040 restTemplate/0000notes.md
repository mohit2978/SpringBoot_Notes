# Notes

![alt text](image.png)

Order Service runs on port 8081 and ProductService on 8082!!

2 types of communication

1. Synchronous
2. Asynchronous

![alt text](image-1.png)

![alt text](image-2.png)

![alt text](image-3.png)

![alt text](image-4.png)


![alt text](image-5.png)

![alt text](image-6.png)

![alt text](image-7.png)

Let's first see, without using above SpringBoot communication types, what it takes to invoke the REST endpoint just using plain JAVA.

![alt text](image-8.png)

![alt text](image-9.png)

![alt text](image-10.png)

![alt text](image-11.png)

![alt text](image-12.png)


![alt text](image-13.png)

![alt text](image-14.png)

![alt text](image-15.png)

![alt text](image-16.png)

![alt text](image-17.png)

Lets see, some of the other methods which are available in RestTemplate

![alt text](image-18.png)

![alt text](image-19.png)

![alt text](image-20.png)

![alt text](image-21.png)

![alt text](image-22.png)

### Limitations of Rest Template 

- In RestTemplate, there are already so many overloaded methods, so its hard to remember and maintain.(Above we have just covered few)

- RestTemplate was build before concepts like Retry, circuit breaker etc.. So adding support means more overloaded methods and not user friendly.

- RestTemplate is in Maintenance mode - means no new feature, only bug fixes.

That’s where latest RestClient comes into the picture:

- Introduction of Fluent, builder-style API (more readable and user friendly way of configuring and invoking the endpoint)

- RestClient supports easy integration with interceptors, filters etc.































































It is even used today so learn it!!

**HATEOAS**

**H**ypermedia **A**s **T**he **E**ngine **O**f **A**pplication **S**tate

It tells the client, what the next action you can perform on particular item.

**For example:**

State flow diagram:
- POST: /adduser → **Unverified User Added**
- **Unverified User Added** → GET User (GET: /getuser)
- **Unverified User Added** → DELETE User (DELETE: /removeuser)
- **Unverified User Added** → VERIFY User
- VERIFY User → SMS-VERIFY STARTED (POST: /sms-verify-start) → SMS-VERIFY FINISHED (POST: /sms-verify-finish)
- VERIFY User → EMAIL-VERIFY STARTED (POST: /email-verify-start) → EMAIL-VERIFY FINISHED (POST: /email-verify-finish)

after POST you can go various APIS is told by HATEOAS

The below image tells how a response looks like in HATEOAS

![svg](<svgs/img1-hateoas-definition.svg>) ![svg](<svgs/img1-user-state-flow-diagram.svg>) API Response, after *"Unverified User Added"* POST: /adduser API

Without *HATEOAS* link:
```json
{
    "userID": "123456",
    "name": "SJ",
    "verifyStatus": "UNVERIFIED"
}
```

With *HATEOAS* link:
```json
{
    "userID": "123456",
    "name": "SJ",
    "verifyStatus": "UNVERIFIED",
    "links": [
        {
            "rel": "self",
            "href": "http://localhost:8080/api/getUser/123456",
            "type": "GET"
        }
    ]
}
```

See links in HATEOAS where it tells what next action can be performed!!

Rel tells what is relation of this api self means this is the same API!!

Before understanding HOW to do this, lets understand WHEN to use and WHY to use HATEOS link?

2 Major Purpose of using HATEOAS link is to achieve

- "LOOSE COUPLING" and
- "API DISCOVERY"

To achieve above, server provides the next set of APIs (actions) in the Response itself, which client can take. So that client have less business logic around APIs (which API to invoke, when to invoke, how to invoke etc....)

But, Adding all next set of ACTIONS can make our API Response Bloat up and has several disadvantages:

- Increase complexity at server side.
- Latency impact.
- Increase Payload size.

API discovery means we want to give all APIs which can be invoked after this APi!! We are telling client which next set of APIs to be used now!! So client has less business logic around API!!

Loose Coupling means loose couple between client and server!!

![svg](<svgs/img2-response-with-without-hateoas.svg>) ![svg](<svgs/img2-when-why-hateoas.svg>) 


```json
{
    "userID": "123456",
    "name": "SJ",
    "verifyStatus": "UNVERIFIED",
    "links": [
        {
            "rel": "self",
            "href": "http://localhost:8080/api/getUser/123456",
            "type": "GET"
        },
        {
            "rel": "remove",
            "href": "http://localhost:8080/api/removeuser/123456",
            "type": "DELETE"
        },
        {
            "rel": "update",
            "href": "http://localhost:8080/api/updateuser/123456",
            "type": "PATCH"
        },
        {
            "rel": "verify-start",
            "href": "http://localhost:8080/api/sms-verify-start/123456",
            "type": "POST"
        },
        {
            "rel": "verify-finish",
            "href": "http://localhost:8080/api/sms-verify-finish/123456",
            "type": "POST"
        }
    ]
}
```

Never Ever add all possible next set of actions (API) just like that. ❌

Our response looks something like this!!! Self tells this is current API and other API tells what other API can be called!! If you add all next set of action , it will increase server side complexity as these link are put dynamically!!

Adding this business logic we are increasing payload size!!

We cannot add all URLs , we need to properly analyze which url to add!!

![svg](<svgs/img3-never-add-all-actions.svg>)



 
 
 
 So, proper analysis need to be done, what actually will help us to achieve LOOSE COUPLING

Now, if we see the above diagram again

State flow diagram:
- POST: /adduser → **Unverified User Added**
- **Unverified User Added** → GET User (GET: /getuser)
- **Unverified User Added** → DELETE User (DELETE: /removeuser)
- **Unverified User Added** → VERIFY User
- VERIFY User → SMS-VERIFY STARTED (POST: /sms-verify-start) → SMS-VERIFY FINISHED (POST: /sms-verify-finish)
- VERIFY User → EMAIL-VERIFY STARTED (POST: /email-verify-start) → EMAIL-VERIFY FINISHED (POST: /email-verify-finish)



![svg](<svgs/img4-loose-coupling-analysis-diagram.svg>)

See above diagram!! Let us suppose we not using HATEOAS

 I see, the tight coupling lies during VERIFY process.

Client need some info, before it can decide which Verify API to invoke. For example:

```json
{
    "userID": "123456",
    "name": "SJ",
    "verifyStatus": "UNVERIFIED",
    "verifyType": "SMS",
    "verifyState": "NOT_YET_STARTED"
}
```

Client need to put business logic, that

```
if(VerifyStatus == "UNVERIFIED")
{
     if(verifyType== "SMS")
     {
          if(verifyState== "NOTE_YET_STARTED")
          {
               Call POST: /sms-verify-start
          }
          Else if (verifyState == "STARTED")
          {
               Call POST: /sms-verify-finish
          }
     }
     Else if(verifyType == "EMAIL")
     {
          if(verifyState== "NOTE_YET_STARTED")
          {
               Call POST: /email-verify-start
          }
          Else if (verifyState == "STARTED")
          {
               Call POST: /email-verify-finish
          }
     }
}
```

 This on right is client side program!!

This client side program tells which api to call!!This is tight coupling based on response by server the api will be called now suppose HATEOAS comes!!

This dependency, can be removed by HATEOAS link

```json
{
    "userID": "123456",
    "name": "SJ",
    "verifyStatus": "UNVERIFIED",
    "links": [
        {
            "rel": "verify",
            "href": "http://localhost:8080/api/sms-verify-finish/123456",
            "type": "POST"
        }
    ]
}
```

Now, we have achieved LOOSE COUPLING and Client code looks like this.

```
if(verifyStatus == "UNVERIFIED") {

     //invoke the verify URI, given in HATEOAS link

}
```

![svg](<svgs/img5-tight-coupling-client-logic.svg>) 


![svg](<svgs/img5-loose-coupling-with-hateoas.svg>) 


 Here HATEOAS gives API which needed to be called is sent to client so client just need to call that link so it becomes loose coupling!!

**Dependency Required:**

```xml
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-hateoas</artifactId>
<version>2.6.4</version>
</dependency>
```

This is dependent we need!!

```java
@RestController
@RequestMapping(value = "/api/")
public class UserController {

    @Autowired
    User user;

    @PostMapping(path = "/adduser")
    public ResponseEntity<UserResponse> addUser() {

        UserResponse response = user.getUser();

        //our business logic to determine which Verify API need to be invoked.
        Link verifyLink =  WebMvcLinkBuilder.linkTo(UserController.class)
                .slash( object: "sms-verify-finish")
                .slash(response.getUserID())
                .withRel("verify")
                .withType("POST");

        response.addLink(verifyLink);

        return new ResponseEntity<>(response, HttpStatus.OK);
    }
}
```
```java
public class HateoasLinks {

    private List<Link> links = new ArrayList<>();

    public void addLink(Link link) {
        links.add(link);
    }
}
```
```java
public class UserResponse extends HateosLinks {

    private String userID;
    private String name;
    private String verifyStatus;

    //getters and setters here
}
```

UserResponse has HatOAS links as it extends HATEOAS links  you can see!!
Just to be clear!!

![svg](<svgs/img6-dependency-required.svg>) ![svg](<svgs/img6-controller-hateoaslinks-userresponse.svg>)



```java
@RestController
@RequestMapping(value = "/api/")
public class UserController {

    @Autowired
    User user;

    @PostMapping(path = "/adduser")
    public ResponseEntity<UserResponse> addUser() {

        UserResponse response = user.getUser();

        //our business logic to determine which Verify API need to be invoked.
        Link verifyLink =  WebMvcLinkBuilder.linkTo(UserController.class)
                .slash( object: "sms-verify-finish")
                .slash(response.getUserID())
                .withRel("verify")
                .withType("POST");

        response.addLink(verifyLink);

        return new ResponseEntity<>(response, HttpStatus.OK);
    }
}
```

Another way to add link is below!!

![svg](<svgs/img7-usercontroller-adduser-link.svg>)



 
 Other way to create Link:

```java
Link verifyLink =  Link.of( href: "/api/sms-verify-finish/" + response.getUserID())
        .withRel( relation: "verify")
        .withType("POST");
```

```json
{
    "userID": "123456",
    "name": "SJ",
    "verifyStatus": "UNVERIFIED",
    "links": [
        {
            "rel": "verify",
            "href": "http://localhost:8080/api/sms-verify-finish/123456",
            "type": "POST"
        }
    ]
}
```

![svg](<svgs/img8-other-way-create-link.svg>)

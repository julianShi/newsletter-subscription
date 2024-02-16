## Newsletter Subscriptioin Architecture Design

We want to design a newsletter subscription solution for an e-commerce application. The e-commerce application has already adopted service-oriented architecture (SOA). 

## User Analysis
<img width="176" alt="Screen Shot 2024-02-16 at 8 39 22 PM" src="https://github.com/julianShi/newsletter-subscription/assets/11760687/c8c70fd4-7957-4501-81ee-772c6e26758d">

Take the above LinkedIn screenshot on https://www.linkedin.com/mypreferences/d/categories/notifications for example, the front-end majorly wants to read and update the newsletter subscription metadata. Specifically. 

- Users are asked contact information and newsletter preference when they create an account. 
- The UI allows users to customize the newsletter topic, channel. Example of topics are "marketing", "discount" etc. channel examples are "email", "sms", "in-app" etc. 
- Per GDPR requirement, users should be prompt with the link to unsubscribe when they receive newsletters. 
- Assuming 100 new user registrations daily, and 1000 daily new users during promotion, we use 0.01 query per second (qps) to estimate the traffic. 

Other internal services also want to query the newsletter subscription metadata in the following scenario. 

- The marketing team wants to have a range query on the newsletter subscription metadata. For example, `SELECT * FROM subscriptions WHERE user_id between 1000 and 1999`. If there are around one million users to query, then data warehouse if favorred over remote processing call (RPC). 
- The marketing team may also want to query using user_id for more specific marketing targets. `SELECT * FROM subscriptions WHERE user_id in (1001, 1003) 

- The data analysis team wants to query the metadata in data warehouses instead. 

## Architecture Design in [C4 model](https://en.wikipedia.org/wiki/C4_model)

The overall newsletter subscription metadata life-cycle can be visualized in the following container-level diagram. The upper flows in the diagram are about the metadata manipulation, the lower flows are about consuming the metadata. 

<img src="https://plantuml.gitlab-static.net/png/U9nrKJrFmp0GtVqhJjaBUmScEY0X18qgXf3XR1-9rSIkR0S6n7_dOaieSThDFh-lUqOLdOSfa1VYWkgC7K5ruYlK5AEn7PoUAlWH04qzoQ2ykKJZB4zRyRkb-2-ZAECrHfGO25vS_VOyW_ydrIEVu1qzzOwjguLEhNhIqr2ADKUoceS7snbBQ-l3Y6QuuPqVxrzxPzc6MPZs7Pqq0manxmsxtGCqnShj7b1hKCv6Pe2nd-x3Xbo098WEB7s7WV5StAQPbMIArS8UmX8rKiGvfH1DwgU5kqHQDzFUOXNYcRigOdPSncUZGZkdt1JEdXAZwY5YE8ihxt3TpASV8hrgfa2bdBbyPYo0Vxv0slS0" alt="img" style="zoom:80%;" />

I isolate the user service as follows for further analysis. 

<img src="https://github.com/julianShi/newsletter-subscription/assets/11760687/f40ca14b-9335-4130-9869-e68623c0ab2a" alt="user-service" style="zoom:80%;" />

Noted here

- In typical domain-driven designs (DDD) of e-commerce application kernels, there are domains and containers of 7 domains, namingly product service, trading service, payment service, inventory service, shipping service, user service, order service. Supporting domains like data analysis, monitoring are out of the scope of the application kernel. 

- It's not recommended to create a new container or a new database table solely for the newsletter subscription metadata manipulation. This function actually falls into the domain of user service. 
- The user metadata are recommended to be saved in NoSQL (document database). Since a user manipulates only its own metadata, there is no need to adopt a transactional database. 

- No need to create a new database table. We can simply add a new "subscription" field in the user metadata document table for all subscription metadata. 
- "contact_info" is another field in the user metadata document that saves email address, phone number, mobile token etc. "contact_info" is different from "subscription.channels". Email addresses are personal identifiable information (PII). We want to keep the PII data only in certified services. 

## Interface Schema Design

Get the subscription metadata, and render on the Subscription page. Usually, there is default subscription setting when a user creates an account. So I only show the OpenAPI schema snippet of the GET and the POST operations. The creation and deletion operations are supposed to be used by internal services only. OpenAPI and Swagger provide succinct interface schema designs in yaml format as follows. 

```yaml
paths:
  /user/subscription/{user_id}:
    get:
      summary: Get subscription metadata
      operationId: getSubscription
      tags:
        - subscriptions
      requestBody:
        required: false
      responses:
        '200':
          description: Subscription read successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SubscriptionResponse'
        '400':
          description: Invalid input
        '500':
          description: Internal server error

    post:
      summary: Update subscription metadata
      operationId: updateSubscription 
      tags:
        - subscriptions
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/SubscriptionRequest'
      responses:
        '200':
          description: Subscription updated successfully
        '400':
          description: Invalid input
        '404':
          description: Subscription not found
        '500':
          description: Internal server error
```

schema used in the above two APIs. 

```yaml
components:
  schemas:
    SubscriptionRequest:
      type: object
      properties:
        topics:
          type: array
          items:
            type: string
        channels:
          type: array
          items:
            type: string

    SubscriptionResponse:
      type: object
      properties:
        topics:
          type: array
          items:
            type: string
        channels:
          type: array
          items:
            type: string
```

Examples: 

```bash
GET /user/subscription/123513
response={
  "channels": ["email","sms","in-app"] 
  "topics": ["marketing", "discount", "new-product"] 
}
```

> We can make the channels enumerable 

```bash
POST /user/subscription/123513
request={
  "channels": ["email","sms","in-app"]
  "topics": ["marketing", "discount", "new-product"]
}
response={
  "code": 200,
  "message": 
}
```

## REST API Framework Design

Popular REST frameworks for enterprises are Python Django REST Framework and Java Spring Boot. I sample a few comparisons as follows

| framework   | Django Rest Framework in Python                              | Spring Boot in Java                                          |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| syntax      | Python scripting language is more succinct, is easier to onboard. | Java is verbose. But the annotation and dependency injection simplified coding as well. |
| performance | It handles 1 qps.                                            | Better performance                                           |
| ORM         | Django ORM。It's relatively slow                             | MyBatis is fast due to the compiled data types               |
| Operation   | Great community support. Quickly deployable without compiling | Better community support                                     |

Of course, the choice will be made by team preference eventually. 

Let me write some code in Django for this task. 

"view.py"

```python
from django.http import JsonResponse
from model import get_newsletter_subscription, update_newsletter_subscription

def newsletter_subscribe(request, _):
    if request.method == 'GET' or request.method == 'get':
        data = get_newsletter_subscription(request)
        rsp = JsonResponse({'data': data, 'code': 200, 'errorMsg': 'successfully read the subscription'})
    elif request.method == 'POST' or request.method == 'post':
        update_newsletter_subscription(request)
        rsp = JsonResponse({'code': 200, 'errorMsg': 'successfully updated the subscription'})
    else:
        rsp = JsonResponse({'code': 400, 'errorMsg': 'invalid request method'})
    return rsp
```

"model.py"

```python
import pymongo
from django.conf import settings

my_client = pymongo.MongoClient(CONNECT_STRING)
dbname = my_client['dbname']
collection_name = dbname["user_info"]

def get_newsletter_subscription(user_id):
  user_info = collection_name.find({'user_id': 12351})
  subscription = {}
  subscription['topics'] = user_info['topics']
  subscription['channels'] = user_info['channels']
  return subscription

# Update one document
def update_newsletter_subscription(user_id, request):
  topics = request['topics']
  channels = request['channels']
  collection_name.update_one({'user_id': user_id}, {'$set':{'channels': channels, 'topics': topics}})
```

## Document Database Design

As discussed, document databases are more scalable, and therefore better choices for this task. What's more, the traffic is around 0.01 qps, the concurrency issue is less a concern. 

```json
{
  "user_id": "1231",
  "language_code": "en",
  "country_code": "us",
  "subscription": {
    "channels": ["email", "sms"],
    "topics": ["transaction", "marketing"],
    "customized_topics": ["sunglass"]
  },
  "contact_info": {
    "emails": ["encrypted-email-address"],
    "phones": ["encrypted-phone-number"]
  }
}
```

REST API or GraphQL QueryLanguage (e.g. Apache Thrift, GraphQL)

## Document Database Comparison

We make a simple comparison between the popular MongoDB, AWS DynamoDB. 

| database       | MongoDB                                | DynamoDB                         |
| -------------- | -------------------------------------- | -------------------------------- |
| data structure | JSON. More flexible.                   | K-V, which is faster.            |
| Support        | support from the open-source community | Better in-time support from AWS. |

Since we develop on top of an existing service, either MongoDB or DynamoDB depends on the existing tech stack. 

## Security Design

We use enums like "email" or "sms" as options of the "subscription.channels" field. This avoid saving actual email address or phone number in database. 

This function is less critical, as it has nothing to do with payment. Still, it's important to avoid user data leaking. 

## Middleware

###Authentication, throttle

This new REST API is not exposed to users. So there is no need for user authorization. We can trust the user_id decrypted, and passed by the front-end. 

There are plenty of third-party middlewares for throttling, for example Spring Cloud Gateway, Alibaba Sentinel. 

### Monitoring

We want to collect, process, tranport data, and to alert for certain pattern. There are popular middlewares like Prometheus、Grafana、Zabbix for our choice. 

## Looking Forward

### Extendability

For extension in the future, we might want to allow customers to custmize the newsletter according to the hash tags they choose. The noSQL provides a better scalability and flexibility. For example, some customer wants the newsletter about sunglasses. In contast, MySQL requires altering the table for adding a new column. 

### Compatibility

There is a scenario when the users do not register at all. For example, the users may come from third-party social network platforms. If we still want to retain these users, we will provide these users the capacity to unsubscribe. 

## Miscellaneous

### Email Service

As we recall that we plotted the Email Service as a downstream of the User service, I explain briefly here. 

- Whenever there is an update of newsletter subscription metadata, there is a message send to the data warehouse through a message queue. The data are consolidate and deduplicated in the data warehouse hourly or daily. 
- The Scheduler Service reads the newsletter subscription metadata in batch, and decide what topics and by what channels to send to users. 
- There is a database in the Email Service that maps user_id to a list of decrypted email addresses and phone numbers. 
- The Scheduler Service calls the Email Service to send the rendered emails to users. 

Code examples

```bash
POST /newsletter/feed
request={
  "user_id": 123513,
  "context": "rendered text in user language",
  "channels": ["sms","in-app"]
}
```

```bash
POST /newsletter/feed
request={
  "user_id": 123513,
  "context": "rendered email in user language",
  "channels": ["email"]
}
```

Nodemailer, SendGrid are options of email service brokers. 

Twilio is a famour SMS service broker. 

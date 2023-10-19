# Transforming an ERP Platform to Cloud-Native Microservices with MongoDB Models

In my capacity as the lead architect at [Hybrid Web Agency](https://hybridwebagency.com/), one of the most formidable projects I've ever undertaken involved the conversion of an archaic monolithic ERP system into a contemporary microservices architecture. The outdated monolith had become an unwieldy financial burden to maintain, constraining my client's ability to foster innovation.

Fortunately, we had confronted similar challenges on numerous occasions during our years of providing tailored [Software Development Services in Charlotte](https://hybridwebagency.com/charlotte-nc/best-software-development-company/). We were acutely aware of the predicaments organizations faced when saddled with inflexible and resistant-to-change systems. This is precisely why I am enthused to share the systematic approach we adopted to overhaul the data model and dismantle the monolithic structure, both at the code and database tiers.

Throughout this migration endeavor, my team and I unearthed some lesser-known best practices for shaping domain entities in a document database like MongoDB, all in the pursuit of supporting fully autonomous microservices. If you've ever pondered over how to future-proof your database architecture for the cloud while preserving historical data, you're in for a treat with these strategies.

As you peruse this article, you'll gain access to a comprehensive blueprint for migrating your own legacy systems to a modern architectural framework. I will dispense a wealth of practical insights to help you steer clear of common pitfalls and expedite the delivery of fresh features to your clientele. So, let's embark on this journey of transformation!

## The Merits of a Microservices Architecture

The adoption of a microservices architecture bequeaths manifold advantages over the traditional monolithic approach. Microservices stand as independently deployable entities, endowing organizations with the capability to swiftly develop and release features sans the disruption of the entire application.

Moreover, individual services can be crafted using diverse programming languages and frameworks, affording organizations the freedom to cherry-pick the most fitting technologies for each domain. To illustrate, a recommendation engine might leverage Python's machine learning libraries, while a user interface could be elegantly constructed with React.

This decoupling of concerns empowers specialized teams to operate autonomously on discrete services. It also paves the way for the rapid validation of ideas through prototypes before fully committing to a monolithic overhaul. In addition, new team members can seamlessly contribute to a single service that aligns with their proficiencies.

In an era marked by the rapid evolution of technology trends, microservices mitigate the risks associated with these transitions. Replacements only affect small, isolated segments of the system, circumventing the need for an all-encompassing overhaul of the entire monolith. Precisely defined interfaces facilitate the seamless migration of components to new implementations.

#### Independent Scalability

Effective resource management materializes when services can scale independently in response to demand. Consider, for example, a frontend API gateway adeptly routing traffic based on URLs. It can securely deploy behind a load balancer to gracefully manage traffic surges. During peak times, such as holidays, only the order processing service necessitates additional servers, averting the unnecessary scaling of the entire system.

```
ordersAPI.scale(replicas: 5);
```

Horizontal scaling at the service level translates into substantial cost savings through the precise allocation of resources. Idle support microservices, like user profiles, no longer warrant expensive overprovisioning to accommodate traffic that bears no impact on them.

## Scrutinizing the Data Model

### Comprehending Entity Relationships

The inaugural step in transitioning to microservices entails a meticulous analysis of how entities interrelate within the monolithic data model. We conducted an exhaustive examination of each collection in the MongoDB database, meticulously pinpointing clusters of domains and transactional boundaries.

Ubiquitous entities such as Users, Products, and Orders materialized as pivotal elements within bounded contexts. The relationships binding these core entities became prime candidates for service deconstruction. For instance, we discerned that Orders harbored foreign keys linking to Users for customer information and Products to represent purchased items.

To obtain a deeper grasp of these interdependencies, we generated sample documents to visually represent associated fields. An eye-opening revelation surfaced: legacy code redundantly stowed data that was now pertinent to separate business capabilities. For instance, shipping addresses needlessly duplicated user profiles instead of referring to lightweight counterparts.

The analysis of relationships brought to light perniciously snug interconnections between modules, engendering cascading updates. The act of normalizing redundant data cleared the path for the independent development of user profiles and shipping namespaces.

We harnessed database tools to navigate the intricate network of connections. Utilizing MongoDB Compass, we meticulously charted relationships through $lookup pipelines and executed aggregate queries to tally references between entities. This exhaustive process unearthed pivotal breakpoints for segmenting logic into cohesive services.

These relationships served as the guiding star for demarcating domain boundaries and ensuring that services presented lucid, well-defined interfaces. These well-defined contracts empowered autonomous teams to incrementally develop and deploy modules as micro frontends, free from the shackles of mutual dependency.

### Identifying Transactional Boundaries

Beyond delving into relationships, we embarked on a thorough investigation of transactions within the existing codebase, meticulously deciphering the flow of business processes. This endeavor cast light on areas where data modifications required confinement within individual services to uphold data consistency and integrity.

For instance, within the realm of order processing, we came to the realization that any updates pertaining to the order itself, associated payments, inventory levels, and shipment notifications must occur within a single service, in a transactional manner. This profound insight informed the delineation of our Order Management service boundary.

The painstaking analysis of both relationships and transactions supplied invaluable insights that steered the refactoring of the data model and logic into independently deployable microservices, each distinguished by unequivocally defined interfaces.

## Refactoring for Microservices

### Normalizing Data Schemas

To accommodate independent services that might opt for different data stores, we embarked on the normalization of schemas. The goal was to eradicate redundancy and encompass only the essential data mandated by each service.

Take, for instance, the original Orders schema, which encapsulated the entire User object. We streamlined this by introducing a lightweight reference:

```
// Before
Orders: {
  user: {
    name: 'John',
    address: '123 Main St...' 
  }
  //...
}

// After  
Orders: {
  userId: 1234
  //...  
}
```

Analogously, we extracted product details from Orders and assigned them to their dedicated collections. This approach liberated these entities to evolve independently over time.

### Leveraging Domain-Driven Design

We harnessed the concept of bounded contexts from Domain-Driven Design to logically segregate services, be it Order Fulfillment or User Profiles. Interfaces played a pivotal role in abstracting data access:

```
interface UserRepository {
  getUser(id): User;
  updateProfile(user: User): void;
}

class MongoUserRepository implements UserRepository {

  users = db.collection('users');

  async getUser(id) {
    return this.users.findOne({_id: id}); 
  }

  //...
}
```

### Evolving Data Access Patterns

Queries and commands underwent a substantial overhaul to align with the new architectural paradigm. In the past, services accessed data directly using calls like `db.collection.find()`. We introduced abstraction through data access libraries:

```
// order.service.ts

constructor(private ordersRepo

: OrderRepository) {}

getOrders() {
  return this.ordersRepo.find();
}

// mongodb.repo.ts

@Injectable()
class MongoOrderRepository implements OrderRepository {

  constructor(private mongoClient: MongoClient) {}

  find() {
   return this.mongoClient.db.collection('orders').find(); 
  }

}
```

This architectural shift ensured flexibility, affording us the latitude to migrate databases without necessitating alterations to consumer code.

## Deploying Microservices

### Independent Scaling

In the realm of microservices, autoscaling operates at the individual service level, marking a departure from the application-wide approach. We implemented scaling logic using Docker Swarm and Kubernetes.

Deployment manifests were meticulously crafted to define scaling policies grounded in CPU and memory usage:

```yaml
# docker-compose.yml

services:
  orders:
    image: orders
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
      restart_policy: any
      placement:
        constraints: [node.role == worker]  
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      scaler:
        min_replicas: 2    
        max_replicas: 6
```

These scaling tiers provided a reservoir of capacity and adeptly managed elastic overflow. The orders service autonomously monitored its performance and spawned or terminated containers as needed:

```js
// orders.js

const cpusUsed = process.cpuUsage();

if(cpusUsed > 80) {
  swarm.scaleService('orders', 1);
}

if(cpusUsed < 20) {
  swarm.scaleService('orders', -1);  
}
```

Load balancers such as Nginx and Traefik meticulously directed traffic to scaled replica sets. This optimization not only enhanced resource utilization but also bolstered throughput while concurrently reducing operational costs.

### Implementing Resilience

Our arsenal of resiliency techniques included retry policies, timeouts, and circuit breakers. Rate limiting and throttling acted as safeguards against cascading failures, while the Platform service shouldered the responsibility of transient error policies for dependent services.

Homegrown and open-source solutions, including Polly, Hystrix, and Resilience4j, stood as stalwart guardians against potential mishaps. Centralized logging through Elasticsearch proved instrumental in tracing errors across the vast landscape of distributed applications.

### Ensuring Reliability

Incorporating reliability into microservices demanded the implementation of a spectrum of techniques designed to stave off single points of failure. We prioritized automated responses to transient errors and designed measures to tackle scenarios of overload.

Leveraging the Resilience4J library, we implemented circuit breakers to gracefully manage faults:

```java
// OrdersService.java

@Slf4j
@Service
public class OrdersService {

  @CircuitBreaker(name="ordersService", fallbackMethod="fallback")
  public List<Order> getOrders() {
    return orderRepository.getOrders();
  }

  public List<Order> fallback(Exception e) {
    log.error("Circuit open, returning empty orders");
    return Collections.emptyList();
  }
}
```

Rate limiting was introduced to mitigate the risk of service flooding during periods of heightened stress:

```java 
// RateLimiterConfig.java

@Configuration
public class RateLimiterConfig {

  @Bean
  public RateLimiter ordersLimiter() {
    return RateLimiter.create(maxOrdersPerSecond); 
  }

}
```

Timeouts were meticulously enforced to curtail prolonged calls:

```java
// ProductClient.java

@HystrixCommand(fallbackMethod="getProductFallback",
               commandProperties={
                 @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="1000")  
               })
public Product getProduct(Long id) {
  return productService.getProduct(id);
}
```

We introduced retry logic through policies that were finely defined at the client level:

```java
// RetryConfig.java

@Configuration
public class RetryConfig {

  @Bean
  public RetryTemplate ordersRetryTemplate() {
    SimpleRetryPolicy policy = new SimpleRetryPolicy();
    policy.setMaxAttempts(3);
    return new RetryTemplate(policy);
  }

} 
```

These techniques guaranteed uniform responses and put a halt to cascading failures across the expanse of services.

## In Conclusion

The migration of our monolithic ERP system into the realm of microservices has emerged as an enlightening odyssey. It transcends the boundaries of a mere technical transition and signifies an organizational metamorphosis that equips our client to cater to the ever-evolving needs of their customer base with unmatched agility.

Through the disintegration of tightly interconnected layers and the establishment of well-defined domain boundaries, our development team has unlocked a new realm of agility. Features can now be sculpted and deployed independently, driven solely by business priorities rather than architectural constraints. This newfound capacity for swift experimentation and refinement positions the application to remain perpetually attuned to the dynamic demands of the market.

Simultaneously, our operations team now enjoys unfettered visibility and control over each constituent of the system. Anomalous behaviors are detected promptly through enhanced monitoring of individual services. The deployment of scaling and failover mechanisms has shifted from manual endeavors to automated orchestration, fostering an elevated level of resilience that will serve as the bedrock for our client's sustained expansion.

While the merits of a microservices architecture are beyond dispute, embarking on such a migration is not without its share of challenges. Our painstaking efforts in meticulous relationship analysis, interface definition, and the introduction of abstraction, rather than resorting to a brute 'rip and replace' approach, have bestowed upon us a flexible architectural framework capable of evolving harmoniously with the evolving needs of our clientele.

Above all, I remain profoundly grateful for the opportunity this project has afforded us to collaborate closely with our client on their digital transformation journey. The act of demystifying and sharing our experiences and insights in this article is my way of paying it forward, with the hope that it empowers more businesses to embrace modernization. The rewards, both for customers and businesses alike, are unquestionably worth the endeavor.

## References 

- MongoDB Documentation - Official documentation covering data modeling, queries, deployment, and more. [https://docs.mongodb.com/](https://docs.mongodb.com/)

- Microservices Pattern - Martin Fowler's seminal overview of microservices architecture principles. [https://martinfowler.com/articles/microservices.html](https://martinfowler.com/articles/microservices.html)

- Domain-Driven Design - Eric Evans' book introducing DDD concepts for structuring services around business domains. [https://domainlanguage.com/ddd/](https://domainlanguage.com/ddd/)

- Twelve-Factor App Methodology - Best practices for building software-as-a-service apps that are readily deployable to the cloud. [https://12factor.net/](https://12factor.net/)

- Container Journal - In-depth articles on containerization, orchestration, and cloud platforms. [https://containerjournal.com/](https://containerjournal.com/)

- Docker Documentation - Comprehensive guides for building, deploying, and managing containerized apps. [https://docs.docker.com/](https://docs.docker.com/)

- Kubernetes Documentation - Kubernetes, the leading container orchestrator, and its architecture. [https://kubernetes.io/docs/home/](https://kubernetes.io/docs/home/)

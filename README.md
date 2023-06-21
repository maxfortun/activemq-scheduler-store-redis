### Premise
[ActiveMQ Scheduler](https://activemq.apache.org/delay-and-schedule-message-delivery) persistence comes with a [KahaDb](https://activemq.apache.org/kahadb) implementation, which mostly works great, but it does require a shared file system for the [HA](https://activemq.apache.org/shared-file-system-master-slave) architectures. Shared file systems are fanous for generally being slow. If we add a DR scenario, where we have a primary activemq cluster in one region and a secondary activemq cluster in another region - file level replication, like rsync, of KahaDb files between such regions just does not work, and the secondary broker ends up with an inconsistent state.  

If only we could move the scheduler persistence outside of the broker process and into something that supports replication...

### Redis
[Redis](https://redis.io) is a well known in-memory data store that is rather performant, and supports [persistence](https://redis.io/docs/management/persistence/) and [replication](https://redis.io/topics/replication), which makes it well suited for multi-region architectures. 

This project is a JobSchedulerStore implementation with [Redisson](https://github.com/redisson/redisson).

### Loading from redis
Scheduled messages are loaded from redis when broker becomes active.   

### Storing in redis
Scheduled messages are stored into redis as they are scheduled on the broker.

### Expiration
Stored scheduled messages are set to expire at the time they are scheduled for.  
In addition a filter can be set to drop messages that have already expired.  

#### Url format
`schema`://`address`:`port`.   
`schema` is either `redis` for unencrypted connections or `rediss` for TLS.  
`address` is either an `ip address` or a `hostname`.  
`port` is a numerical port.  

#### Mistmatched hostname in certificates
When redis hostname does not match the one in a certificate the connection will fail. To disable hostname validation set `sslEnableEndpointIdentification` to `false`.  

#### Filtering retrieved messages
When scheduler retrieves stored messages it is possible that they have already expired and may need to be filtered out. To achieve that set `filterFactory` to an instance of `org.apache.activemq.broker.scheduler.redis.RedisJobSchedulerStoreFilterExpiredFactory`.   

#### Broker configuration
```xml
<beans
  xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
  http://activemq.apache.org/schema/core http://activemq.apache.org/schema/core/activemq-core.xsd">

    <bean id="filterFactory" class="org.apache.activemq.broker.scheduler.redis.RedisJobSchedulerStoreFilterExpiredFactory" />

    <broker xmlns="http://activemq.apache.org/schema/core" brokerName="localhost" dataDirectory="${activemq.data}" schedulePeriodForDestinationPurge="10000" schedulerSupport="true">

        <jobSchedulerStore>
            <bean xmlns="http://www.springframework.org/schema/beans" id="jobSchedulerStore" class="org.apache.activemq.broker.scheduler.redis.RedisJobSchedulerStore">
                <property name="redisAddress" value="rediss://redis.hostname.here:6379" />
                <property name="redisPassword" value="redis.password.here" />
                <property name="sslEnableEndpointIdentification" value="false" />
                <property name="filterFactory" ref="filterFactory" />
            </bean>
        </jobSchedulerStore>

        ...

    </broker>

</beans>
```


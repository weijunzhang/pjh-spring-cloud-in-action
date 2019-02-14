# pjh-spring-cloud-in-action
Summary of Spring Microservice in actin wroten by @carnellj

## 3. Spring Cloud Config

Refer to https://github.com/carnellj/spmia-chapter3

### 3.1 Spring Cloud Config Server

1. Create App from https://start.spring.io, select **Config Server** dependency, then check pom.xml.

   ```xml
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-config-server</artifactId>
   </dependency>
   ```

2. Add anotation **@EnableConfigServer** in the Application class.

3. Create **application.yml**  file in confsvr/src/main/resources dir, then configure server.port and location of the configuration data storage (normally git type). 

4. Create **configuration files in the data storage**, the file name should be servicename.yml or servicename-profilename.yml, servicename.yml is just for default profile.

Run it by **mvn spring-boot:run**, then you can verify configuration info by http://confsvr:port/servicename/profilename. Now the Config Server is ready for Config Client to fetch configuration info.

### 3.2 Spring Cloud Config Client

1. Create App from https://start.spring.io, select **Config Client** dependency, then check pom.xml.

   ```xml
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-starter-config</artifactId>
   </dependency>
   ```

2. Create **bootstrap.yml** file in src/main/resources dir, then configure spring.application.name, spring.profiles.active and spring.cloud.config.uri (the value also can be defined by -D while running).

3. Write codes to use the configuration info from Config Server, i.e. by @Value("${prop-name}") or spring.datasource which is auto-configured.

While the Config Client is started the configuration info will be fetched from Config Server automatically. 

But, how to refresh running Config Clients if the configuration info is modified in remote Git storage? There are three approaches as below:

1. just restart all App instances.
2. use Eureka discovery client to find all App instances, and then invoke /refresh endpoint of all instances.
3. PCF have a mechanism to send refresh event to all app instances by RabbitMQ.



## 4. Spring Cloud Discovery

Refer to https://github.com/carnellj/spmia-chapter4

### 4.1 Spring Eureka Server

1. Create App from https://start.spring.io, select **Eureka Server** dependency, then check pom.xml.

   ```xml
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
   </dependency>
   ```

2. Add anotation **@EnableEurekaServer** in the Application class.

3. Create application.yml file in src/main/resources dir, then configure server.port and eureka.client.* properties.

Run it by **mvn spring-boot:run**, then you can verify eureka web console http://eurekasvr:port/eureka.

### 4.2 Spring Eureka DIscovery Client

1. Create App from https://start.spring.io, select **Eureka Discovery** dependency, verify pom.xml.

   ```xml
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
   </dependency>
   ```

2. Create **bootstrap.yml** file in src/main/resources dir, then configure spring.application.name, spring.profiles.active.

3. Create **application.yml** file in src/main/resources dir, then configure server.port and eureka.client.* properties.

Run it by **mvn spring-boot:run**, after about 30 seconds, you can verify the client status in eureka web console http://eurekasvr:port/eureka. You also can get client status by http://eurekasvr:port/eureka/apps/servicename.

### 4.3 How to lookup and invoke service 

There are three approaches to lookup and invoke service, by DiscoveryClient, by RestTemplate, by Feign.

#### By DiscoveryClient

1. Add anotation @EnableDiscoveryClient in Application class 
2. Write codes according to OrganizationDiscoveryClient class in spmia-chapter4's licensing-service.

#### By RestTemplate

1. Register RestTemplate as a bean, with anotation @LoadBalanced.

   ```java
   @LoadBalanced
   @Bean
   public RestTemplate getRestTemplate(){
       return new RestTemplate();
   }
   ```

2. Just use the RestTemplate to invoke service registered in Eureka Server by application name, no need by application physical address. (refer to OrganizationRestTemplateClient class)

#### By Feign

1. While create App from https://start.spring.io, should select **Feign** dependency.

2. Add anotation @EnableFeignClients in Application class

3. Use anotation @FeignClient to define interface like below

   ```java
   @FeignClient("organizationservice")
   public interface OrganizationFeignClient {
       @RequestMapping(
               method= RequestMethod.GET,
               value="/v1/organizations/{organizationId}",
               consumes="application/json")
       Organization getOrganization(@PathVariable("organizationId") String organizationId);
   }
   ```

4. Just use method in the feign interface.



## 5. Spring Cloud Circuit Breaker

Refer to https://github.com/carnellj/spmia-chapter5

### 5.1 How to include Hystrix

1. While create App from https://start.spring.io, should select **Hystrix** dependency, check pom.xml.

   ```xml
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
   </dependency>
   ```

2. Add anotation **@EnableCircuitBreaker** in Application class

3. Use anotation **@HystrixCommand** for method which invoke external service or DB.

   ```java
   @HystrixCommand
   private Organization getOrganization(String organizationId) {
       return organizationRestClient.getOrganization(organizationId);
   }
   ```

### 5.2 How to use Hystrix advance feature

1. use fallbackMethod
2. use sperated thread pool
3. config circuit break and recovery parameters.

```java
    @HystrixCommand(fallbackMethod = "buildFallbackLicenseList",
            threadPoolKey = "licenseByOrgThreadPool",
            threadPoolProperties =
                    {@HystrixProperty(name = "coreSize",value="30"),
                     @HystrixProperty(name="maxQueueSize", value="10")},
            commandProperties={
                     @HystrixProperty(name="circuitBreaker.requestVolumeThreshold", value="10"),
                     @HystrixProperty(name="circuitBreaker.errorThresholdPercentage", value="75"),
                     @HystrixProperty(name="circuitBreaker.sleepWindowInMilliseconds", value="7000"),
                     @HystrixProperty(name="metrics.rollingStats.timeInMilliseconds", value="15000"),
                     @HystrixProperty(name="metrics.rollingStats.numBuckets", value="5")}
    )
    public List<License> getLicensesByOrg(String organizationId){
        logger.debug("LicenseService.getLicensesByOrg  Correlation id: {}", UserContextHolder.getContext().getCorrelationId());
        randomlyRunLong();

        return licenseRepository.findByOrganizationId(organizationId);
    }

    private List<License> buildFallbackLicenseList(String organizationId){
        List<License> fallbackList = new ArrayList<>();
        License license = new License()
                .withId("0000000-00-00000")
                .withOrganizationId( organizationId )
                .withProductName("Sorry no licensing information currently available");

        fallbackList.add(license);
        return fallbackList;
    }
```



## 6. Spring Cloud Zuul

Refer to https://github.com/carnellj/spmia-chapter6

### 6.1 How to implement Zuul server as a API gateway

1. While create App from https://start.spring.io, should select **Zuul** dependency, check pom.xml.

   ```xml
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-starter-netflix-zuul</artifactId>
   </dependency>
   ```

2. Add anotation **@EnableZuulProxy** in Application class, do not use @EnableZuulServer

3. Configure Zuul server as eureka client to communicate with eureka server

   ```yml
   server:
     port: 5555
   eureka:
     instance:
       preferIpAddress: true
     client:
       registerWithEureka: true
       fetchRegistry: true
       serviceUrl:
           defaultZone: http://localhost:8761/eureka/
   ```

### 6.2 How to configure routes in Zuul

There are threee approaches to configure routes in Zuul, as below:

1. All services which registered in Eureka server will be routed automatically, if they have not been ignored        manually. Url is http://zuulsvr:port/prefix/servicename/app-api-mapping-address

2. Configure routes of services (which registered in eureka server) manually in applcation.yml or yml file in config server. In below example, automatic route from eureka server is ignored, after that two service's routes are defined manually.

   ```yaml
   zuul:
     ignored-services: '*'
     prefix:  /api
     routes:
         organizationservice: /organization/**
         licensingservice: /licensing/**
   ```

3. Configure routes to static URL, not to service name which registered in eureka.

   ```yaml
   zuul:
     routes:
       licensestatic:
         path: /licensestatic/**
         url: http://static1:8081
   ```

   or

   ```yaml
   zuul:
     routes:
       licensestatic:
         path: /licensestatic/**
         serviceId: licensestatic
         
   ribbin:
     eureka:
       enabled: false
       
   licensestatic:
     ribbon:
       listOfServers: http://static1:8081,http://static2:8082
   ```

### 6.3 How to check or refresh routes in Zuul

1. qurey routes info by endpoint **GET /routes**
2. refresh routes info by endpoint **POST /refresh**

### 6.4 How to set backend service timeout in Zuul

1. Set Hystrix timeout value in yml file, default value is 1000ms,

   ```yaml
   hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 3000
   hystrix.command.default.<servicename>.isolation.thread.timeoutInMilliseconds: 6000
   ```

2. Set Ribbon timeout value in yml file, default value is 5000ms

   ```yaml
   <servicename>.ribbon.ReadTimout: 6000
   ```

### 6.5 Zuul Filter

Zuul have three types filter as below.

1. **Pre-Filter** for incoming request process or authentication or authorizatio, and so on.
2. **Post-Filter** for outgoing response process, i.e. add header info into response message.
3. **Route-Filter** for control route dynamically between two or more backend service versions.

To implement a filter, you need extend **ZuulFilter** class, and then override four methods, i.e. **filterType(), filterOrder(), shouldFilter(), run()**, refer below examples.

1. **zuulsvr/filters/TrackingFilter** is a pre-filter, which try to add tmx-correlation-id header into the incomming http request if the id is not exist.
2. **zuulsvr/filters/ResponseFilter** is a post-filter, which try to add tmx-correlation-id header into the outgoing http response.
3. **zuulsvr/filters/SpecialRoutesFilter** is a route-filter, which check whether the backend service have test version or not, if yes then try to forward some requests to test version.



## 7. Spring Cloud Security

Refer to https://github.com/carnellj/spmia-chapter7

### 7.1 How to implement OAuth2 Authorization Server

1. While create App from https://start.spring.io, should select **Cloud Security, Cloud OAuth2** dependency, check pom.xml.

   ```xml
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-starter-oauth2</artifactId>
   </dependency>
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-starter-security</artifactId>
   </dependency>
   ```

2. Add anotation **@EnableAuthorizationServer, @EnableResourceServer** in Application class.

3. Implement endpoint **/auth/user**, which will be invoked by protected service to query visiting user role.

   ```java
       @RequestMapping(value = { "/user" }, produces = "application/json")
       public Map<String, Object> user(OAuth2Authentication user) {
           Map<String, Object> userInfo = new HashMap<>();
           userInfo.put("user", user.getUserAuthentication().getPrincipal());
           userInfo.put("authorities", AuthorityUtils.authorityListToSet(user.getUserAuthentication().getAuthorities()));
           return userInfo;
       }
   ```

4. To register client app, need to extend AuthorizationServerConfigurerAdapter class, please refer to OAuth2Config class in authentication-service.

   ```java
   @Configuration
   public class OAuth2Config extends AuthorizationServerConfigurerAdapter {
   
       @Autowired
       private AuthenticationManager authenticationManager;
   
       @Autowired
       private UserDetailsService userDetailsService;
   
       @Override
       public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
           clients.inMemory()
                   .withClient("eagleeye")
                   .secret("thisissecret")
                   .authorizedGrantTypes("refresh_token", "password", "client_credentials")
                   .scopes("webclient", "mobileclient");
       }
   
       @Override
       public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
         endpoints
           .authenticationManager(authenticationManager)
           .userDetailsService(userDetailsService);
       }
   }
   ```

5. To configure user credentials, need to extend WebSecurityConfigurerAdapter class, please refer to WebSecurityConfigurer class in authentication-service.

   ```java
   @Configuration
   public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {
           @Override
       @Bean
       public AuthenticationManager authenticationManagerBean() throws Exception {
           return super.authenticationManagerBean();
       }
   
      @Override
       @Bean
       public UserDetailsService userDetailsServiceBean() throws Exception {
           return super.userDetailsServiceBean();
       }
   
   
       @Override
       protected void configure(AuthenticationManagerBuilder auth) throws Exception {
           auth
                   .inMemoryAuthentication()
                   .withUser("john.carnell").password("password1").roles("USER")
                   .and()
                   .withUser("william.woodward").password("password2").roles("USER", "ADMIN");
       }
   }
   ```

### 7.2 How to test OAuth2 Authorization Server

1. Use postman send POST request to http://oauth2-svr:port/auth/oauth/token , fill client app's credentials as Basic Auth, then fill grant-type, scope, username, password as message body with form-data type, you will get token in response if success.
2. After you get token, send GET request to http://oauth2-svr:port/auth/user, fill 'Bearer' and token value as Basic Auth, then you will get user's authorization info. 

### 7.3 How to implement OAuth2 Resource Server

1. While create App from https://start.spring.io, should select **Cloud Security, Cloud OAuth2** dependency, check pom.xml.

   ```xml
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-starter-oauth2</artifactId>
   </dependency>
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-starter-security</artifactId>
   </dependency>
   ```

2. Add anotation **@EnableResourceServer** in Application class.

3. Configure oauth2 userinfourl in yml file as below

   ```yaml
   security:
     oauth2:
       resource:
          userInfoUri: http://localhost:8901/auth/user
   ```

4. To verify whether user have priority to access resource, need to extend ResourceServerConfigurerAdapter to protect resources.

   ```java
   @Configuration
   public class ResourceServerConfiguration extends ResourceServerConfigurerAdapter {
   
   
       @Override
       public void configure(HttpSecurity http) throws Exception{
           http
           .authorizeRequests()
             .antMatchers(HttpMethod.DELETE, "/v1/organizations/**")
             .hasRole("ADMIN")
             .anyRequest()
             .authenticated();
       }
   }
   ```

### 7.4 How to spread token between services

Need to do two things as below:

1. Config Zuul to permit HTTP header **Authorization** pass to backend service, just remove Authorization from sensitiveHeaders blacklist.

   ```yaml
   zuul.sensitiveHeaders: Cookie, Set-Cookie
   ```

2. Use OAuth2RestTemplate to replace normal RestTemplate, you need create a bean for it at first, and then use it.

   ```java
   	@Bean
       public OAuth2RestTemplate oauth2RestTemplate(OAuth2ClientContext oauth2ClientContext, OAuth2ProtectedResourceDetails details) {
           return new OAuth2RestTemplate(details, oauth2ClientContext);
       }
   ```

### 7.5 How to implement JWT token in Authorization Server

1. add jwt library in pom.xml

   ```xml
   <dependency>
       <groupId>org.springframework.security</groupId>
       <artifactId>spring-security-jwt</artifactId>
   </dependency>
   ```

2. create a configuration class to assemble beans of JwtTokenStore, JwtTokenService, JwtAccessTokenConverter, and JwtTokenEnhancer.

   ```java
       @Bean
       public TokenStore tokenStore() {
           return new JwtTokenStore(jwtAccessTokenConverter());
       }
   
       @Bean
       @Primary
       public DefaultTokenServices tokenServices() {
           DefaultTokenServices defaultTokenServices = new DefaultTokenServices();
           defaultTokenServices.setTokenStore(tokenStore());
           defaultTokenServices.setSupportRefreshToken(true);
           return defaultTokenServices;
       }
   
   
       @Bean
       public JwtAccessTokenConverter jwtAccessTokenConverter() {
           JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
           converter.setSigningKey(serviceConfig.getJwtSigningKey());
           return converter;
       }
   
       @Bean
       public TokenEnhancer jwtTokenEnhancer() {
           return new JWTTokenEnhancer();
       }
   ```

3. (Optional) To customize JWT token with addtional business field, need to implements TokenEnhancer interface, and then override enhance() method. Please refer JWTTokenEnhance class.

4. edit OAuth2Config class to use Jwt beans created in the previous step.

   ```java
       @Override
       public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
           TokenEnhancerChain tokenEnhancerChain = new TokenEnhancerChain();
           tokenEnhancerChain.setTokenEnhancers(Arrays.asList(jwtTokenEnhancer, jwtAccessTokenConverter));
   
           endpoints.tokenStore(tokenStore)                             //JWT
                   .accessTokenConverter(jwtAccessTokenConverter)       //JWT
                   .tokenEnhancer(tokenEnhancerChain)                   //JWT
                   .authenticationManager(authenticationManager)
                   .userDetailsService(userDetailsService);
       }
   ```

### 7.6 How to get customized info from JWT token

1. add jjwt library in pom.xml

   ```xml
   <dependency>
       <groupId>io.jsonwebtoken</groupId>
       <artifactId>jjwt</artifactId>
       <version>0.7.0</version>
   </dependency>
   ```

2. use Claims and Jwts class to get customized data in JWT token, as below example.

   ```java
       private String getOrganizationId(){
   
           String result="";
           if (filterUtils.getAuthToken()!=null){
   
               String authToken = filterUtils.getAuthToken().replace("Bearer ","");
               try {
                   Claims claims = Jwts.parser()
                           .setSigningKey(serviceConfig.getJwtSigningKey().getBytes("UTF-8"))
                           .parseClaimsJws(authToken).getBody();
                   result = (String) claims.get("organizationId");
               }
               catch (Exception e){
                   e.printStackTrace();
               }
           }
           return result;
       }
   
   ```



## 8. Spring Cloud Stream with Kafka

Refer to https://github.com/carnellj/spmia-chapter8

1. While create App from https://start.spring.io, should select **Cloud Stream, Kafka** dependency, check pom.xml.

   ```xml
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-stream</artifactId>
   </dependency>
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-stream-binder-kafka</artifactId>
   </dependency>
   <dependency>
   	<groupId>org.springframework.kafka</groupId>
   	<artifactId>spring-kafka</artifactId>
   </dependency>
   ```

2. define Kafka streams, also can use pre-defined stream Source, Sink and Processor. Why? In order for our application to be able to communicate with Kafka, we'll need to define an outbound stream to write messages to a Kafka topic, and an inbound stream to read messages from a Kafka topic.

   ```java
   public interface GreetingsStreams {
       String INPUT = "greetings-in";
       String OUTPUT = "greetings-out";
       @Input(INPUT)
       SubscribableChannel inboundGreetings();
       @Output(OUTPUT)
       MessageChannel outboundGreetings();
   }
   ```

   Sink / Source / Processor are pre-defined, you can just use it.

   ```java
   public interface Sink {
   
     String INPUT = "input";
   
     @Input(Sink.INPUT)
     SubscribableChannel input();
   }
   ```

   ```java
   public interface Source {
   
     String OUTPUT = "output";
   
     @Output(Source.OUTPUT)
     MessageChannel output();
   }
   ```

   ```java
   public interface Processor extends Source, Sink {}
   ```

3. bind Kafka streams to Spring Cloud Stream.

   ```java
   @EnableBinding(GreetingsStreams.class)
   public class StreamsConfig {
   }
   ```

4. Config properties for kafka in yml file

   ```yaml
   spring:
     cloud:
       stream:
         kafka:
           binder:
             brokers: localhost:9092
             zkNodes: localhost:2181
         bindings:
           greetings-in:
             destination: greetings
             contentType: application/json
             group: greetingGroup
           greetings-out:
             destination: greetings
             contentType: application/json
   ```

5. send message to kafka

   ```java
   public void sendGreeting(final Greetings greetings) {
       log.info("Sending greetings {}", greetings);
       MessageChannel messageChannel = greetingsStreams.outboundGreetings();
       messageChannel.send(MessageBuilder
           .withPayload(greetings)
           .setHeader(MessageHeaders.CONTENT_TYPE,	MimeTypeUtils.APPLICATION_JSON)
           .build());
   }
   ```

6. listening on kafka topic

   ```java
   @Component
   public class GreetingsListener {
       @StreamListener(GreetingsStreams.INPUT)
       public void handleGreetings(@Payload Greetings greetings) {
           log.info("Received greetings: {}", greetings);
       }
   }
   ```



## 9. Spring Cloud Sleuth and Zipkin

Refer to https://github.com/carnellj/spmia-chapter9

### 9.1 Spring Cloud Sleuth

Spring Cloud Sleuth provide below features for distributed tracing.

1. Add correlation-id into HTTP request header and cloud stream message.
2. Add correlation-id and span-id info into log.

how to include Sleuth into a service, you just need to add dependency in pom.xml.

```xml
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-sleuth</artifactId>
	</dependency>
```

### 9.2 Zipkin

Zipkin is a distributed tracing platform for distributed transaction among multi services. Use it to trace each span's respond time by Web UI.

How to include Zipkin client into a service, you need to:

1. add dependency in pom.xml

   ```xml
   	<dependency>
   		<groupId>org.springframework.cloud</groupId>
   		<artifactId>spring-cloud-starter-zipkin</artifactId>
   	</dependency>
   ```

2. add zipkin configuration in yml file

   ```yaml
   spring:
     sleuth:
       sampler:
         percentage:  1
     zipkin:
       baseUrl:  localhost:9411
   ```

### 9.3 Setup Zipkin Server

1. add dependency in pom.xml

   ```xml
     <dependencies>
         <dependency>
             <groupId>io.zipkin.java</groupId>
             <artifactId>zipkin-autoconfigure-ui</artifactId>
         </dependency>
         <dependency>
             <groupId>io.zipkin.java</groupId>
             <artifactId>zipkin-server</artifactId>
         </dependency>
         
         <!-- mysql or h2 for data storage -->
         <dependency>
             <groupId>io.zipkin.java</groupId>
             <artifactId>zipkin-autoconfigure-storage-mysql</artifactId>
         </dependency>
         <dependency>
             <groupId>com.h2database</groupId>
             <artifactId>h2</artifactId>
         </dependency>
         <dependency>
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-jdbc</artifactId>
         </dependency>
     </dependencies>
   
   ```

2. Add anotation **@EnableZipkinServer** in Application class.

3. Config server.port in yml file, config data storage if necessory

Now you can run it and visit web portal to trace HTTP message and Cloud stream message.

You also can add user-defined span, for example, DB operation. as below.

```java
	public Organization getOrg
            (String organizationId) {
        Span newSpan = tracer.createSpan("getOrgDBCall");

        logger.debug("In the organizationService.getOrg() call");
        try {
            return orgRepository.findById(organizationId);
        }
        finally{
          newSpan.tag("peer.service", "postgres");
          newSpan.logEvent(org.springframework.cloud.sleuth.Span.CLIENT_RECV);
          tracer.close(newSpan);
        }
    }
```



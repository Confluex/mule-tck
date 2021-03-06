# Mule Testing Extensions

## Setting up using maven
### Mule 3.7.x
```xml
<dependency>
    <groupId>com.confluex</groupId>
    <artifactId>mule-testing-kit</artifactId>
    <version>3.7.0.1</version>
</dependency>
```

### Mule 3.6.x
```xml
<dependency>
    <groupId>com.confluex</groupId>
    <artifactId>mule-testing-kit</artifactId>
    <version>3.6.0.0</version>
</dependency>
```

### Mule 3.5.x
```xml
<dependency>
    <groupId>com.confluex</groupId>
    <artifactId>mule-testing-kit</artifactId>
    <version>1.0.0</version>
</dependency>
```

# Using BetterFunctionalTestCase
All of the features of mule-testing-kit are provided through inheritance.  You need to extend 
`BetterFunctionalTestCase` instead of `FunctionalTestCase` to use them.

## Making sure things are finished

One of the reasons we use Mule for integration is that it handles threading for us.  This can make testing flows a
little more complicated, since our junit test thread isn't the one doing the work.  These tools allow you to block the
thread running your FunctionalTestCase test until certain things happen in the Mule server.

**BlockingEndpointListener**

This event listener will attach to an endpoint and allow your test to wait and collect the messages processed by
the endpoint. The following example configuration and test case demonstrate basic usage:

_Mule Configuration_


```xml
<mule>
    <notifications dynamic="true">
        <notification event="ENDPOINT-MESSAGE" />
    </notifications>


    <vm:endpoint name="in" path="in" />
    <vm:endpoint name="out" path="out" />
    <http:endpoint name="slowRestService"
                   address="http://services.slowbusinesspartner.com/important-resource" />

    <flow name="my-flow">
        <inbound-endpoint ref="in"/>
        <outbound-endpoint ref="slowRestService" exchange-pattern="request-response" />
        <outbound-endpoint ref="out"/>
    </flow>
</mule>
```

Notice that the endpoints are declared globally, and given names.  We need to initialize our BlockingEndpointListener
using the name of the endpoint we expect our messages to be sent to:

```java
public class MyFlowFunctionalTest extends BetterFunctionalTestCase {

    @Test
    public void shouldOutputResultFromSlowBusinessPartner() {
        BlockingEndpointListener listener = listenForEndpoint("out");

        muleContext.getClient().dispatch("in", "source data", new HashMap<String, Object>());
        // MuleClient.dispatch() returns immediately, while the flow is still processing the message.
        // We need to wait for the outbound-endpoint "out" to receive the message before we can
        // make assertions about it.

        assertTrue("No message received on outbound endpoint", listener.waitForMessages());
        // BlockingEndpointListener.waitForMessages() returns true only if a message is received
        // before the timeout. The default timeout is ten seconds.

        MuleMessage result = listener.getMessages().get(0);
        // the BlockingEndpointListener captures all messages sent to that endpoint.
        // We can now make assertions about that message

        // ...
    }
}
```

When you expect a certain number of messages to be sent before you are ready to make assertions, you can set up the
listener to wait for multiple messages:

```java

    BlockingEndpointListener listener = listenForEndpoint("updateOrderJmsEndpoint", 16);
```

Subsequent calls to `waitForMessages()` will block until 16 messages are received.

You can also control the amount of time the listener will wait before giving up and returning false:

```java
    listener.waitForMessages(500); // this waits for 500ms instead of the default 10s.
```

**BlockingMessageProcessorListener**

Sometimes you need to wait for a MessageProcessor to handle a message before your test method continues.  For example,
many Cloud Connectors provide their functionality as a MessageProcessor, not an endpoint.  For these cases, we provide
`BlockingMessageProcessorListener`, which works pretty much like the `BlockingEndpointListener`.  You do need to remember
to enable dynamic notification for `MESSAGE-PROCESSOR` events:

 ```xml
 <mule>
     <notifications dynamic="true">
         <notification event="ENDPOINT-MESSAGE" />
         <notification event="MESSAGE-PROCESSOR" />
     </notifications>
 </mule>
 ```
 
 ```java
 BlockingMessageProcessorListener listener = listenForMessageProcessor("myMongoOperation", 2000);
 ```

## Setting up and cleaning up

Normally we can use the `@Before` and `@After` annotations provided by junit to set up before each test method, and clean
up afterwards.  When we use these annotations with Mule's `FunctionalTestCase`, the platform makes sure that Mule is
initialized before it runs our `@Before` methods, and it makes sure our `@After` methods run before Mule stops.

Sometimes, however, we want to do things before Mule starts or after Mule stops.  For these scenarios, we extend Mule's
FunctionalTestCase and provide two new annotations: `@BeforeMule` and `@AfterMule`.  

_@BeforeMule_
Methods annotated with `@BeforeMule` will be run after the configuration files are processed and the Spring context is
initialized, but before Mule starts.  These methods can take one optional parameter of type `MuleContext`, if you need
access to it.  This is because `this.muleContext` is not yet initialized when these methods run.

_@AfterMule_
Methods annotated with `@AfterMule` will be run after Mule stops, and after the Spring context is disposed.  This can be
useful for making sure that all connections have been closed before your `@AfterMule` method runs.
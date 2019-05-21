---

title: 'Message Events'
category: 'Events'

keywords: 'message start intermediate catching boundary intermediate throwing end event definition messages'

---

Message events are events which reference a named message. A message has a name and a payload. Unlike a signal, a message event is always directed at a single receiver.

A message event definition is declared using the messageEventDefinition element. The attribute messageRef references a message element declared as a child element of the definitions root element. The following is an excerpt of a process where two message events is declared and referenced by a start event and an intermediate catching message event.

```xml
<definitions id="definitions"
  xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
  xmlns:camunda="http://activiti.org/bpmn"
  targetNamespace="Examples"
  xmlns:tns="Examples">

  <message id="newInvoice" name="newInvoiceMessage" />
  <message id="payment" name="paymentMessage" />

  <process id="invoiceProcess">

    <startEvent id="messageStart" >
        <messageEventDefinition messageRef="newInvoice" />
    </startEvent>
    ...
    <intermediateCatchEvent id="paymentEvt" >
        <messageEventDefinition messageRef="payment" />
    </intermediateCatchEvent>
    ...
  </process>
</definitions>
```

## Triggering Message Events

As an embeddable process engine, the camunda engine is not concerned with the receiving part of the message. This would be environment dependent and entail platform-specific activities like connecting to a JMS (Java Messaging Service) Queue/Topic or processing a Webservice or REST request. The reception of messages is therefore something you have to implement as part of the application or infrastructure into which the process engine is embedded.

After you have received a message, you can choose whether you employ the engine's in-built correlation or explicitly deliver the message to start a new process instance or trigger a waiting execution.



### Using the runtime service\'s correlation methods

The engine offers a basic correlation mechanism that will either signal an execution waiting for a specific message or instantiate a process with a matching message start event. You can choose from these methods in the runtime service:

```java
void correlateMessage(String messageName);
void correlateMessage(String messageName, String businessKey);
void correlateMessage(String messageName, Map<String, Object> correlationKeys);
void correlateMessage(String messageName, String businessKey, Map<String, Object> processVariables);
void correlateMessage(String messageName, Map<String, Object> correlationKeys, Map<String, Object> processVariables);
void correlateMessage(String messageName, String businessKey, Map<String, Object> correlationKeys, Map<String, Object> processVariables);
```

The `messageName` identifies the message as defined in the message name attribute in the process definition xml.

Correlation is successful, if there exists a single matching entity among the following:

* __Process Definition__: A process definition matches, if it can be started by a message named `messageName`.
* __Execution (Process Instance)__: An execution matches, if it is waiting for a message named `messageName` and its process instance matches the given `businessKey` and `correlationKeys` (if provided). The `correlationKeys` map is matched against the process instance variables.


<div class="alert alert-warning">
  <strong>Current limitation:</strong>

  <code>correlationKeys</code> is matched against process instance variables only. These are variables that are globally visible throughout the process instance.

  Accordingly, variables that are defined in the scope of a child execution (e.g. in a subprocess) are not considered for correlation.
</div>

In case of successful correlation, the correlated or newly created process instance is updated with the variables from the `processVariables` map.


### Explicitly delivering a message

Alternatively, you can explicitly deliver a message to start a process instance or trigger a waiting execution.

If the message should trigger the start of a new process instance, choose between the following methods offered by the runtime service:

```java
ProcessInstance startProcessInstanceByMessage(String messageName);
ProcessInstance startProcessInstanceByMessage(String messageName, Map<String, Object> processVariables);
ProcessInstance startProcessInstanceByMessage(String messageName, String businessKey, Map<String, Object> processVariables);
```

These methods allow starting a process instance using the referenced message.

If the message needs to be received by an existing process instance, you first have to correlate the message to a specific process instance (see next section) and then trigger the continuation of the waiting execution. The runtime service offers the following methods for triggering an execution based on a message event subscription:

```java
void messageEventReceived(String messageName, String executionId);
void messageEventReceived(String messageName, String executionId, HashMap<String, Object> processVariables);
```



### Querying for Message Event subscriptions

The engine supports message start events and intermediate message events.

In the case of a message start event, the message event subscription is associated with a particular process definition. Such message subscriptions can be queried using a ProcessDefinitionQuery:

```java
ProcessDefinition processDefinition = repositoryService.createProcessDefinitionQuery()
        .messageEventSubscription("newCallCenterBooking")
        .singleResult();
```

Since there can only be one process definition for a specific message subscription, the query always returns zero or one results. If a process definition is updated, only the newest version of the process definition has a subscription to the message event.

In the case of an intermediate catch message event, the message event subscription is associated with a particular execution. Such message event subscriptions can be queried using a ExecutionQuery:

```java
Execution execution = runtimeService.createExecutionQuery()
      .messageEventSubscriptionName("paymentReceived")
      .processVariableValueEquals("orderId", message.getOrderId())
      .singleResult();
```

Such queries are called correlation queries and usually require knowledge about the processes (in this case that there will be at most one process instance for a given orderId).



## Receiving Message Events

### Message Start Event

A message start event can be used to start a process instance using a named message. This effectively allows us to select the right start event from a set of alternative start events using the message name.

When deploying a process definition with one or more message start events, the following considerations apply:

* The name of the message start event must be unique across a given process definition. A process definition must not have multiple message start events with the same name. The engine throws an exception upon deployment of a process definition such that two or more message start events reference the same message of if two or more message start events reference messages with the same message name.
* The name of the message start event must be unique across all deployed process definitions. The engine throws an exception upon deployment of a process definition such that one or more message start events reference a message with the same name as a message start event already deployed by a different process definition.
* Process versioning: Upon deployment of a new version of a process definition, the message subscriptions of the previous version are canceled. This is also true for message events that are not present in the new version.

When starting a process instance, a message start event can be triggered using the following methods on the RuntimeService:

```java
ProcessInstance startProcessInstanceByMessage(String messageName);
ProcessInstance startProcessInstanceByMessage(String messageName, Map<String, Object> processVariables);
ProcessInstance startProcessInstanceByMessage(String messageName, String businessKey, Map<String, Object< processVariables);
```

The messageName is the name given in the name attribute of the message element referenced by the messageRef attribute of the messageEventDefinition. The following considerations apply when starting a process instance:

* Message start events are only supported on top-level processes. Message start events are not supported on embedded sub processes.
* If a process definition has multiple message start events, `runtimeService.startProcessInstanceByMessage(...)` allows to select the appropriate start event.
* If a process definition has multiple message start events and a single none start event, `runtimeService.startProcessInstanceByKey(...)` and `runtimeService.startProcessInstanceById(...)` starts a process instance using the none start event.
* If a process definition has multiple message start events and no none start event, `runtimeService.startProcessInstanceByKey(...)` and `runtimeService.startProcessInstanceById(...)` throw an exception.
* If a process definition has a single message start event, `runtimeService.startProcessInstanceByKey(...)` and `runtimeService.startProcessInstanceById(...)` start a new process instance using the message start event.
* If a process is started from a call activity, message start event(s) are only supported if
    * in addition to the message start event(s), the process has a single none start event
    * the process has a single message start event and no other start events.

The XML representation of a message start event is the normal start event declaration with a messageEventDefinition child-element:

```xml
<definitions id="definitions"
  xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
  xmlns:camunda="http://activiti.org/bpmn"
  targetNamespace="Examples"
  xmlns:tns="Examples">

  <message id="newInvoice" name="newInvoiceMessage" />

  <process id="invoiceProcess">

    <startEvent id="messageStart" >
        <messageEventDefinition messageRef="tns:newInvoice" />
    </startEvent>
    ...
  </process>

</definitions>
```

A process can be started using one of two different messages, this is useful if the process needs alternative ways to react to different start events but eventually continues in a uniform way.

<div data-bpmn-diagram="implement/event-message-start-alternative" > </div>


### Message Intermediate Catching Event


When a token arrives at the message intermediate catching event it will wait there until a message with the proper name arrives. As already described the message must be handed into the engine via the appropriate API calls.

The following example shows different message events in a process model:

<div data-bpmn-diagram="implement/event-message"></div>

```xml
<intermediateCatchEvent id="message">
        <messageEventDefinition signalRef="newCustomerMessage" />
</intermediateCatchEvent>
```

Instead of the message intermediate catching event you might want to think about a <a href="ref:#tasks-receive-task">Receive Task</a> instead which can serve similar purposes, but is able to be used in combination with boundary events. Together with the  message intermediate catching event you might want to use the <a href="ref:#gateways-event-based-gateway">Event-based Gateway</a>.


### Message Boundary Event

Boundary events are catching events that are attached to an activity. This means that while the activity is running, the message boundary event is listening for named message. When this is caught, two things might happen depending on the configuration of the boundary event:

* Interrupting boundary event: The activity is interrupted and the sequence flow going out of the event is followed.
* Non-interrupting boundary event: One token stays in the activity and an additional token is created which follows the sequence flow going out of the event.


## Sending Message Events



### Message Intermediate Throwing Event

Message intermediate throwing event sends a message to an external service. This event has the same behavior as a [service task](ref:#tasks-service-task).

<div data-bpmn-diagram="implement/event-message-throwing" > </div>

```xml
<intermediateThrowEvent id="message">
  <messageEventDefinition camunda:class="org.camunda.bpm.MyMessageServiceDelegate" />
</intermediateThrowEvent>
```


## Message End Event

When process execution arrives in a message end event, the current path of execution is ended and a message is sent. The message end event has the same behavior as a [service task](ref:#tasks-service-task).

```xml
<endEvent id="end">
  <messageEventDefinition camunda:class="org.camunda.bpm.MyMessageServiceDelegate" />
</endEvent>
```
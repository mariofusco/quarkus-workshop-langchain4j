# Step 1 - Implementing AI Agents

## A new challenge

The Miles of Smiles management team needs help with managing their cars. 

When customers return cars they had rented, the team processing the return should be able to record comments about any issues they notice with the car. The car should then be sent for cleaning. The car wash team will pay attention to the comments and clean the car accordingly. When the cleaning is complete, the car wash team will provide their own comments and return the car. If the car is returned with no issues it can be put back into the available pool to be rented.

## Running the application

Run the application with the following command:

```bash
./mvnw quarkus:dev
```

This will bring up the page at [http://localhost:8080](http://localhost:8080){target="_blank"}.

The UI has two sections. The **Fleet Status** section shows all the cars in the Miles of Smiles fleet. The **Returns** section shows cars that are either rented or at the car wash.

![Agentic App UI](../images/agentic-UI-1.png){: .center}

Acting as one of the Miles of Smiles team members accepting car rental returns, fill in a comment for one of the cars in the Rental Return section and click the corresponding Return button. 

```
Car has dog hair all over the back seat
```

After a few moments the car status will be updated in the fleet status section and the car should no longer appear in the returns section. With the above return comment, the log output would show evidence of a car wash request being made similar to the following:

```
CarWashTool result: Car wash requested for Mercedes-Benz C-Class (2020), Car #6:
- Interior cleaning
Additional notes: Interior cleaning required due to dog hair in back seat.
```

Try again, this time with a comment that indicates the car is clean:

```
Car looks good
```

In the logs you should see a response indicating a car wash is not required:

TODO: replace with OpenAI log
```
- status code: 200
- headers: [content-length: 442], [content-type: application/json; charset=utf-8], [date: Mon, 08 Sep 2025 18:58:10 GMT]
- body: {"model":"gpt-oss:20b","created_at":"2025-09-08T18:58:10.119563Z","message":{"role":"assistant","content":"CARWASH_NOT_REQUIRED","thinking":"We need to decide if car wash needed. Feedback says car looks good. So no wash. Output \"CARWASH_NOT_REQUIRED\"."},"done_reason":"stop","done":true,"total_duration":1307237250,"load_duration":132135042,"prompt_eval_count":284,"prompt_eval_duration":443868833,"eval_count":42,"eval_duration":729291917}
```

## Building Agents with LangChain4j

The [langchain4j-agentic](https://github.com/langchain4j/langchain4j/tree/main/langchain4j-agentic){target="_blank"} module introduces the ability to create Agents. In their simplest form, agents are very similar to AI Services (introduced earlier):

- Agents are declared in interfaces (and are implemented for you automatically)
- Agent interfaces let you specify a `SystemMessage` and `UserMessage`
- Agents can be assigned ==tools== which they can use
- Agents can be defined programmatically or declaratively (with annotations).

In contrast to AI Services, only one method on an agent interface can be annotated with `@Agent`. This method is the method callers will call to invoke the agent.

## Understanding the app

![App Blueprint](../images/agentic-app-1.png){: .center}


```java title="CarManagementResource.java"
--8<-- "../../section-2/step-01/src/main/java/com/carmanagement/resource/CarManagementResource.java:car-management"
```

The `CarManagementResource` provides REST APIs to handle returns of cars from the rental team and the car wash team.

```java title="CarManagementService.java"
--8<-- "../../section-2/step-01/src/main/java/com/carmanagement/service/CarManagementService.java:createCarWashAgent"
```

The `CarManagementResource` calls the `CarManagementService` to handle car returns. The `CarManagementService` uses the `CarWashAgent` to request car washes. The `CarManagementService` sets up the agent, which entails:

- defining the chat model it should use
- indicating the output name in the `AgenticScope` to use to hold the result from the call to the agent (more will be said about the `AgenticScope` in the next step)

```java title="CarWashAgent.java"
--8<-- "../../section-2/step-01/src/main/java/com/carmanagement/agentic/agents/CarWashAgent.java:carWashAgent"
```

The `CarWashAgent` looks at the comments from when the car was returned and decides which car wash options to select.

- `@SystemMessage` is used to tell the agent its role and how to handle requests.
- `@UserMessage` is used to provide content specific to the request.
- You don't provide the implementation (that is created for you by LangChain4j)
- `@Agent` annotation identifies the method in the interface to use as the agent. Only one method can have the `@Agent` annotation per interface.

```java title="CarWashTool.java"
--8<-- "../../section-2/step-01/src/main/java/com/carmanagement/agentic/tools/CarWashTool.java:CarWashTool"
```

The `CarWashTool` is a mock tool for requesting the car wash. The `@Tool` annotation is used to identify each method that should be registered as a tool that agents can use.

??? question "Why do we use @Dependent scope for the Tool?"
    When a tool is added to the definition of an agent, LangChain4j introspects the tool object to see which methods have `@Tool` annotations. CDI creates proxies around objects that are defined with certain CDI scopes (such as `@ApplicationScoped` or `@SessionScoped`). The proxy methods do not have the `@Tool` annotations and therefore the agents don't properly recognize the tool methods on those objects. If you need your tools to be defined with other CDI scopes, you can use a `ToolProvider` to add tools (not discussed in this tutorial).

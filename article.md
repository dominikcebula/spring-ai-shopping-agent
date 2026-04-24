# AI Shopping Agent using Spring AI

![logo.png](docs/logo.png)

## Introduction

In this article, I will describe how I built a conversational AI shopping agent using Spring AI, and how I extended it
with **persistent, per-user chat history** and **long-term memory (LTM)** so that it can remember user preferences
across sessions.

The end result is a conversational shopping assistant that helps users search a product catalog, compare products, and
place orders through natural language interactions. The agent retains recent conversation context for coherent
multi-turn dialogues, and it remembers important preferences and facts about the user even across separate
conversations.

The agent is accessible through a web-based chat interface built with React. It looks like this:

![agent-chat-ui.png](docs/agent-chat-ui.png)

### Example Interaction

The agent helps users find the right products and create orders using prompts such as:

```text
I would like to buy a budget laptop for daily usage with at least 8GB RAM and 512 GB of storage.
Please also include a monitor, mouse and a keyboard.
Select products that match my criteria and create the order.
```

The agent processes the user's request using an LLM and MCP tools, and reports the created order back:

```text
Your order has been created successfully! Here are the details:

- **CoreBook 14 Everyday Laptop** - Price: $699.99
- **Wireless Gaming Mouse** - Price: $59.99
- **Mechanical Gaming Keyboard** - Price: $129.99
- **27-inch 4K Monitor** - Price: $349.99

**Order ID:** 9f992c3d-2af9-4de4-b8e2-1f939237866f

**Total Value:** $1239.96
```

The full source code is available on GitHub:

https://github.com/dominikcebula/spring-ai-shopping-agent

## Architecture

The diagram below illustrates the overall architecture of the solution.

![architecture.drawio.png](docs/architecture.drawio.png)

The project is divided into two business domains:

- **Products** — browsing and searching the product catalog
- **Orders** — creating, updating, and cancelling orders on the user's behalf

Each domain consists of an MCP Server, a set of MCP Tools, a REST API, and a Microservice that implements the API.

Within each domain, it is possible to perform search and management operations either via a REST API or through the MCP
protocol, which is well suited for AI agent integrations.

The agent itself uses **MongoDB** for two purposes:

- persisting chat history (short-term memory)
- storing long-term memory entries as vector embeddings (using MongoDB Atlas as a vector database)

For simplicity, all catalog and order data is kept in-memory in the domain microservices — no real database is used for
the business domains themselves.

## Notes on the Solution Design

There are multiple ways this problem could be decomposed and implemented.

Instead of deploying a dedicated MCP Server per domain, the MCP Servers could be combined into a single microservice
exposing both REST APIs and the MCP protocol.

Another option would be to implement MCP Tools directly on the agent side, allowing the agent to call microservices via
REST without an intermediate MCP Server layer.

Alternatively, the entire solution could be implemented as a modular monolith using Spring Modulith, with each domain
represented as a separate module.

Different approaches may be preferable depending on project requirements, team preferences, and operational constraints.

The current approach prioritizes decoupling, independent deployments, and fine-grained scalability, at the cost of
increased complexity, network overhead, and latency.

## Tech Stack

The following technologies are used in this solution:

| Layer    | Technology                                     |
|----------|------------------------------------------------|
| AI / LLM | Spring AI 1.1.2                                |
| Backend  | Java 25, Spring Boot 3.5.12, Spring AI MCP     |
| Frontend | React 19, TypeScript                           |
| Protocol | Model Context Protocol (MCP) - Streamable HTTP |
| Storage  | MongoDB Atlas (chat history + vector store)    |
| Build    | Maven (multi-module), npm                      |

## Implementation

### Agent

The agent is built using Spring AI and uses the **Claude Opus 4.5** model hosted on Amazon Bedrock.

It accesses MCP tools for products and orders via an MCP client. For each domain, an MCP server communicates with a
backing microservice over a REST API to execute the business logic for search and order operations.

To constrain the agent's interactions to those of a "helpful shopping assistant" that supports product search and
ordering, a system prompt is used to guide and control the agent's behavior.

The agent code looks like this:

```java

@RestController
@RequestMapping("/api/v1")
public class AgentController {
    private final ChatClient chatClient;

    public AgentController(ChatClient.Builder chatClientBuilder,
                           ToolCallbackProvider toolCallbackProvider,
                           ChatMemory chatMemory,
                           MemoryRecorderAdvisor memoryRecorderAdvisor,
                           MemoryRetrievalAdvisor memoryRetrievalAdvisor) {
        this.chatClient = chatClientBuilder
                .defaultToolCallbacks(toolCallbackProvider)
                .defaultAdvisors(
                        MessageChatMemoryAdvisor.builder(chatMemory).build(),
                        memoryRecorderAdvisor,
                        memoryRetrievalAdvisor
                )
                .defaultSystem(
                        """
                                You are a helpful shopping assistant who can help users find products in the catalog and place orders on their behalf.
                                Your primary responsibility is to help users search for, compare, and order products efficiently and accurately.
                                
                                Use provided Products Tools and Orders Tools to assist the user with their shopping needs.
                                Always use the tools available to get information and perform actions on behalf of the user.
                                
                                When creating an order, each order item must include productId, productName, quantity, and unitPrice.
                                Take the productName and unitPrice directly from the product catalog at the moment of ordering (snapshot the current price).
                                
                                Be professional, concise, and friendly.
                                Use clear, structured responses that are easy to scan.
                                Avoid unnecessary verbosity while ensuring all critical order information is communicated.
                                Your goal is to act as a reliable, tool-driven shopping assistant that helps users find the right products and place orders with confidence and clarity.
                                
                                You have access to the following types of memory:
                                1. Short-term memory: Chat history, the current conversation thread
                                2. Long-term memory:
                                   A. EPISODIC: Personal experiences and user-specific preferences
                                      Examples: "User prefers budget laptops", "User prefers wireless peripherals"
                                   B. SEMANTIC: General domain knowledge and facts
                                      Examples: "User is setting up a home office", "User is a gamer"
                                
                                If the user asks for information that is not related to shopping or placing orders, respond politely that you can only assist with shopping and orders.
                                """)
                .build();
    }

    @GetMapping("/agent")
    public String generation(@RequestParam String userInput, @RequestParam UUID conversationId) {
        return chatClient.prompt()
                .user(userInput)
                .advisors(advisorSpec -> advisorSpec.param(CONVERSATION_ID, conversationId))
                .call()
                .content();
    }
}
```

The agent accesses MCP tools and configures both chat memory storage and vector storage in `application.yml`:

```yaml
spring:
  ai:
    bedrock:
      aws:
        region: eu-central-1
      converse:
        chat:
          options:
            model: eu.anthropic.claude-opus-4-5-20251101-v1:0
            max-tokens: 4096
    mcp:
      client:
        streamable-http:
          connections:
            products-mcp-server:
              url: ${PRODUCTS_MCP_URL:http://localhost:8021}
            orders-mcp-server:
              url: ${ORDERS_MCP_URL:http://localhost:8031}
    model:
      embedding: bedrock-cohere
    vectorstore:
      mongodb:
        initialize-schema: true
        collection-name: ai_vector_store
        index-name: vector_index
        path-name: embedding
        metadata-fields-to-filter: conversationId,memoryType,createdAt
  data:
    mongodb:
      uri: ${MONGO_DB_URI:mongodb://admin:secret@localhost:27017/?authSource=admin&directConnection=true}
      database: ${MONGO_DB_NAME:shopping-agent}
server:
  port: 8050
```

The agent can be used without a web UI by sending requests to the `/api/v1/agent` endpoint with user input and a
conversation identifier. For example:

```bash
$ curl "http://localhost:8050/api/v1/agent?userInput=Show%20me%20all%20available%20laptops&conversationId=859bdab7-deef-4cef-90a6-3addda92c072"
```

The full source code for the agent is available on
GitHub: https://github.com/dominikcebula/spring-ai-shopping-agent/tree/main/agent

### Agent Chat Web UI

The agent's chat web UI is built with ReactJS and TypeScript. It communicates with the agent via the `/api/v1/agent`
endpoint.

On the frontend, a `conversationId` is kept in `localStorage` and sent with every request so that each user has an
isolated conversation (see the "Why isolate the conversation history per user?" section below for details):

```typescript
const STORAGE_KEY = 'shopping-agent.conversationId';

export function getConversationId(): string {
    const existing = localStorage.getItem(STORAGE_KEY);
    if (existing) {
        return existing;
    }
    const created = crypto.randomUUID();
    localStorage.setItem(STORAGE_KEY, created);
    return created;
}

export function resetConversationId(): string {
    const created = crypto.randomUUID();
    localStorage.setItem(STORAGE_KEY, created);
    return created;
}
```

The code that calls the agent API is shown below:

```typescript
export async function sendMessage(
    userInput: string,
    conversationId: string
): Promise<string> {
    const url = `/api/v1/agent?userInput=${encodeURIComponent(userInput)}` +
        `&conversationId=${encodeURIComponent(conversationId)}`;

    const response = await fetch(url, {method: 'GET'});

    if (!response.ok) {
        throw new Error(`Request failed with status ${response.status}`);
    }

    return response.text();
}
```

The full source code is available on
GitHub: https://github.com/dominikcebula/spring-ai-shopping-agent/tree/main/agent-chat-ui

### MCP Servers and MCP Tools

MCP servers host MCP tools for each domain, allowing the agent to access and manage the product catalog and orders.

Each MCP server acts as a wrapper around a backing microservice, which handles the business logic for search and order
operations within its domain.

Here is an example of MCP tools for the product catalog:

```java

@Component
public class ProductsTools {
    private final ProductsApi productsApi;

    public ProductsTools(ProductsApi productsApi) {
        this.productsApi = productsApi;
    }

    @McpTool(description = "Get all products from the catalog, optionally filtered by category and/or a search term matching product name, SKU, or tags")
    public List<Product> getAllProducts(
            @McpToolParam(required = false, description = "Product category (e.g. Laptops, Monitors, Keyboards, Mice, Headsets, Tablets, Smartphones, Smartwatches, Cameras, Audio, Accessories)")
            String category,
            @McpToolParam(required = false, description = "Search term matched against product name, SKU, or tags")
            String search) {
        return productsApi.getAllProducts(category, search);
    }

    @McpTool(description = "Get a product by its numeric identifier")
    public Product getProductById(
            @McpToolParam(description = "Product identifier")
            Long id) {
        return productsApi.getProductById(id);
    }
}
```

Here is an example of MCP tools for order management:

```java

@Component
public class OrdersTools {
    private final OrdersApi ordersApi;

    public OrdersTools(OrdersApi ordersApi) {
        this.ordersApi = ordersApi;
    }

    @McpTool(description = "Get all orders")
    public List<Order> getAllOrders() {
        return ordersApi.getAllOrders();
    }

    @McpTool(description = "Get an order by its identifier")
    public Order getOrder(
            @McpToolParam(description = "Order identifier (UUID)")
            UUID orderId) {
        return ordersApi.getOrder(orderId);
    }

    @McpTool(description = "Create a new order with customer details and items. Each item must include productId, productName, quantity, and unitPrice (the price snapshot taken from the product catalog at the time of ordering)")
    public Order createOrder(
            @McpToolParam(description = "Create order request containing customerName, customerEmail, and items (each with productId, productName, quantity, unitPrice)")
            CreateOrderRequest request) {
        return ordersApi.createOrder(request);
    }

    @McpTool(description = "Update an existing order with new customer details and/or items")
    public Order updateOrder(
            @McpToolParam(description = "Order identifier (UUID)")
            UUID orderId,
            @McpToolParam(description = "Update request containing customerName, customerEmail, and items (each with productId, productName, quantity, unitPrice)")
            UpdateOrderRequest request) {
        return ordersApi.updateOrder(orderId, request);
    }

    @McpTool(description = "Cancel an existing order by its identifier")
    public Order cancelOrder(
            @McpToolParam(description = "Order identifier (UUID)")
            UUID orderId) {
        return ordersApi.cancelOrder(orderId);
    }
}
```

The full source code is available on GitHub:

- https://github.com/dominikcebula/spring-ai-shopping-agent/tree/main/products/products-mcp-server
- https://github.com/dominikcebula/spring-ai-shopping-agent/tree/main/orders/orders-mcp-server

### MCP Server to Microservice Communication

MCP servers communicate with backing microservices via REST APIs.

To avoid duplicating code between MCP servers and
microservices, [Declarative HTTP Service Clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-http-service-client)
are used.

![MCP_Server_to_Microservice_Communication.drawio.png](docs/MCP_Server_to_Microservice_Communication.drawio.png)

Each microservice exposes a REST API that is also represented as a Java interface, for example:

```java

@HttpExchange("/api/v1/products")
public interface ProductsApi {

    @GetExchange
    List<Product> getAllProducts(
            @RequestParam(required = false) String category,
            @RequestParam(required = false) String search);

    @GetExchange("/{id}")
    Product getProductById(@PathVariable Long id);
}
```

The same interface is then used in the controller implementation:

```java

@RestController
public class ProductsController implements ProductsApi {

    // ...

    @Override
    public List<Product> getAllProducts(String category, String search) {
        // ...
    }

    @Override
    public Product getProductById(Long id) {
        // ...
    }
}
```

It is also used for client creation:

```java
public class ProductsClientFactory {
    private ProductsClientFactory() {
    }

    public static ProductsApi newProductsApiClient(String baseUrl) {
        return createClient(ProductsApi.class, baseUrl);
    }

    private static <S> S createClient(Class<S> serviceType, String baseUrl) {
        RestClient restClient = RestClient.create(baseUrl);
        RestClientAdapter adapter = RestClientAdapter.create(restClient);
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();
        return factory.createClient(serviceType);
    }
}
```

Each REST API client bean is then created as shown below:

```java

@Configuration
public class ProductsRestApiClientConfiguration {
    @Value("${products.api.base-uri}")
    private String baseUri;

    @Bean
    public ProductsApi productsApi() {
        return ProductsClientFactory.newProductsApiClient(baseUri);
    }
}
```

Downstream microservice URLs are configured in the application properties:

```properties
products.api.base-uri=${PRODUCTS_API_BASE_URI:http://localhost:8020}
```

By default, these URLs point to the local development environment. During deployment, the `PRODUCTS_API_BASE_URI`
environment variable is set to the actual URL of the microservice.

The full source code is available on GitHub (using the products example; the same approach applies to orders):

- https://github.com/dominikcebula/spring-ai-shopping-agent/tree/main/products/products-microservice-api
- https://github.com/dominikcebula/spring-ai-shopping-agent/tree/main/products/products-microservice-client
- https://github.com/dominikcebula/spring-ai-shopping-agent/blob/main/products/products-mcp-server/src/main/java/com/dominikcebula/spring/ai/products/configuration/ProductsRestApiClientConfiguration.java

### Microservices

Two microservices are used in this solution:

- **Products Microservice** — owns the product catalog (laptops, monitors, keyboards, mice, tablets, smartphones,
  smartwatches, cameras, audio, accessories)
- **Orders Microservice** — owns customer orders

Each microservice is developed with Spring Boot and exposes a REST API to handle search and management operations for
its domain.

For simplicity, each microservice uses in-memory storage to manage its data.

Each microservice implements its API as a Java interface to avoid code duplication between the REST API service
implementation and the REST API client.

Here is an example of the Orders API declared as a Java interface:

```java

@HttpExchange("/api/v1/orders")
public interface OrdersApi {

    @GetExchange
    List<Order> getAllOrders();

    @GetExchange("/{orderId}")
    Order getOrder(@PathVariable UUID orderId);

    @PostExchange
    Order createOrder(@RequestBody CreateOrderRequest request);

    @PutExchange("/{orderId}")
    Order updateOrder(@PathVariable UUID orderId, @RequestBody UpdateOrderRequest request);

    @DeleteExchange("/{orderId}")
    Order cancelOrder(@PathVariable UUID orderId);
}
```

Here is an example implementation of the Orders REST API service:

```java

@RestController
public class OrdersController implements OrdersApi {

    private final OrdersService ordersService;

    public OrdersController(OrdersService ordersService) {
        this.ordersService = ordersService;
    }

    @Override
    public List<Order> getAllOrders() {
        return ordersService.getAllOrders();
    }

    @Override
    public Order getOrder(UUID orderId) {
        return ordersService.getOrderById(orderId)
                .orElseThrow(() -> new ResourceNotFoundException("Order not found: " + orderId));
    }

    @Override
    @ResponseStatus(HttpStatus.CREATED)
    public Order createOrder(CreateOrderRequest request) {
        return ordersService.createOrder(request);
    }

    @Override
    public Order updateOrder(UUID orderId, UpdateOrderRequest request) {
        return ordersService.updateOrder(orderId, request)
                .orElseThrow(() -> new ResourceNotFoundException("Order not found: " + orderId));
    }

    @Override
    public Order cancelOrder(UUID orderId) {
        return ordersService.cancelOrder(orderId)
                .orElseThrow(() -> new ResourceNotFoundException("Order not found: " + orderId));
    }
}
```

Most of the business logic is implemented in `Service` classes, while data is stored in in-memory repository classes.

The full source code is available on GitHub:

- https://github.com/dominikcebula/spring-ai-shopping-agent/tree/main/products/products-microservice
- https://github.com/dominikcebula/spring-ai-shopping-agent/tree/main/orders/orders-microservice

## Persistent and Isolated Chat History

With the base agent in place, the next concern is memory. Shopping is inherently iterative — the user narrows a search,
asks follow-up questions, compares options, adjusts quantities, and eventually places an order. For the assistant to
feel
natural rather than mechanical, it must remember what has already been said.

### Why chat history is important?

Large Language Models (LLMs) are stateless by default. Each request is processed independently, with no built-in memory
of previous interactions. This means that context must be explicitly provided on every request if we want the model to
behave as if it "remembers" a conversation.

For a conversational application such as an AI shopping agent, this context is not optional. Chat history is what
allows the system to move beyond isolated question-answering and into a goal-oriented dialogue.

Chat history enables the AI to understand references to earlier messages, such as:

- Pronouns and implicit context ("Order that laptop instead")
- Follow-up questions ("What about cheaper options?")
- Progressive refinement ("Actually, make it one with at least 16 GB of RAM")

Without chat history, each user message must restate all prior context, leading to repetitive prompts and a poor user
experience.

Shopping is inherently iterative. A user might:

- Ask for product recommendations
- Narrow down by category, price range, or specifications
- Compare models
- Add accessories to the order
- Adjust quantities based on budget

Persisted chat history allows the AI to reason across these steps and maintain a shared mental model of the order as it
evolves. This is critical for producing consistent, relevant, and personalized responses.

By keeping chat history, the AI can adapt to user preferences over the course of a conversation:

- Preferred brands
- Budget sensitivity
- Required specifications (RAM, storage, screen size)
- Style preferences (gaming vs. business, wireless vs. wired)
- Shipping and warehouse preferences

Even within a single session, this significantly improves response quality and makes the interaction feel natural rather
than mechanical.

### Short-term memory vs. long-term memory

Chat history is a form of **short-term memory (STM)**. It stores the recent conversation for a single user as a series
of events. Each question and answer is stored, allowing the agent to access the context of the current conversation and
provide relevant, contextual responses.

**Long-term memory (LTM)** is different: it contains information extracted from conversations and stored in a
structured form. It captures key information such as user preferences, facts, and knowledge, and it persists across
sessions. LTM typically involves extraction and consolidation of information from conversations.

Both are needed, and we will cover them in turn. Chat history first, long-term memory in the next section.

### Why persist chat history?

By default, Spring AI stores chat history in memory using `InMemoryChatMemoryRepository`. This is fine for development
but is not suitable for production.

We need to persist chat history to durable storage so that it can be shared across multiple instances of the agent
service and survive restarts.

### How to persist chat history?

Chat history is persisted and accessed using `ChatMemoryRepository`.

The `ChatMemory` abstraction manages chat memory and decides which messages to keep and which to remove.

In this project, MongoDB is used as the persistent storage with `MongoChatMemoryRepository`.

The `ChatClient` creation remains unchanged — the advisor `MessageChatMemoryAdvisor` simply reads and writes through the
configured `ChatMemory`, which is backed by the MongoDB repository:

```java
public AgentController(ChatClient.Builder chatClientBuilder, ToolCallbackProvider toolCallbackProvider,
                       ChatMemory chatMemory, /* ... */) {
    this.chatClient = chatClientBuilder
            .defaultToolCallbacks(toolCallbackProvider)
            .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build()/* , ... */)
            // ...
            .build();
}
```

The MongoDB chat memory dependency autoconfigures `MongoChatMemoryRepository` and wires it up as the implementation of
`ChatMemoryRepository`, which is then used by `ChatClient`.

The Maven dependency looks like this:

```xml

<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-mongodb</artifactId>
</dependency>
```

To run the application locally, MongoDB is started via Docker Compose. For long-term memory we also need the vector
database capability, so the `mongodb/mongodb-atlas-local` image is used:

```yaml
services:
  mongo:
    image: 'mongodb/mongodb-atlas-local:8.0.14'
    restart: always
    ports:
      - "27017:27017"
    environment:
      MONGODB_INITDB_DATABASE: shopping-agent
      MONGODB_INITDB_ROOT_USERNAME: admin
      MONGODB_INITDB_ROOT_PASSWORD: secret
```

To automatically start MongoDB as a Docker container using Docker Compose, the `spring-boot-docker-compose` dependency
is
also added:

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-docker-compose</artifactId>
</dependency>
```

Whenever a user interacts with the agent, the chat history is persisted in MongoDB.

Let's look at the MongoDB collection where chat history is stored:

```bash
$ mongosh -u admin -p secret localhost:27017
...
test> use shopping-agent
shopping-agent> db.ai_chat_memory.find()
[
  {
    _id: ObjectId('698262d6ae98b4a088ba03ae'),
    conversationId: '859bdab7-deef-4cef-90a6-3addda92c072',
    message: {
      content: 'I prefer budget laptops for daily usage.',
      type: 'USER',
      metadata: { messageType: 'USER' }
    },
    timestamp: ISODate('2026-02-03T21:04:22.833Z'),
    _class: 'org.springframework.ai.chat.memory.repository.mongo.Conversation'
  },
  {
    _id: ObjectId('698262d6ae98b4a088ba03af'),
    conversationId: '859bdab7-deef-4cef-90a6-3addda92c072',
    message: {
      content: "Thanks for letting me know you prefer budget laptops!",
      type: 'ASSISTANT',
      metadata: { messageType: 'ASSISTANT' }
    },
    timestamp: ISODate('2026-02-03T21:04:22.833Z'),
    _class: 'org.springframework.ai.chat.memory.repository.mongo.Conversation'
  },
  ...
]
```

### Why isolate the conversation history per user?

Without isolation, all users would share the same chat history, resulting in a single conversation history containing
all user messages. This would cause conversation context to spill over into unrelated conversations.

As a result, the AI might confuse users' preferences, leading to incorrect orders and a poor user experience.

### How to implement conversation history isolation?

Implementing conversation history isolation requires changes on both the frontend and the agent side.

On the frontend, a `conversationId` is stored as a `UUID` in local storage. Whenever a user opens the app, the client
checks if there is a `conversationId` in local storage; if not, a new random UUID is generated and stored.

Whenever the `/api/v1/agent` endpoint is called, the `conversationId` is passed as a query parameter.

The code snippet below shows how this is implemented on the frontend:

```typescript
const STORAGE_KEY = 'shopping-agent.conversationId';

export function getConversationId(): string {
    const existing = localStorage.getItem(STORAGE_KEY);
    if (existing) {
        return existing;
    }
    const created = crypto.randomUUID();
    localStorage.setItem(STORAGE_KEY, created);
    return created;
}

export async function sendMessage(
    userInput: string,
    conversationId: string
): Promise<string> {
    const url = `/api/v1/agent?userInput=${encodeURIComponent(userInput)}` +
        `&conversationId=${encodeURIComponent(conversationId)}`;
    const response = await fetch(url, {method: 'GET'});
    if (!response.ok) {
        throw new Error(`Request failed with status ${response.status}`);
    }
    return response.text();
}
```

On the agent side, `conversationId` is received as a query parameter along with the user input.

The `conversationId` is then used by `advisorSpec` to set the `CONVERSATION_ID` when using `chatClient`.

The code snippet below shows how this is implemented on the agent side:

```java

@GetMapping("/agent")
public String generation(@RequestParam String userInput, @RequestParam UUID conversationId) {
    return chatClient.prompt()
            .user(userInput)
            .advisors(advisorSpec -> advisorSpec.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
}
```

With these two mechanisms in place — persistent storage and per-conversation isolation — the agent can hold stateful,
private conversations that survive restarts and scale across multiple service instances.

## Long-Term Memory (LTM)

Chat history solves short-term context, but it does not scale. This is where long-term memory comes in.

### Why do we need long-term memory?

Chat history usually contains the last 10–20 messages. Extending it with more messages increases costs due to higher
token consumption and may also cause issues with the maximum token limit. Chat history adds all of those messages to
each user prompt. From the user's perspective, interacting with an agent that uses short-term memory looks like the
agent is processing only a single message, but under the hood, all previous messages from chat history are appended as
well. So what appears to be a short message is actually the user's message plus the last 20 messages, which increases
token consumption.

The solution to scale agent memory in a cost-efficient way is to use **long-term memory (LTM) on top of short-term
memory (STM)**.

In this mode, long-term memory stores important information about user preferences and relevant facts, while chat
history provides recent context (last 20 messages).

That way the agent can process the user's prompt taking into account both recent conversation context and user
preferences and relevant facts from previous conversations — even if those preferences and facts were communicated
beyond the chat history window.

### How does long-term memory work?

Long-term memory is a storage mechanism for information that the agent should remember about the user over time.

Compared to short-term memory, it does not store all messages. Instead, relevant information is extracted from messages
and stored in a structured format. This allows for more selective and targeted storage in a compact form.

Long-term memory entries are extracted using LLMs and stored in a vector database. Each entry is associated with a
vector embedding, enabling efficient retrieval based on semantic similarity search.

Introducing long-term memory means that LLMs are used not only to answer questions but also to extract structured
information from user messages. Additionally, an embedding model is used to generate embeddings for each memory entry.

When the agent needs to access long-term memory, it queries the vector database using the current context to retrieve
relevant entries. This allows the agent to access important user preferences and facts from previous conversations
without including the entire chat history in the prompt.

When storing information in long-term memory, the agent checks whether a similar entry already exists in the vector
database. If not, it creates a new entry.

### Prompting for long-term memory extraction

The agent uses an LLM to extract information from user messages. Below is an example prompt for extracting long-term
memory from a dialog with the user:

```text
USER SAID:
I'm looking for a budget laptop with at least 8GB RAM for daily usage.

ASSISTANT REPLIED:
Thanks for sharing! I've noted that you prefer budget laptops with at least 8GB RAM
for everyday use. I'll keep this in mind for any future product recommendations.

YOUR TASK:
Extract up to 5 memories.
```

Here is the system prompt for extracting long-term memory:

```text
Extract long-term memories from a dialog with the user.

A memory is either:

1. EPISODIC: Personal experiences and user-specific preferences
   Examples: "User prefers budget laptops", "User prefers wireless peripherals"

2. SEMANTIC: General domain knowledge and facts
   Examples: "User is setting up a home office", "User is a gamer"

Limit extraction to clear, factual information. Do not infer information that was not explicitly stated.
Return an empty array, if no memories can be extracted.

The instance must conform to this JSON schema:
{
  "$schema" : "https://json-schema.org/draft/2020-12/schema",
  "type" : "object",
  "properties" : {
    "memories" : {
      "type" : "array",
      "items" : {
        "type" : "object",
        "properties" : {
          "content" : {
            "type" : "string"
          },
          "memoryType" : {
            "type" : "string",
            "enum" : [ "EPISODIC", "SEMANTIC" ]
          }
        },
        "required" : [ "content", "memoryType" ],
        "additionalProperties" : false
      }
    }
  },
  "required" : [ "memories" ],
  "additionalProperties" : false
}

Do not include code fences, schema, or properties. Output a single-line JSON object.
```

As a result, a long-term memory entry is extracted by the LLM and stored together with the vector created by the
embedding model:

```text
  {
    _id: 'd6440f1a-15d6-4253-a748-f5b68e5013bc',
    content: 'User prefers budget laptops with at least 8GB RAM for daily usage',
    metadata: {
      createdAt: ISODate('2026-02-17T20:45:18.584Z'),
      memoryType: 'EPISODIC',
      conversationId: '859bdab7-deef-4cef-90a6-3addda92c072'
    },
    embedding: [
            -0.034637451171875,  -0.0021152496337890625,       0.062103271484375,
            0.0277252197265625,        -0.0360107421875, -0.00009626150131225586,
          0.002040863037109375,     -0.0180816650390625,    0.005931854248046875,
          ...
          -0.0011997222900390625,     -0.0261383056640625,        -0.017456054687
    ],
    _class: 'org.springframework.ai.vectorstore.mongodb.atlas.MongoDBAtlasVectorStore$MongoDBDocument'
  }
```

### How will the AI agent use long-term memory?

The agent searches for relevant long-term memory entries using semantic search before answering a question. If entries
are found, they are added to the prompt context.

Here is an example of a prompt with extracted long-term memories:

```text
Use the Long-term MEMORY below if relevant. Keep answers factual and concise.

----- MEMORY -----
1. Memory Type: EPISODIC, Memory Content: User prefers budget laptops with at least 8GB RAM
2. Memory Type: EPISODIC, Memory Content: User prefers wireless peripherals
3. Memory Type: SEMANTIC, Memory Content: User is setting up a home office
------------------
```

### Types of long-term memory

Two types of long-term memory are implemented:

- **EPISODIC** — Personal experiences and user-specific preferences.
  Examples: "User prefers budget laptops", "User prefers wireless peripherals".

- **SEMANTIC** — General domain knowledge and facts.
  Examples: "User is setting up a home office", "User is a gamer".

### How to implement long-term memory in AI agents?

#### High-Level Architecture

The main additions compared to the base agent are memory retrieval before answering the user's question and memory
recording after processing the request. Both are implemented as Spring AI advisors that hook into the `ChatClient` call
chain.

#### Vector Storage

**MongoDB Atlas** is used as a vector database to store embeddings for each long-term memory entry. These embeddings are
used to retrieve entries through semantic search.

`application.yml` was updated as follows to configure MongoDB Atlas as a vector database:

```yaml
spring:
  ai:
    vectorstore:
      mongodb:
        initialize-schema: true
        collection-name: ai_vector_store
        index-name: vector_index
        path-name: embedding
        metadata-fields-to-filter: conversationId,memoryType,createdAt
  data:
    mongodb:
      uri: mongodb://localhost:27017/
      database: shopping-agent
```

The `docker-compose.yaml` shown earlier (in the chat history section) uses the `mongodb/mongodb-atlas-local` image,
which
provides both the chat memory store and the vector store out of a single container.

#### Embedding Model

To create vector embeddings for each long-term memory entry, the `bedrock-cohere` embedding model is used. The
`application.yml` file is configured as follows:

```yaml
spring:
  ai:
    model:
      embedding: bedrock-cohere
```

#### Recording Memories

Memories are recorded by `MemoryRecorderAdvisor` based on the user's prompt and the agent's response. The LLM is called
to extract long-term memories from the conversation, and the resulting entries are stored in the vector database.

See the code snippet below showing how `MemoryRecorderAdvisor` is implemented:

```java

@Component
public class MemoryRecorderAdvisor implements CallAdvisor {
    private final MemoryService memoryService;
    private final ChatModel chatModel;

    public MemoryRecorderAdvisor(MemoryService memoryService, ChatModel chatModel) {
        this.memoryService = memoryService;
        this.chatModel = chatModel;
    }

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest chatClientRequest, CallAdvisorChain callAdvisorChain) {
        ChatClientResponse chatClientResponse = callAdvisorChain.nextCall(chatClientRequest);

        extractAndStoreMemories(chatClientRequest, chatClientResponse);

        return chatClientResponse;
    }

    private void extractAndStoreMemories(ChatClientRequest chatClientRequest, ChatClientResponse chatClientResponse) {
        String userPrompt = chatClientRequest.prompt().getUserMessage().getText();
        String chatResponse = getChatResponse(chatClientResponse);

        MemoryExtractionResult memoryExtractionResult = extractMemories(userPrompt, chatResponse);

        memoryExtractionResult = filterOutSimilarMemories(chatClientRequest, memoryExtractionResult);

        storeExtractedMemories(chatClientRequest, memoryExtractionResult);
    }

    @NonNull
    private String getChatResponse(ChatClientResponse chatClientResponse) {
        return Optional.ofNullable(chatClientResponse.chatResponse())
                .map(response -> response.getResults().stream()
                        .map(Generation::getOutput)
                        .map(AssistantMessage::getText)
                        .collect(Collectors.joining()))
                .orElseThrow();
    }

    @NonNull
    private MemoryExtractionResult extractMemories(String userPrompt, String chatResponse) {
        String memoryExtractionUserMessage = getMemoryExtractionUserMessage(userPrompt, chatResponse);
        String memoryExtractionSystemMessage = getMemoryExtractionSystemMessage();

        ChatResponse memoryExtractionResponse = chatModel.call(new Prompt(List.of(
                new UserMessage(memoryExtractionUserMessage),
                new SystemMessage(memoryExtractionSystemMessage)
        )));

        String extractedMemories = memoryExtractionResponse.getResults().stream()
                .map(Generation::getOutput)
                .map(AssistantMessage::getText)
                .collect(Collectors.joining());

        return EXTRACTION_CONVERTER.convert(extractedMemories);
    }

    private MemoryExtractionResult filterOutSimilarMemories(ChatClientRequest chatClientRequest, MemoryExtractionResult memoryExtractionResult) {
        return new MemoryExtractionResult(
                memoryExtractionResult.memories().stream()
                        .filter(memory -> !memoryService.similarMemoryExists(
                                getConversationId(chatClientRequest), memory.content(), memory.memoryType(), SIMILARITY_90_PRC))
                        .toList());
    }

    private void storeExtractedMemories(ChatClientRequest chatClientRequest, MemoryExtractionResult memoryExtractionResult) {
        memoryExtractionResult.memories()
                .forEach(memory -> memoryService.storeMemory(
                        getConversationId(chatClientRequest), memory.content(), memory.memoryType()));
    }

    @NonNull
    private String getMemoryExtractionUserMessage(String userPrompt, String chatResponse) {
        return """
                USER SAID:
                """ +
                userPrompt
                + """
                
                ASSISTANT REPLIED:
                """ +
                chatResponse
                + """
                
                YOUR TASK:
                Extract up to 5 memories.
                """;
    }

    @NonNull
    private String getMemoryExtractionSystemMessage() {
        return """
                Extract long-term memories from a dialog with the user.
                
                A memory is either:
                
                1. EPISODIC: Personal experiences and user-specific preferences
                   Examples: "User prefers budget laptops", "User prefers wireless peripherals"
                
                2. SEMANTIC: General domain knowledge and facts
                   Examples: "User is setting up a home office", "User is a gamer"
                
                Limit extraction to clear, factual information. Do not infer information that was not explicitly stated.
                Return an empty array, if no memories can be extracted.
                
                The instance must conform to this JSON schema:
                """ +
                EXTRACTION_CONVERTER.getJsonSchema()
                + """
                
                    Do not include code fences, schema, or properties. Output a single-line JSON object.
                """.trim();
    }

    @Override
    public String getName() {
        return getClass().getSimpleName();
    }

    @Override
    public int getOrder() {
        return HIGHEST_PRECEDENCE + 60;
    }

    private record MemoryCandidate(String content, MemoryType memoryType) {
    }

    private record MemoryExtractionResult(List<MemoryCandidate> memories) {
    }

    private static final BeanOutputConverter<MemoryExtractionResult> EXTRACTION_CONVERTER = new BeanOutputConverter<>(MemoryExtractionResult.class);
}
```

The full source code of the `MemoryRecorderAdvisor` class can be found under:
https://github.com/dominikcebula/spring-ai-shopping-agent/blob/main/agent/src/main/java/com/dominikcebula/spring/ai/agent/memory/MemoryRecorderAdvisor.java

#### Retrieving Memories

Relevant memories are retrieved by `MemoryRetrievalAdvisor` and added to the prompt before calling the LLM. This way,
the LLM has access to relevant information about user preferences and facts.

The code snippet below shows how `MemoryRetrievalAdvisor` is implemented:

```java

@Component
public class MemoryRetrievalAdvisor implements CallAdvisor {
    private final MemoryService memoryService;

    public MemoryRetrievalAdvisor(MemoryService memoryService) {
        this.memoryService = memoryService;
    }

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest chatClientRequest, CallAdvisorChain callAdvisorChain) {
        String userPrompt = chatClientRequest.prompt().getUserMessage().getText();

        List<Memory> memories = memoryService.retrieveMemory(
                getConversationId(chatClientRequest),
                userPrompt, MEMORY_LIMIT_5_MEMORIES, SIMILARITY_90_PRC);

        if (!memories.isEmpty()) {
            String memory = """
                    Use the Long-term MEMORY below if relevant. Keep answers factual and concise.
                    
                    ----- MEMORY -----
                    """ +
                    IntStream.range(0, memories.size())
                            .mapToObj(idx -> String.format("%d. Memory Type: %s, Memory Content: %s",
                                    idx + 1, memories.get(idx).memoryType(), memories.get(idx).content()))
                            .reduce("", (a, b) -> a + b + "\n")
                    + """
                    ------------------
                    """;

            chatClientRequest.prompt().augmentSystemMessage(message -> {
                String currentPrompt = message.getText();

                String promptWithMemory = new StringBuilder()
                        .append(currentPrompt)
                        .append("\n\n")
                        .append(memory)
                        .toString();

                return message.mutate()
                        .text(promptWithMemory)
                        .build();
            });
        }

        return callAdvisorChain.nextCall(chatClientRequest);
    }

    @Override
    public String getName() {
        return getClass().getSimpleName();
    }

    @Override
    public int getOrder() {
        return HIGHEST_PRECEDENCE + 40;
    }
}
```

The full source code of the `MemoryRetrievalAdvisor` class can be found under:
https://github.com/dominikcebula/spring-ai-shopping-agent/blob/main/agent/src/main/java/com/dominikcebula/spring/ai/agent/memory/MemoryRetrievalAdvisor.java

#### Agent System Prompt

The system prompt for the agent (shown earlier in the "Agent" section) is updated to inform the model that two types of
memory are available: short-term memory (chat history) and long-term memory (EPISODIC and SEMANTIC). This helps the LLM
reason about when to rely on the current conversation thread and when to apply remembered preferences and facts.

#### Memory Service

Memory storage, retrieval, and similarity search are implemented in `MemoryService`.

The code snippet below shows how `MemoryService` is implemented:

```java

@Service
public class MemoryService {
    private static final String META_CONVERSATION_ID = "conversationId";
    private static final String META_MEMORY_TYPE = "memoryType";
    private static final String META_CREATED_AT = "createdAt";

    private final VectorStore vectorStore;

    public MemoryService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public void storeMemory(UUID conversationId, String content, MemoryType memoryType) {
        Memory memory = new Memory(UUID.randomUUID(), conversationId, content, memoryType, LocalDateTime.now());

        Document document = new Document(
                memory.id().toString(),
                memory.content(),
                Map.of(
                        META_CONVERSATION_ID, memory.conversationId().toString(),
                        META_MEMORY_TYPE, memory.memoryType(),
                        META_CREATED_AT, memory.createdAt()
                )
        );

        vectorStore.add(singletonList(document));
    }

    public boolean similarMemoryExists(UUID conversationId, String content, MemoryType memoryType, float distanceThreshold) {
        FilterExpressionBuilder filterExpressionBuilder = new FilterExpressionBuilder();

        Filter.Expression filterExpression = filterExpressionBuilder.and(
                filterExpressionBuilder.eq(META_CONVERSATION_ID, conversationId.toString()),
                filterExpressionBuilder.eq(META_MEMORY_TYPE, memoryType.name())
        ).build();

        List<Document> foundMemories = vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(content)
                        .topK(1)
                        .filterExpression(filterExpression)
                        .similarityThreshold(distanceThreshold)
                        .build());

        return !foundMemories.isEmpty();
    }

    public List<Memory> retrieveMemory(UUID conversationId, String userPrompt, int limit, float distanceThreshold) {
        FilterExpressionBuilder filterExpressionBuilder = new FilterExpressionBuilder();

        Filter.Expression filterExpression = filterExpressionBuilder.eq(META_CONVERSATION_ID, conversationId.toString()).build();

        List<Document> documents = vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(userPrompt)
                        .topK(limit)
                        .filterExpression(filterExpression)
                        .similarityThreshold(distanceThreshold)
                        .build());

        return documents.stream()
                .map(this::mapToMemory)
                .toList();
    }

    private Memory mapToMemory(Document document) {
        return new Memory(
                UUID.fromString(document.getId()),
                UUID.fromString(document.getMetadata().get(META_CONVERSATION_ID).toString()),
                document.getText(),
                MemoryType.valueOf(document.getMetadata().get(META_MEMORY_TYPE).toString()),
                DateUtils.toLocalDateTime((Date) document.getMetadata().get(META_CREATED_AT))
        );
    }
}
```

The full source code of the `MemoryService` class can be found under:
https://github.com/dominikcebula/spring-ai-shopping-agent/blob/main/agent/src/main/java/com/dominikcebula/spring/ai/agent/memory/MemoryService.java

## Running the Project Locally

To run the project locally, you will need the following prerequisites:

- Java 25
- Maven 3.9+
- Node.js 18+
- Docker / Docker Compose
- AWS account with Bedrock access (Claude Opus 4.5 model enabled)
- AWS credentials configured (`~/.aws/credentials` or environment variables)

Once all prerequisites are in place, you can clone the project:

```bash
git clone https://github.com/dominikcebula/spring-ai-shopping-agent.git
```

Build the project:

```bash
mvn clean install
```

### Using Docker Compose

Start everything (agent, MCP servers, microservices, UI, and MongoDB Atlas Local) with a single command:

```bash
docker compose up
```

Then open http://localhost:8080.

### Running Services Locally

Alternatively, run each service directly:

```bash
# Microservices
(cd products/products-microservice && mvn spring-boot:run) &
(cd orders/orders-microservice && mvn spring-boot:run) &

# MCP Servers
(cd products/products-mcp-server && mvn spring-boot:run) &
(cd orders/orders-mcp-server && mvn spring-boot:run) &

# Agent (starts MongoDB Atlas Local via docker-compose integration)
(cd agent && mvn spring-boot:run) &
```

Run the agent chat UI:

```bash
cd agent-chat-ui
npm install
npm start
```

Alternatively, you can use the IntelliJ run configurations included in the repository to start all the services and the
agent chat UI.

Once everything is running, open http://localhost:3000/.

You can now interact with the agent to search the catalog, compare products, and create orders — and the agent will
remember your preferences across conversations.

## Further Enhancements

The following enhancements could be implemented in the future:

- Replace per-conversation isolation with per-user isolation tied to an authenticated identity, so that long-term memory
  follows the user across devices and sessions.
- Implement agent correctness evaluation.
- Add automated expiration and re-consolidation of long-term memories.
- Security hardening.
- Store products and orders data in a database instead of in-memory storage.
- Perform validation of user input on the microservices side.

## Summary

This article demonstrated how to build an AI-powered shopping agent using Spring AI, and then how to extend it with
persistent chat history and long-term memory.

The base solution uses a microservices architecture with two domains — products and orders — each consisting of a REST
API microservice and an MCP server exposing tools to the AI agent. The central agent, powered by Claude Opus 4.5 on
Amazon Bedrock, connects to the MCP servers via streamable HTTP and uses a system prompt to constrain its behavior to
shopping-related tasks. A React-based chat UI provides the user interface for natural language interactions.

Persistent chat history is implemented using MongoDB with Spring AI's `MongoChatMemoryRepository`, so conversation data
survives application restarts and can be shared across multiple service instances. Conversation isolation is achieved
with a unique `conversationId` (UUID) per user session — stored in local storage on the frontend and passed with each
request, then used on the agent side with `ChatMemory.CONVERSATION_ID` to keep each user's conversation separate.

Long-term memory is added on top of that: structured memories (EPISODIC and SEMANTIC) are extracted from conversations
by an LLM, embedded using `bedrock-cohere`, and stored in MongoDB Atlas as a vector database. A `MemoryRetrievalAdvisor`
retrieves the most relevant memories via semantic search before each LLM call and injects them into the system prompt; a
`MemoryRecorderAdvisor` extracts and stores new memories after each response, deduplicating against existing entries.

This combination — tool-driven domain access via MCP, persistent and isolated chat history, and scalable long-term
memory — enables more personalized, context-aware, and cost-efficient interactions without overloading the prompt with
full chat history. The project serves as a practical example of using Spring AI to build conversational AI applications
that can perform real actions through tool calling and remember what matters about each user.

## References

- [Source Code on GitHub](https://github.com/dominikcebula/spring-ai-shopping-agent/)
- [Spring AI Documentation](https://docs.spring.io/spring-ai/reference/)
- [Awesome Spring AI](https://github.com/spring-ai-community/awesome-spring-ai)
- De Lio, R. (2025, July 16). *Agent Long-term Memory with Spring AI & Redis*. Retrieved February 16, 2026,
  from https://medium.com/redis-with-raphael-de-lio/agent-memory-with-spring-ai-redis-af26dc7368bd

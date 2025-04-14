# Notion MCP SSE Server - Endpoints & Implementation Details

## Required Endpoints

Based on the MCP TypeScript SDK's example for HTTP with SSE, the following endpoints are required for your Notion MCP SSE Server:

### 1. Messages Endpoint (`/messages`)

This endpoint handles synchronous message processing via HTTP POST.

**POST Method:**
- **Purpose**: Process MCP client requests (tool calls, initialization)
- **Headers**:
  - `Content-Type: application/json`
  - `Authorization: Bearer <token>` (for future auth implementation)
- **Request Body**: MCP message object formatted as JSON
- **Response**:
  - Status: 200 OK (success), appropriate error codes for failures
  - Content-Type: application/json
  - Body: MCP response object

### 2. SSE Endpoint (`/sse`)

This endpoint establishes Server-Sent Events connections for streaming updates to clients.

**GET Method:**
- **Purpose**: Establish SSE connection for streaming updates
- **Headers**:
  - `Accept: text/event-stream`
  - `Cache-Control: no-cache`
  - `Connection: keep-alive`
  - `Last-Event-ID: <id>` (optional, for reconnection)
- **Response**:
  - Status: 200 OK
  - Content-Type: text/event-stream
  - Connection: keep-alive
  - Cache-Control: no-cache
  - Body: Stream of SSE events

### 2. Health Check Endpoint (`/health`)

Simple endpoint for monitoring the service health.

**GET Method:**
- **Purpose**: Verify service is running and healthy
- **Response**:
  - Status: 200 OK if healthy
  - Content-Type: application/json
  - Body: `{"status": "ok", "timestamp": "<iso-timestamp>"}`

### 3. Version/Info Endpoint (`/info`)

Provides information about the server implementation.

**GET Method:**
- **Purpose**: Return server metadata
- **Response**:
  - Status: 200 OK
  - Content-Type: application/json
  - Body: 
    ```json
    {
      "name": "notion-mcp-sse",
      "version": "1.0.0",
      "transport": "http+sse",
      "capabilities": {
        "streaming": true,
        "...": "..."
      }
    }
    ```

## SSE Event Types

The SSE implementation must support several event types for MCP protocol compliance:

### 1. Default Event (no explicit type)
For standard MCP protocol messages:
```
data: {"type": "response", "id": "123", "result": {...}}
```

### 2. Error Event
For error notifications:
```
event: error
data: {"code": "error_code", "message": "Error description"}
```

### 3. Ping/Heartbeat Event
To keep the connection alive:
```
event: ping
data: {"timestamp": "2025-04-14T12:00:00Z"}
```

### 4. Tool Update Event
For tool list updates:
```
event: tools-update
data: {"tools": [...]}
```

### 5. Server Request Event
For server-initiated requests:
```
event: request
data: {"type": "request", "id": "123", "method": "method_name", "params": {...}}
```

## Implementation Architecture

### 1. Express.js Server Configuration

```typescript
import express from 'express';
import cors from 'cors';
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { HttpSseServerTransport } from '@modelcontextprotocol/sdk/server/http-sse.js';
import { NotionApiClient } from './notion-api-client';

const app = express();

// Middleware
app.use(cors({
  origin: '*', // Configure appropriately for production
  methods: ['GET', 'POST', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'Accept', 'Last-Event-ID']
}));
app.use(express.json());

// MCP Server setup
const server = new McpServer({
  name: 'notion-mcp-sse',
  version: '1.0.0'
});

// Initialize the HTTP+SSE transport
const transport = new HttpSseServerTransport(server);

// Register the Notion API tools
const notionClient = new NotionApiClient(process.env.NOTION_API_TOKEN);
notionClient.registerTools(server);

// Messages endpoint for synchronous HTTP requests
app.post('/messages', async (req, res) => {
  try {
    const message = req.body;
    const response = await server.handleMessage(message);
    res.json(response);
  } catch (error) {
    res.status(500).json({
      error: error.message || 'Internal server error'
    });
  }
});

// SSE endpoint for asynchronous streaming
app.get('/sse', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  // Send any pending messages to the client
  transport.flush(res);

  // Save the response object so we can send future messages
  transport.connect(res);

  // When the client disconnects, stop sending messages
  req.on('close', () => {
    transport.disconnect(res);
  });
});

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    timestamp: new Date().toISOString()
  });
});

// Info endpoint
app.get('/info', (req, res) => {
  res.json({
    name: server.info.name,
    version: server.info.version,
    transport: 'http+sse',
    capabilities: server.capabilities
  });
});

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Notion MCP SSE Server running on port ${PORT}`);
});
```

### 2. Using the Built-in HTTP+SSE Transport

The MCP TypeScript SDK provides a built-in `HttpSseServerTransport` implementation that we can leverage instead of creating our own. This transport handles all the necessary functionality for managing SSE connections and message delivery.

The key components of this transport are:

1. **Connection Management**:
   - `connect(res)`: Registers a client connection
   - `disconnect(res)`: Removes a client connection
   - `flush(res)`: Sends any pending messages to a client

2. **Message Handling**:
   - The transport automatically subscribes to the server's message events
   - Messages are sent to all connected clients
   - Event IDs are managed for reconnection support

3. **Error Handling**:
   - Connection errors are properly managed
   - The transport handles client disconnections

Here's an example of the transport initialization in your application:

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { HttpSseServerTransport } from '@modelcontextprotocol/sdk/server/http-sse.js';

// Create MCP server
const server = new McpServer({
  name: 'notion-mcp-sse',
  version: '1.0.0'
});

// Initialize HTTP+SSE transport
const transport = new HttpSseServerTransport(server, {
  // Optional configuration options
  heartbeatInterval: 30000, // Send heartbeat every 30 seconds
  maxConnections: 100,      // Maximum number of SSE connections
  // Any other transport-specific options
});

// The transport automatically connects to the server's events
// and will handle message delivery to connected clients
```

This approach leverages the SDK's built-in transport implementation, which is maintained by the MCP team and follows the protocol specification.
```

### 3. Docker Configuration

```dockerfile
# Build stage
FROM node:18-alpine as build

WORKDIR /app

# Copy package files
COPY package.json package-lock.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY tsconfig.json ./
COPY src ./src

# Build TypeScript code
RUN npm run build

# Production stage
FROM node:18-alpine

WORKDIR /app

# Copy built files and dependencies from build stage
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
COPY package.json ./

# Run as non-root user
USER node

# Set environment variables
ENV NODE_ENV=production
ENV PORT=8080

# Expose port
EXPOSE 8080

# Start the server
CMD ["node", "dist/index.js"]
```

## Google Cloud Run Deployment

To deploy this service to Google Cloud Run:

1. Build the Docker image:
   ```
   docker build -t notion-mcp-sse .
   ```

2. Tag the image for Google Container Registry:
   ```
   docker tag notion-mcp-sse gcr.io/[YOUR-PROJECT-ID]/notion-mcp-sse
   ```

3. Push the image:
   ```
   docker push gcr.io/[YOUR-PROJECT-ID]/notion-mcp-sse
   ```

4. Deploy to Cloud Run:
   ```
   gcloud run deploy notion-mcp-sse \
     --image gcr.io/[YOUR-PROJECT-ID]/notion-mcp-sse \
     --platform managed \
     --allow-unauthenticated \
     --region [REGION] \
     --set-env-vars="NOTION_API_TOKEN=[YOUR-TOKEN]"
   ```

## Client Connection Example

Here's an updated client connection example that uses the correct endpoints:

```javascript
// Browser-side code
const connectToMcpServer = () => {
  const eventSource = new EventSource('https://your-cloud-run-url.run.app/sse');
  
  eventSource.onopen = () => {
    console.log('Connection established');
  };
  
  eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log('Received message:', data);
    // Handle MCP message
  };
  
  eventSource.addEventListener('error', (event) => {
    const data = JSON.parse(event.data);
    console.error('Error:', data);
  });
  
  eventSource.addEventListener('ping', (event) => {
    console.log('Ping received');
  });
  
  eventSource.onerror = (error) => {
    console.error('EventSource error:', error);
    eventSource.close();
    
    // Implement reconnection logic
    setTimeout(connectToMcpServer, 3000);
  };
  
  // Function to make tool calls
  window.callMcpTool = async (toolId, params) => {
    try {
      const response = await fetch('https://your-cloud-run-url.run.app/messages', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          id: crypto.randomUUID(),
          method: 'tools.call',
          params: {
            tool: toolId,
            params: params
          }
        })
      });
      
      return await response.json();
    } catch (error) {
      console.error('Error calling tool:', error);
      throw error;
    }
  };
  
  return eventSource;
};
```

## Security Considerations

1. **CORS Configuration**: For production, restrict CORS to only allowed origins.

2. **Authentication**: Add proper authentication for both HTTP and SSE endpoints.

3. **Rate Limiting**: Implement rate limiting to prevent abuse.

4. **Secure Headers**: Use secure headers to prevent common web vulnerabilities.

5. **Cloud Run Configuration**: Configure Cloud Run with appropriate security settings:
   - Use service account with minimal permissions
   - Set up VPC Service Controls if needed
   - Configure Cloud Armor for additional protection

6. **Secure Environment Variables**: Store sensitive information like API tokens in Secret Manager.
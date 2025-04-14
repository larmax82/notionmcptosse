# Notion MCP Server - Implementation Documentation

## Overview

The Notion MCP Server is an implementation of the [Model Context Protocol (MCP)](https://spec.modelcontextprotocol.io/) for the [Notion API](https://developers.notion.com/reference/intro). It allows AI assistants like Claude to interact with Notion via a standardized interface. This document describes the technical implementation of the MCP server and its components.

## Architecture

The Notion MCP Server is based on an OpenAPI-to-MCP conversion approach, which translates the Notion API's OpenAPI specification into MCP-compatible tools that AI assistants can use. The overall architecture follows these layers:

1. **MCP Layer**: Handles communication with AI assistants using the Model Context Protocol
2. **OpenAPI Conversion Layer**: Converts OpenAPI specifications to MCP tools
3. **HTTP Client Layer**: Manages API requests to the Notion API
4. **Authentication Layer**: Manages authentication with the Notion API

## Key Components

### 1. MCP Proxy (`MCPProxy` class)

The `MCPProxy` class (in `src/openapi-mcp-server/mcp/proxy.ts`) serves as the main entry point for the MCP implementation. It:

- Creates an MCP server using the `@modelcontextprotocol/sdk` library
- Sets up handlers for tool listing and tool calls
- Converts Notion API specifications to MCP tool
- Routes MCP tool calls to the underlying HTTP client

Key methods:
- `setupHandlers()`: Configures MCP request handlers for tool listing and tool calls
- `findOperation()`: Looks up the appropriate OpenAPI operation for a given MCP tool name
- `parseHeadersFromEnv()`: Extracts authentication headers from environment variables
- `connect()`: Connects the MCP server to a transport (like stdio)

### 2. OpenAPI to MCP Converter (`OpenAPIToMCPConverter` class)

The `OpenAPIToMCPConverter` class (in `src/openapi-mcp-server/openapi/parser.ts`) handles conversion from OpenAPI specifications to MCP tools. It:

- Parses OpenAPI operations, parameters, and schemas
- Converts OpenAPI schemas to JSON Schema (used by MCP)
- Generates appropriate tool descriptions and input schemas
- Maintains a mapping between MCP tool names and OpenAPI operations

Key methods:
- `convertToMCPTools()`: Converts the entire OpenAPI spec to MCP tools
- `convertOpenApiSchemaToJsonSchema()`: Transforms OpenAPI schemas to JSON Schema
- `convertOperationToMCPMethod()`: Converts an individual OpenAPI operation to an MCP tool method

### 3. HTTP Client (`HttpClient` class)

The `HttpClient` class (in `src/openapi-mcp-server/client/http-client.ts`) manages communication with the Notion API. It:

- Initializes an OpenAPI client using the provided specification
- Handles authentication via headers
- Manages request parameters, URL construction, and response handling
- Supports file uploads and multipart form data

Key methods:
- `executeOperation()`: Executes an OpenAPI operation with provided parameters
- `prepareFileUpload()`: Prepares file upload requests using FormData

### 4. Server Initialization

The server initialization (in `src/init-server.ts` and `scripts/start-server.ts`) handles:

- Loading and validating the OpenAPI specification
- Creating the MCP proxy with the validated specification
- Connecting the proxy to the appropriate transport (stdio)
- Error handling for invalid specifications or server failures

## Data Flow

1. **Initialization**:
   - The OpenAPI specification for Notion API is loaded
   - The specification is validated
   - An `MCPProxy` instance is created with the specification
   - The proxy connects to the stdio transport for communication with the AI assistant

2. **Tool Listing**:
   - When an AI assistant requests available tools, the MCP server responds with the converted Notion API operations
   - Each API operation is represented as an MCP tool with appropriate name, description, and input schema
   - Tool names are truncated to meet MCP limitations (64 characters max)

3. **Tool Calls**:
   - When an AI assistant calls a tool, the MCP server looks up the corresponding OpenAPI operation
   - Parameters are extracted and formatted according to the operation's requirements
   - The HTTP client executes the operation against the Notion API
   - The response is formatted as an MCP response and returned to the AI assistant

4. **Error Handling**:
   - HTTP errors from the Notion API are caught and formatted as MCP responses
   - Validation errors for parameters are reported back to the AI assistant
   - Server-side errors are logged and may terminate the server process

## Authentication

The MCP server authenticates with the Notion API using:

1. An API token provided via the `OPENAPI_MCP_HEADERS` environment variable
2. The headers are passed to every API request made to Notion
3. Example header configuration:
   ```json
   {
     "Authorization": "Bearer ntn_****",
     "Notion-Version": "2022-06-28"
   }
   ```

## Deployment Methods

The Notion MCP Server can be deployed using:

1. **NPM Package**:
   - Installed via `npx -y @notionhq/notion-mcp-server`
   - Configured in the Cursor or Claude Desktop configuration file

2. **Docker**:
   - Built using the provided Dockerfile
   - Run with environment variables for authentication

## Security Considerations

1. The MCP server has access to all Notion resources connected to the provided API token
2. The scope of access can be limited by configuring the integration's capabilities in Notion
3. Content must be explicitly connected to the integration in Notion for access

## Technical Details

### Dependencies

- `@modelcontextprotocol/sdk`: Provides the MCP server implementation
- `openapi-client-axios`: Generates API clients from OpenAPI specifications
- `axios`: Handles HTTP requests to the Notion API
- `form-data`: Supports file uploads to the Notion API
- `zod`: Used for schema validation

### File Structure

```
notion-mcp-server/
├── src/
│   ├── init-server.ts                  # Server initialization
│   └── openapi-mcp-server/
│       ├── auth/                       # Authentication handling
│       ├── client/
│       │   └── http-client.ts          # HTTP client implementation
│       ├── mcp/
│       │   └── proxy.ts                # MCP server implementation
│       ├── openapi/
│       │   └── parser.ts               # OpenAPI to MCP conversion
│       └── index.ts                    # Main exports
├── scripts/
│   ├── build-cli.js                    # CLI builder
│   ├── notion-openapi.json             # Notion API OpenAPI spec
│   └── start-server.ts                 # Server startup script
└── bin/
    └── cli.mjs                         # Generated CLI executable
```

## Important Implementation Notes

1. **Fork History**: This implementation is a fork from v1 of https://github.com/snaggle-ai/openapi-mcp-server. The original library took a different direction with v2 which was not compatible with the development approach needed for Notion integration.

2. **Tool Name Uniqueness**: Since multiple API operations might have similar names, the implementation ensures unique tool names by appending a counter when necessary.

3. **OpenAPI Schema Conversion**: The implementation includes comprehensive conversion logic to transform OpenAPI schemas into JSON Schema format used by MCP, handling nested references, complex types, and file uploads.

4. **Performance Considerations**: The implementation caches resolved schema references to improve performance when working with complex API specifications.

## Conclusion

The Notion MCP Server provides a bridge between AI assistants and the Notion API by implementing the Model Context Protocol. This enables AI assistants to:

1. Search for Notion content
2. Create new pages and databases
3. Update existing content
4. Add comments and other interactions

All through a standardized, well-documented interface that follows the MCP specification. 
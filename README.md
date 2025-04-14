# Notion MCP SSE Server

An Express-based server implementation of the Model Completion Protocol (MCP) using Server-Sent Events (SSE) for streaming responses. This server integrates with Notion to provide AI completions via the MCP protocol.

## Overview

The Notion MCP SSE Server extends an existing Express.js application to support Server-Sent Events (SSE) functionality for streaming AI model completions. It implements the MCP protocol to facilitate communication between Notion and AI models.

## Features

- HTTP endpoint for synchronous message processing (`/messages`)
- SSE endpoint for streaming updates and real-time events (`/sse`)
- Health check and version/info endpoints for monitoring
- Comprehensive security middleware
- Authentication system using Bearer tokens
- Docker containerization support
- Google Cloud Run deployment configuration

## Getting Started

### Prerequisites

- Node.js 16+
- npm or yarn
- Docker (optional for containerization)

### Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/notion-mcp-sse.git
   cd notion-mcp-sse
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Run the server:
   ```bash
   npm start
   ```

### Development

```bash
npm run dev
```

## Documentation

For more detailed information about the implementation and API endpoints, see the [MCP_Implementation_Documentation.md](MCP_Implementation_Documentation.md) file.

## License

This project is licensed under the MIT License - see the LICENSE file for details.

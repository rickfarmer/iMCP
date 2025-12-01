# iMCP Claude Code Plugin Architecture Specification

## Executive Summary

This specification describes the architecture for a hybrid Claude Code plugin that wraps the existing iMCP MCP server, providing seamless integration between Claude Code and macOS system services (Calendar, Contacts, Location, Maps, Messages, Reminders, Weather, Operator, Files, Capture).

### Design Philosophy

- **Hybrid Architecture**: Plugin wraps existing MCP server rather than reimplementing services
- **Process Management**: Plugin manages iMCP server lifecycle (start/stop/restart)
- **JSON-RPC Communication**: stdio-based communication following MCP protocol
- **Service Discovery**: No hardcoded paths - use actual iMCP.app bundle location
- **Error Resilience**: Graceful degradation with clear error messages
- **Connection Management**: Automatic reconnection with backoff strategy

---

## 1. Directory Structure

```
~/.claude/plugins/imcp/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── lib/
│   ├── imcp-client.mjs          # Core MCP client library
│   ├── process-manager.mjs      # Server process lifecycle management
│   ├── json-rpc.mjs             # JSON-RPC 2.0 protocol implementation
│   └── error-handler.mjs        # Error handling and recovery
├── tools/
│   ├── calendar.mjs             # Calendar service tools
│   ├── contacts.mjs             # Contacts service tools
│   ├── location.mjs             # Location service tools
│   ├── maps.mjs                 # Maps service tools
│   ├── messages.mjs             # Messages service tools
│   ├── reminders.mjs            # Reminders service tools
│   ├── weather.mjs              # Weather service tools
│   ├── operator.mjs             # Operator service tools (UI automation)
│   ├── files.mjs                # Files service tools
│   └── capture.mjs              # Screen capture tools
├── tests/
│   ├── test_imcp-client.mjs     # Client library tests
│   ├── test_calendar.mjs        # Calendar tools tests
│   └── ...                      # Tests for each service
├── SKILL.md                     # Main skill documentation
├── README.md                    # Plugin overview and setup
└── LICENSE.txt                  # MIT License

```

---

## 2. MCP Client Library Architecture

### 2.1 Core Components

#### `lib/imcp-client.mjs`

The main client library that orchestrates all interactions with the iMCP server.

**Primary Responsibilities:**
- Initialize and manage connection to iMCP server
- Send MCP protocol requests and handle responses
- Provide high-level API for tool calls
- Handle errors and implement retry logic
- Manage connection lifecycle

**API Interface:**

```javascript
import { IMCPClient } from './lib/imcp-client.mjs';

// Initialize client
const client = new IMCPClient({
  serverPath: '/path/to/iMCP.app/Contents/MacOS/imcp-server',
  timeout: 30000,              // 30 second timeout
  maxRetries: 3,               // Retry failed requests 3 times
  reconnectBackoff: 1000,      // Start with 1s backoff, exponential
  logLevel: 'info'             // debug, info, warn, error
});

// Lifecycle methods
await client.connect();        // Start server and establish connection
await client.disconnect();     // Stop server gracefully
client.isConnected();          // Check connection status

// Tool calling API
const result = await client.callTool('calendar_list_events', {
  startDate: '2025-11-30T00:00:00Z',
  endDate: '2025-12-07T23:59:59Z'
});

// Resource API (for Files service)
const fileContent = await client.readResource('file:///Users/farmer/document.txt');

// List available tools
const tools = await client.listTools();

// Health check
const status = await client.getServerStatus();
```

#### `lib/process-manager.mjs`

Manages the iMCP server process lifecycle.

**Primary Responsibilities:**
- Start/stop the iMCP server process
- Monitor process health
- Handle process crashes and restarts
- Capture stdout/stderr for debugging

**API Interface:**

```javascript
import { ProcessManager } from './lib/process-manager.mjs';

const manager = new ProcessManager({
  serverPath: '/path/to/iMCP.app/Contents/MacOS/imcp-server',
  args: [],                    // Additional server arguments
  env: { ...process.env },     // Environment variables
  restartOnCrash: true,        // Auto-restart if server crashes
  maxRestarts: 5,              // Maximum restart attempts
  logStdout: true,             // Log server stdout
  logStderr: true              // Log server stderr
});

await manager.start();         // Start the server process
await manager.stop();          // Stop the server gracefully
await manager.restart();       // Restart the server
manager.isRunning();           // Check if process is running
manager.getProcessId();        // Get server PID
```

#### `lib/json-rpc.mjs`

Implements JSON-RPC 2.0 protocol for MCP communication.

**Primary Responsibilities:**
- Serialize requests to JSON-RPC format
- Parse responses and handle errors
- Manage request IDs
- Implement notification support

**API Interface:**

```javascript
import { JSONRPCClient } from './lib/json-rpc.mjs';

const rpc = new JSONRPCClient({
  send: async (message) => {
    // Send to stdio stream
  },
  receive: async () => {
    // Read from stdio stream
  }
});

// Call a method
const result = await rpc.call('tools/call', {
  name: 'calendar_list_events',
  arguments: { startDate: '...' }
});

// Send notification (no response expected)
await rpc.notify('ping');
```

#### `lib/error-handler.mjs`

Centralized error handling and recovery strategies.

**Primary Responsibilities:**
- Classify error types (network, protocol, application)
- Implement retry logic with exponential backoff
- Provide user-friendly error messages
- Log errors for debugging

**Error Categories:**

1. **Connection Errors**: Server not running, connection refused
2. **Protocol Errors**: Invalid JSON-RPC, malformed responses
3. **Application Errors**: Tool not found, invalid arguments
4. **Permission Errors**: Service not activated in iMCP.app
5. **Timeout Errors**: Request took too long

**Recovery Strategies:**

```javascript
import { ErrorHandler } from './lib/error-handler.mjs';

const handler = new ErrorHandler({
  maxRetries: 3,
  retryDelay: 1000,
  backoffMultiplier: 2
});

try {
  const result = await handler.withRetry(async () => {
    return await client.callTool('weather_get_current', { location: 'SF' });
  });
} catch (error) {
  if (handler.isRetryable(error)) {
    // Can retry
  } else {
    // Fatal error - show user message
    console.error(handler.getUserMessage(error));
  }
}
```

---

## 3. Tool Implementation Pattern

Each service is exposed through a dedicated tool module that wraps MCP server calls.

### Example: Calendar Tools

**File: `tools/calendar.mjs`**

```javascript
import { IMCPClient } from '../lib/imcp-client.mjs';

/**
 * Calendar service tools for accessing macOS Calendar events
 */
export class CalendarTools {
  constructor(client) {
    this.client = client;
  }

  /**
   * List calendar events within a date range
   * @param {string} startDate - ISO 8601 start date
   * @param {string} endDate - ISO 8601 end date
   * @param {string} [calendar] - Optional calendar name filter
   * @returns {Promise<Array>} Array of calendar events
   */
  async listEvents({ startDate, endDate, calendar }) {
    return await this.client.callTool('calendar_list_events', {
      startDate,
      endDate,
      calendar
    });
  }

  /**
   * Create a new calendar event
   * @param {Object} event - Event details
   * @returns {Promise<Object>} Created event
   */
  async createEvent(event) {
    return await this.client.callTool('calendar_create_event', event);
  }

  /**
   * Get list of available calendars
   * @returns {Promise<Array>} Array of calendar names
   */
  async listCalendars() {
    return await this.client.callTool('calendar_list_calendars', {});
  }
}

/**
 * Export tool registration for Claude Code
 */
export function registerCalendarTools(server, client) {
  const tools = new CalendarTools(client);

  server.registerTool({
    name: 'imcp_calendar_list_events',
    description: 'List calendar events within a date range from macOS Calendar',
    inputSchema: {
      type: 'object',
      properties: {
        startDate: {
          type: 'string',
          description: 'ISO 8601 start date (e.g., 2025-11-30T00:00:00Z)'
        },
        endDate: {
          type: 'string',
          description: 'ISO 8601 end date (e.g., 2025-12-07T23:59:59Z)'
        },
        calendar: {
          type: 'string',
          description: 'Optional calendar name filter'
        }
      },
      required: ['startDate', 'endDate']
    }
  }, async (params) => {
    try {
      const events = await tools.listEvents(params);
      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify(events, null, 2)
          }
        ]
      };
    } catch (error) {
      return {
        content: [
          {
            type: 'text',
            text: `Error listing events: ${error.message}`
          }
        ],
        isError: true
      };
    }
  });

  // Register other calendar tools...
}
```

---

## 4. Server Discovery and Path Resolution

Instead of hardcoding the server path, implement dynamic discovery:

### Approach 1: Bundle Location Detection (Preferred)

```javascript
import { execSync } from 'child_process';
import { existsSync } from 'fs';
import path from 'path';

/**
 * Find iMCP.app bundle location
 * @returns {string|null} Path to imcp-server executable
 */
export function findIMCPServer() {
  // Common installation locations
  const locations = [
    '/Applications/iMCP.app',
    '/Applications/Utilities/iMCP.app',
    `${process.env.HOME}/Applications/iMCP.app`
  ];

  // Try using mdfind (Spotlight)
  try {
    const result = execSync(
      'mdfind "kMDItemKind == Application && kMDItemDisplayName == iMCP"',
      { encoding: 'utf8', timeout: 5000 }
    );
    const apps = result.trim().split('\n').filter(Boolean);
    if (apps.length > 0) {
      locations.unshift(...apps);
    }
  } catch (error) {
    // Spotlight search failed, continue with manual locations
  }

  // Check each location
  for (const appPath of locations) {
    const serverPath = path.join(
      appPath,
      'Contents/MacOS/imcp-server'
    );
    if (existsSync(serverPath)) {
      return serverPath;
    }
  }

  return null;
}
```

### Approach 2: User Configuration

Allow users to configure the path in plugin settings:

**File: `.claude-plugin/config.json`**

```json
{
  "imcpServerPath": "/Applications/iMCP.app/Contents/MacOS/imcp-server",
  "autoStart": true,
  "timeout": 30000
}
```

---

## 5. Connection Management

### 5.1 stdio Communication Pattern

The iMCP server uses stdio for JSON-RPC communication:

```javascript
import { spawn } from 'child_process';
import { EventEmitter } from 'events';

export class StdioConnection extends EventEmitter {
  constructor(serverPath) {
    super();
    this.serverPath = serverPath;
    this.process = null;
    this.messageBuffer = '';
  }

  async connect() {
    this.process = spawn(this.serverPath, [], {
      stdio: ['pipe', 'pipe', 'pipe']
    });

    // Handle stdout (responses)
    this.process.stdout.on('data', (data) => {
      this.messageBuffer += data.toString();
      this.processMessages();
    });

    // Handle stderr (logs)
    this.process.stderr.on('data', (data) => {
      console.error('[iMCP Server]', data.toString());
    });

    // Handle process exit
    this.process.on('exit', (code) => {
      this.emit('disconnect', code);
    });

    // Wait for server to be ready
    await this.waitForReady();
  }

  processMessages() {
    const lines = this.messageBuffer.split('\n');
    this.messageBuffer = lines.pop(); // Keep incomplete line

    for (const line of lines) {
      if (line.trim()) {
        try {
          const message = JSON.parse(line);
          this.emit('message', message);
        } catch (error) {
          console.error('Failed to parse message:', line, error);
        }
      }
    }
  }

  send(message) {
    const json = JSON.stringify(message) + '\n';
    this.process.stdin.write(json);
  }

  async disconnect() {
    if (this.process) {
      this.process.kill('SIGTERM');
      this.process = null;
    }
  }

  async waitForReady(timeout = 5000) {
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(new Error('Server ready timeout'));
      }, timeout);

      const checkReady = async () => {
        try {
          // Send initialize request
          this.send({
            jsonrpc: '2.0',
            id: 1,
            method: 'initialize',
            params: {
              protocolVersion: '2024-11-05',
              capabilities: {},
              clientInfo: {
                name: 'claude-code-imcp-plugin',
                version: '1.0.0'
              }
            }
          });

          // Wait for initialize response
          const handler = (message) => {
            if (message.id === 1 && message.result) {
              clearTimeout(timer);
              this.removeListener('message', handler);
              resolve();
            }
          };

          this.on('message', handler);
        } catch (error) {
          clearTimeout(timer);
          reject(error);
        }
      };

      checkReady();
    });
  }
}
```

### 5.2 Request/Response Handling

Implement promise-based request/response correlation:

```javascript
export class RequestManager {
  constructor(connection) {
    this.connection = connection;
    this.nextId = 1;
    this.pending = new Map();

    this.connection.on('message', (message) => {
      this.handleMessage(message);
    });
  }

  async request(method, params) {
    const id = this.nextId++;
    const promise = new Promise((resolve, reject) => {
      this.pending.set(id, { resolve, reject });
    });

    this.connection.send({
      jsonrpc: '2.0',
      id,
      method,
      params
    });

    return promise;
  }

  handleMessage(message) {
    if ('id' in message && this.pending.has(message.id)) {
      const { resolve, reject } = this.pending.get(message.id);
      this.pending.delete(message.id);

      if ('error' in message) {
        reject(new Error(message.error.message));
      } else {
        resolve(message.result);
      }
    }
  }
}
```

---

## 6. Error Handling Strategy

### 6.1 Error Categories and Recovery

| Error Type | Category | Recovery Strategy | User Message |
|------------|----------|-------------------|--------------|
| Connection refused | FATAL | Show setup instructions | "iMCP server not running. Please open iMCP.app from Applications." |
| Permission denied | FATAL | Show permission guide | "Service not activated. Open iMCP.app and enable {service}." |
| Tool not found | FATAL | Suggest update | "Tool not available. Update iMCP.app to latest version." |
| Timeout | RETRY | Exponential backoff | "Request timed out. Retrying..." |
| Invalid arguments | FATAL | Show argument requirements | "Invalid parameters: {details}" |
| Server crash | RETRY | Auto-restart | "Server crashed. Restarting..." |
| Rate limit | RETRY | Fixed delay | "Rate limited. Waiting..." |

### 6.2 Error Response Format

```javascript
{
  error: {
    code: 'PERMISSION_DENIED',
    message: 'Calendar service not activated',
    details: {
      service: 'calendar',
      reason: 'User has not granted Calendar access in iMCP.app'
    },
    recovery: {
      action: 'OPEN_IMCP_APP',
      instructions: [
        '1. Open iMCP.app from Applications',
        '2. Click the Calendar icon in the menu bar',
        '3. Click "Allow Full Access" when prompted'
      ]
    },
    retryable: false
  }
}
```

### 6.3 Graceful Degradation

```javascript
export class IMCPClient {
  async callToolWithFallback(toolName, params, fallback) {
    try {
      return await this.callTool(toolName, params);
    } catch (error) {
      if (this.errorHandler.isRetryable(error)) {
        return await this.errorHandler.withRetry(async () => {
          return await this.callTool(toolName, params);
        });
      } else {
        console.warn(`Tool ${toolName} failed, using fallback`, error);
        return fallback;
      }
    }
  }
}
```

---

## 7. Testing Strategy

### 7.1 Unit Tests

Test each component in isolation:

```javascript
// tests/test_imcp-client.mjs
import { describe, test, expect, beforeEach, afterEach } from 'vitest';
import { IMCPClient } from '../lib/imcp-client.mjs';

describe('IMCPClient', () => {
  let client;

  beforeEach(() => {
    client = new IMCPClient({
      serverPath: '/path/to/mock-server'
    });
  });

  afterEach(async () => {
    await client.disconnect();
  });

  test('should connect to server', async () => {
    await client.connect();
    expect(client.isConnected()).toBe(true);
  });

  test('should call tool successfully', async () => {
    await client.connect();
    const result = await client.callTool('weather_get_current', {
      location: 'San Francisco'
    });
    expect(result).toHaveProperty('temperature');
  });

  test('should handle connection errors', async () => {
    client = new IMCPClient({ serverPath: '/nonexistent' });
    await expect(client.connect()).rejects.toThrow();
  });
});
```

### 7.2 Integration Tests

Test against real iMCP server:

```javascript
// tests/integration/test_calendar.mjs
import { describe, test, expect, beforeAll, afterAll } from 'vitest';
import { IMCPClient } from '../../lib/imcp-client.mjs';
import { CalendarTools } from '../../tools/calendar.mjs';
import { findIMCPServer } from '../../lib/server-discovery.mjs';

describe('Calendar Integration', () => {
  let client;
  let calendar;

  beforeAll(async () => {
    const serverPath = findIMCPServer();
    if (!serverPath) {
      throw new Error('iMCP server not found');
    }

    client = new IMCPClient({ serverPath });
    await client.connect();
    calendar = new CalendarTools(client);
  });

  afterAll(async () => {
    await client.disconnect();
  });

  test('should list events', async () => {
    const events = await calendar.listEvents({
      startDate: '2025-11-30T00:00:00Z',
      endDate: '2025-12-07T23:59:59Z'
    });
    expect(Array.isArray(events)).toBe(true);
  });
});
```

---

## 8. Plugin Manifest

**File: `.claude-plugin/plugin.json`**

```json
{
  "name": "imcp",
  "version": "1.0.0",
  "description": "Integration with iMCP (macOS system services: Calendar, Contacts, Location, Maps, Messages, Reminders, Weather, Operator, Files, Capture)",
  "author": {
    "name": "Rick Farmer",
    "email": "rick@rickfarmer.com"
  },
  "license": "MIT",
  "main": "tools/index.mjs",
  "dependencies": {
    "zx": "^8.0.0"
  },
  "configuration": {
    "serverPath": {
      "type": "string",
      "description": "Path to imcp-server executable",
      "default": "auto"
    },
    "autoStart": {
      "type": "boolean",
      "description": "Automatically start server when plugin loads",
      "default": true
    },
    "timeout": {
      "type": "number",
      "description": "Request timeout in milliseconds",
      "default": 30000
    }
  },
  "skills": [
    "./SKILL.md"
  ],
  "tools": [
    "imcp_calendar_list_events",
    "imcp_calendar_create_event",
    "imcp_calendar_list_calendars",
    "imcp_contacts_search",
    "imcp_contacts_get_me",
    "imcp_location_get_current",
    "imcp_location_geocode",
    "imcp_location_reverse_geocode",
    "imcp_maps_search_places",
    "imcp_maps_get_directions",
    "imcp_maps_get_travel_time",
    "imcp_messages_get_history",
    "imcp_reminders_list",
    "imcp_reminders_create",
    "imcp_weather_get_current",
    "imcp_operator_list_apps",
    "imcp_operator_list_windows",
    "imcp_operator_screenshot_window",
    "imcp_operator_list_elements",
    "imcp_operator_press_element",
    "imcp_operator_input_text",
    "imcp_files_read",
    "imcp_capture_screen"
  ]
}
```

---

## 9. Key Design Decisions

### 9.1 Why Hybrid Architecture?

**Decision**: Wrap existing iMCP server instead of reimplementing services

**Rationale**:
- iMCP already handles all macOS permissions and system integration
- Reduces code duplication and maintenance burden
- Ensures consistency between Claude Desktop and Claude Code experiences
- Allows plugin to benefit from iMCP updates automatically

**Trade-offs**:
- Requires iMCP.app to be installed
- Adds process management complexity
- Introduces IPC latency (minimal with stdio)

### 9.2 Why stdio Transport?

**Decision**: Use stdio for MCP communication

**Rationale**:
- MCP standard transport mechanism
- Simple and reliable
- Works well with process spawning
- Minimal latency compared to network transports
- Already implemented in iMCP server

**Trade-offs**:
- Process must be managed (started/stopped)
- Can't share connection across multiple plugin instances
- Debugging requires stderr capture

### 9.3 Why Dynamic Server Discovery?

**Decision**: Auto-detect iMCP.app location instead of hardcoding

**Rationale**:
- Users may install in different locations
- Avoids brittle configuration
- Better user experience (just works)

**Trade-offs**:
- Adds complexity to initialization
- May fail if app is renamed or moved
- Fallback to manual configuration needed

### 9.4 Why Tool Wrapping?

**Decision**: Wrap each MCP tool in a JavaScript function

**Rationale**:
- Provides type checking and validation
- Enables better error handling
- Allows for argument transformation
- Makes tools easier to test
- Can add plugin-specific features (caching, logging)

**Trade-offs**:
- Adds layer of indirection
- Must maintain wrapper functions as iMCP evolves
- Increases code volume

---

## 10. Implementation Phases

### Phase 1: Core Infrastructure (Week 1)

1. Implement `lib/process-manager.mjs` - server lifecycle
2. Implement `lib/json-rpc.mjs` - JSON-RPC protocol
3. Implement `lib/error-handler.mjs` - error handling
4. Implement server discovery logic
5. Write unit tests for core components

**Success Criteria**:
- Can start/stop iMCP server programmatically
- Can send JSON-RPC requests and receive responses
- Error handling covers all failure modes
- Tests pass with >80% coverage

### Phase 2: MCP Client Library (Week 2)

1. Implement `lib/imcp-client.mjs` - main client class
2. Implement connection management and retry logic
3. Implement tool call API
4. Implement resource API (for Files service)
5. Write integration tests with real iMCP server

**Success Criteria**:
- Can call any iMCP tool through client API
- Connection automatically recovers from failures
- Client handles all error cases gracefully
- Integration tests pass against live server

### Phase 3: Tool Implementations (Week 3)

1. Implement all service tool wrappers
2. Add input validation and type checking
3. Write tests for each tool wrapper
4. Create skill documentation

**Success Criteria**:
- All iMCP tools are exposed as Claude Code tools
- Each tool has comprehensive documentation
- Tool tests pass with real server
- Skill documentation is complete

### Phase 4: Testing and Polish (Week 4)

1. End-to-end testing with Claude Code
2. Performance optimization
3. Error message refinement
4. Documentation and examples
5. Plugin packaging

**Success Criteria**:
- Plugin works seamlessly in Claude Code
- Response times < 2 seconds for typical requests
- Error messages are clear and actionable
- Documentation is comprehensive

---

## 11. Performance Considerations

### 11.1 Connection Pooling

For multiple concurrent requests, maintain single server connection:

```javascript
export class IMCPClient {
  constructor() {
    this.connection = null;
    this.requestQueue = [];
    this.processing = false;
  }

  async callTool(name, params) {
    return new Promise((resolve, reject) => {
      this.requestQueue.push({ name, params, resolve, reject });
      this.processQueue();
    });
  }

  async processQueue() {
    if (this.processing || this.requestQueue.length === 0) {
      return;
    }

    this.processing = true;

    while (this.requestQueue.length > 0) {
      const { name, params, resolve, reject } = this.requestQueue.shift();
      try {
        const result = await this.executeToolCall(name, params);
        resolve(result);
      } catch (error) {
        reject(error);
      }
    }

    this.processing = false;
  }
}
```

### 11.2 Response Caching

Cache read-only responses to reduce server load:

```javascript
export class CachedIMCPClient extends IMCPClient {
  constructor(options) {
    super(options);
    this.cache = new Map();
    this.cacheTTL = options.cacheTTL || 60000; // 1 minute default
  }

  async callTool(name, params) {
    if (this.isReadOnlyTool(name)) {
      const cacheKey = `${name}:${JSON.stringify(params)}`;
      const cached = this.cache.get(cacheKey);

      if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
        return cached.result;
      }

      const result = await super.callTool(name, params);
      this.cache.set(cacheKey, { result, timestamp: Date.now() });
      return result;
    }

    return await super.callTool(name, params);
  }

  isReadOnlyTool(name) {
    const readOnly = [
      'calendar_list_events',
      'contacts_search',
      'location_get_current',
      'weather_get_current'
    ];
    return readOnly.includes(name);
  }
}
```

---

## 12. Security Considerations

### 12.1 Process Isolation

The iMCP server runs as a separate process with full macOS permissions. The plugin must:
- Never expose raw stdin/stdout to user input
- Validate all tool parameters before sending to server
- Sanitize error messages to avoid information leakage
- Respect macOS permission model

### 12.2 Permission Checking

Before calling tools, verify service is activated:

```javascript
export class IMCPClient {
  async callTool(name, params) {
    const service = this.getServiceForTool(name);
    const isActivated = await this.checkServiceActivation(service);

    if (!isActivated) {
      throw new Error({
        code: 'PERMISSION_DENIED',
        message: `${service} service not activated`,
        recovery: {
          action: 'ACTIVATE_SERVICE',
          instructions: [
            `1. Open iMCP.app`,
            `2. Click the ${service} icon`,
            `3. Grant permission when prompted`
          ]
        }
      });
    }

    return await super.callTool(name, params);
  }
}
```

---

## 13. Future Enhancements

### 13.1 Service Status Monitoring

Add real-time monitoring of which services are active:

```javascript
export class ServiceMonitor {
  async getServiceStatus() {
    // Query iMCP server for service activation status
    return {
      calendar: { active: true, lastCheck: Date.now() },
      contacts: { active: true, lastCheck: Date.now() },
      messages: { active: false, reason: 'Permission not granted' }
    };
  }
}
```

### 13.2 Batch Tool Calls

Optimize multiple tool calls with batching:

```javascript
export class IMCPClient {
  async callToolBatch(requests) {
    // Send multiple tool calls in single JSON-RPC batch request
    const batchRequest = requests.map((req, index) => ({
      jsonrpc: '2.0',
      id: index,
      method: 'tools/call',
      params: { name: req.name, arguments: req.params }
    }));

    const responses = await this.connection.sendBatch(batchRequest);
    return responses.map(r => r.result);
  }
}
```

### 13.3 Event Streaming

Support streaming responses for long-running operations:

```javascript
export class IMCPClient {
  async *streamToolCall(name, params) {
    // Send request with streaming flag
    const id = await this.sendStreamingRequest(name, params);

    // Yield results as they arrive
    for await (const chunk of this.connection.streamResponse(id)) {
      yield chunk;
    }
  }
}
```

---

## Conclusion

This architecture provides a robust, maintainable integration between Claude Code and iMCP. The hybrid approach leverages the existing iMCP server while providing a clean JavaScript API for Claude Code tools.

Key strengths:
- **Reliability**: Comprehensive error handling and retry logic
- **Maintainability**: Clear separation of concerns with modular design
- **Extensibility**: Easy to add new tools as iMCP evolves
- **User Experience**: Auto-discovery and graceful degradation
- **Performance**: Connection pooling and caching where appropriate

The phased implementation plan allows for incremental development and testing, ensuring quality at each step.

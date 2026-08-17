# ofxLibwebsockets Architecture

`ofxLibwebsockets` is an openFrameworks addon that wraps the C-based [libwebsockets](https://libwebsockets.org/) library to provide an easy-to-use WebSocket client and server API in C++.

## Core Architecture

The architecture is built on top of a threaded core that manages the `libwebsockets` service loop, with distinct classes for clients, servers, connections, and protocols.

### 1. `ofxLibwebsockets::Reactor`
The base class for both `Server` and `Client`. It inherits from `ofThread` to manage the underlying `libwebsockets` context (`lws_context`) and polling loop on a separate thread. It manages a list of connections and protocols, and delegates the raw C callbacks to the appropriate C++ event handlers.

### 2. `ofxLibwebsockets::Server` (inherits `Reactor`)
Provides a simplified API to set up and manage a WebSocket server.
- Allows configuration via `ServerOptions` (port, SSL, keep-alive settings, document root).
- Provides methods to broadcast text (`send()`) or binary (`sendBinary()`) messages to all active connections.
- Automatically handles routing events to standard openFrameworks apps (via `addListener()`).

### 3. `ofxLibwebsockets::Client` (inherits `Reactor`)
Provides an API to connect to a remote WebSocket server.
- Configurable via `ClientOptions` (host, port, SSL, path).
- Supports auto-reconnect features.
- Maintains a single primary `Connection` to interface with the remote server.
- Provides `send()` and `sendBinary()` to communicate with the remote server.

### 4. `ofxLibwebsockets::Connection`
Wraps a single, active `lws` socket connection.
- Used by both `Client` (which has one connection) and `Server` (which manages many).
- Maintains internal thread-safe queues (`messages_text`, `messages_binary`) for outgoing data.
- Provides methods to get the client IP and send data specifically to this endpoint.

### 5. `ofxLibwebsockets::Protocol`
Encapsulates a specific WebSocket protocol and handles lifecycle events.
- Routes low-level `libwebsockets` callbacks (open, close, message, error) into openFrameworks events (`ofEvent`).
- Can manage custom allow/deny rules for client connections.
- Users generally bind to the protocol's events (e.g., `onmessageEvent`) to respond to WebSocket activity.

### 6. `ofxLibwebsockets::Event`
The data object passed to all event listeners (like `onMessage`).
- Contains a reference to the active `Connection`.
- Contains the received `message` (as a string) and attempts to automatically parse it into a JSON object (`ofJson`).
- Holds binary data buffers when receiving binary packets.

### 7. Utilities (`Util.h`/`Util.cpp`)
Contains the static C callbacks (`lws_callback`, `lws_client_callback`) required by `libwebsockets`. These intercept C-level events and map them back to the corresponding `Reactor`, `Protocol`, and `Connection` C++ instances.

## Event Flow and Usage

To use the addon, an openFrameworks app typically initializes a `Server` or `Client`, registers itself as a listener, and implements event listener methods:
- `onConnect(ofxLibwebsockets::Event& args)`
- `onOpen(ofxLibwebsockets::Event& args)`
- `onClose(ofxLibwebsockets::Event& args)`
- `onIdle(ofxLibwebsockets::Event& args)`
- `onMessage(ofxLibwebsockets::Event& args)`

When data is received, the static `lws_callback` routes it to the `Reactor`, which finds the correct `Protocol` and `Connection`, constructs an `Event` object, and triggers the `ofEvent` to notify the application. Outgoing messages are queued up in the `Connection` and sent out by the `Reactor` thread during the polling loop.

# WebSocket Patterns

Real-time communication patterns using WebSockets with reconnection logic, room management, and typed events.

## Core WebSocket Client

```typescript
import { EventEmitter } from 'eventemitter3';

type WebSocketStatus = 'connecting' | 'connected' | 'disconnected' | 'reconnecting';

interface WebSocketOptions {
  url: string;
  protocols?: string[];
  reconnectInterval?: number;
  maxReconnectAttempts?: number;
  heartbeatInterval?: number;
}

interface WebSocketMessage<T = unknown> {
  type: string;
  payload: T;
  id?: string;
  timestamp?: number;
}

class WebSocketClient extends EventEmitter {
  private ws: WebSocket | null = null;
  private status: WebSocketStatus = 'disconnected';
  private reconnectAttempts = 0;
  private reconnectTimer: ReturnType<typeof setTimeout> | null = null;
  private heartbeatTimer: ReturnType<typeof setInterval> | null = null;

  constructor(private options: WebSocketOptions) {
    super();
    this.options = {
      reconnectInterval: 3000,
      maxReconnectAttempts: 10,
      heartbeatInterval: 30000,
      ...options,
    };
  }

  connect(): void {
    if (this.status === 'connected' || this.status === 'connecting') return;
    this.setStatus('connecting');
    this.ws = new WebSocket(this.options.url, this.options.protocols);
    this.ws.onopen = this.handleOpen.bind(this);
    this.ws.onclose = this.handleClose.bind(this);
    this.ws.onerror = this.handleError.bind(this);
    this.ws.onmessage = this.handleMessage.bind(this);
  }

  disconnect(): void {
    this.clearTimers();
    this.reconnectAttempts = this.options.maxReconnectAttempts!;
    this.ws?.close(1000, 'Client disconnected');
  }

  send<T>(type: string, payload: T): void {
    if (this.status !== 'connected' || !this.ws) {
      throw new Error('WebSocket is not connected');
    }
    const message: WebSocketMessage<T> = {
      type,
      payload,
      id: crypto.randomUUID(),
      timestamp: Date.now(),
    };
    this.ws.send(JSON.stringify(message));
  }

  private handleOpen(): void {
    this.reconnectAttempts = 0;
    this.setStatus('connected');
    this.startHeartbeat();
    this.emit('connected');
  }

  private handleClose(event: CloseEvent): void {
    this.clearTimers();
    this.setStatus('disconnected');
    this.emit('disconnected', event);
    if (event.code !== 1000) this.scheduleReconnect();
  }

  private handleError(event: Event): void {
    this.emit('error', event);
  }

  private handleMessage(event: MessageEvent): void {
    try {
      const message: WebSocketMessage = JSON.parse(event.data);
      this.emit('message', message);
      this.emit(`message:${message.type}`, message.payload);
    } catch {
      this.emit('error', new Error('Failed to parse message'));
    }
  }

  private scheduleReconnect(): void {
    if (this.reconnectAttempts >= this.options.maxReconnectAttempts!) return;
    this.setStatus('reconnecting');
    this.reconnectAttempts++;
    const delay = Math.min(
      this.options.reconnectInterval! * Math.pow(1.5, this.reconnectAttempts - 1),
      30000
    );
    this.reconnectTimer = setTimeout(() => this.connect(), delay);
    this.emit('reconnecting', { attempt: this.reconnectAttempts, delay });
  }

  private startHeartbeat(): void {
    this.heartbeatTimer = setInterval(() => {
      if (this.status === 'connected') this.ws?.send(JSON.stringify({ type: 'ping' }));
    }, this.options.heartbeatInterval);
  }

  private clearTimers(): void {
    if (this.reconnectTimer) clearTimeout(this.reconnectTimer);
    if (this.heartbeatTimer) clearInterval(this.heartbeatTimer);
  }

  private setStatus(status: WebSocketStatus): void {
    this.status = status;
    this.emit('status', status);
  }

  getStatus(): WebSocketStatus {
    return this.status;
  }
}
```

## React Hook

```typescript
function useWebSocket<Events extends Record<string, unknown>>(url: string) {
  const clientRef = useRef<WebSocketClient | null>(null);
  const [status, setStatus] = useState<WebSocketStatus>('disconnected');

  useEffect(() => {
    const client = new WebSocketClient({ url });
    clientRef.current = client;
    client.on('status', setStatus);
    client.connect();
    return () => client.disconnect();
  }, [url]);

  const send = useCallback(<K extends keyof Events>(type: K, payload: Events[K]) => {
    clientRef.current?.send(type as string, payload);
  }, []);

  const on = useCallback(<K extends keyof Events>(type: K, handler: (payload: Events[K]) => void) => {
    clientRef.current?.on(`message:${String(type)}`, handler);
    return () => clientRef.current?.off(`message:${String(type)}`, handler);
  }, []);

  return { status, send, on };
}
```

## Server-Side Room Manager (Node.js)

```typescript
import { WebSocketServer, WebSocket } from 'ws';

interface Client {
  id: string;
  ws: WebSocket;
  rooms: Set<string>;
  userId?: string;
}

class RoomManager {
  private clients = new Map<string, Client>();
  private rooms = new Map<string, Set<string>>();

  addClient(id: string, ws: WebSocket, userId?: string): Client {
    const client: Client = { id, ws, rooms: new Set(), userId };
    this.clients.set(id, client);
    return client;
  }

  removeClient(id: string): void {
    const client = this.clients.get(id);
    if (!client) return;
    client.rooms.forEach(room => this.leaveRoom(id, room));
    this.clients.delete(id);
  }

  joinRoom(clientId: string, room: string): void {
    if (!this.rooms.has(room)) this.rooms.set(room, new Set());
    this.rooms.get(room)!.add(clientId);
    this.clients.get(clientId)?.rooms.add(room);
  }

  leaveRoom(clientId: string, room: string): void {
    this.rooms.get(room)?.delete(clientId);
    if (this.rooms.get(room)?.size === 0) this.rooms.delete(room);
    this.clients.get(clientId)?.rooms.delete(room);
  }

  broadcast<T>(room: string, type: string, payload: T, excludeId?: string): void {
    const message = JSON.stringify({ type, payload, timestamp: Date.now() });
    this.rooms.get(room)?.forEach(clientId => {
      if (clientId === excludeId) return;
      const client = this.clients.get(clientId);
      if (client?.ws.readyState === WebSocket.OPEN) client.ws.send(message);
    });
  }

  getRoomSize(room: string): number {
    return this.rooms.get(room)?.size ?? 0;
  }
}
```

## Usage

```typescript
// Client
const { status, send, on } = useWebSocket<{
  'chat:message': { text: string; userId: string };
  'user:join': { userId: string };
}>('wss://api.example.com/ws');

useEffect(() => on('chat:message', (msg) => console.log(msg)), [on]);
send('chat:message', { text: 'Hello', userId: '123' });

// Server
const manager = new RoomManager();
wss.on('connection', (ws) => {
  const id = crypto.randomUUID();
  const client = manager.addClient(id, ws);
  ws.on('message', (data) => {
    const { type, payload } = JSON.parse(data.toString());
    if (type === 'room:join') {
      manager.joinRoom(id, payload.room);
      manager.broadcast(payload.room, 'user:join', { userId: client.userId });
    }
  });
  ws.on('close', () => manager.removeClient(id));
});
```

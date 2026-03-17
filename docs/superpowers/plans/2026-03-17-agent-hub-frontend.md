# Agent Hub Frontend Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the Discord-only Agent Bridge View with a unified Agent Hub that supports all transport types and includes a built-in agent chat client.

**Architecture:** WebSocket-first. UI WS (`/ws`) for session/log broadcasts, Agent WS (`/ws/agent`) for chat. `useAgentWs` hook lives at AgentHubView level (survives tab switches). Sessions panel fetches initial state via HTTP, then updates incrementally via WS events.

**Tech Stack:** Next.js 16, React 19, shadcn/ui (Tabs, Card, Badge, Input, Switch, ScrollArea), Tailwind CSS 4, lucide-react icons.

**Spec:** `docs/superpowers/specs/2026-03-17-agent-hub-frontend-design.md`

---

## File Map

```
Changes overview (13 files total):

BACKEND (3 modified):
  src/agent-session-manager.js  — wire session events → _broadcast (status_change, log)
  src/ws-agent.js               — accept msg.transport in connect handler
  src/routes/agent-api.js       — add GET/PUT /api/agent-api/settings

FRONTEND — New (8 files):
  frontend/lib/agent-utils.ts           — shared constants (STATE_CONFIG, LOG_COLORS, etc.)
  frontend/lib/agent-api.ts             — TypeScript types + HTTP helpers
  frontend/lib/config.ts                — add getAgentWsUrl() helper (modify existing)
  frontend/hooks/use-agent-ws.ts        — Agent WS state machine hook
  frontend/components/agent-hub-view.tsx — main tabbed container
  frontend/components/agent-hub/sessions-panel.tsx
  frontend/components/agent-hub/chat-panel.tsx
  frontend/components/agent-hub/config-panel.tsx
  frontend/components/agent-hub/logs-panel.tsx

FRONTEND — Modified (2 files):
  frontend/app/page.tsx           — swap showBridge → showAgentHub
  frontend/components/app-sidebar.tsx — rename Bridge → Agent Hub
```

---

## Task 1: Backend — Extend SessionManager broadcasts

**Files:**
- Modify: `src/agent-session-manager.js`

The SessionManager currently only broadcasts `session_created` and `session_destroyed`. We need to wire `AgentSession` events so the UI can track status changes and logs in real-time.

- [ ] **Step 1: Add event wiring in createSession()**

In `src/agent-session-manager.js`, inside `createSession()`, after the existing `session.on('destroyed', ...)` listener (line 50-53), add three more event listeners:

```javascript
    // --- Add after line 53 (after the 'destroyed' listener) ---

    session.on('status_change', (data) => {
        _broadcast('session_status_change', {
            sessionId: id,
            state: data.state,
            previousState: data.previousState,
            isBusy: data.isBusy,
        });
    });

    session.on('busy_change', (data) => {
        _broadcast('session_status_change', {
            sessionId: id,
            isBusy: data.isBusy,
        });
    });

    session.on('log', (data) => {
        _broadcast('session_log', {
            sessionId: id,
            transport: session.transport,
            logType: data.type,
            message: data.message,
            timestamp: data.ts,
        });
    });
```

- [ ] **Step 2: Verify server starts without error**

Run: `node src/server.js` (or restart if already running)
Expected: Server starts on port 3500 with no errors. Existing `session_created`/`session_destroyed` broadcasts still work.

- [ ] **Step 3: Commit**

```bash
git add src/agent-session-manager.js
git commit -m "feat(backend): broadcast session status_change and log events via UI WS"
```

---

## Task 2: Backend — Accept transport field in ws-agent.js

**Files:**
- Modify: `src/ws-agent.js`

Currently the connect handler hardcodes `transport: 'websocket'`. The Chat panel needs to identify itself as `'websocket-ui'`.

- [ ] **Step 1: Update connect handler to use msg.transport**

In `src/ws-agent.js`, line 58, change the `transport` value in `sessionManager.createSession()`:

```javascript
// Change line 58 from:
//     transport: 'websocket',
// To:
                        transport: msg.transport || 'websocket',
```

- [ ] **Step 2: Verify no regression**

Run: `node src/server.js`
Expected: Server starts. Existing WS agent connections that don't send `transport` still default to `'websocket'`.

- [ ] **Step 3: Commit**

```bash
git add src/ws-agent.js
git commit -m "feat(backend): accept optional transport field in ws-agent connect message"
```

---

## Task 3: Backend — Add Agent API settings endpoints

**Files:**
- Modify: `src/routes/agent-api.js`

Add two endpoints for the Config panel to read/write Agent API settings. The persistence functions (`getAgentApiSettings`/`saveAgentApiSettings`) already exist in `src/config.js`.

- [ ] **Step 1: Add settings endpoints**

At the end of `src/routes/agent-api.js`, before `module.exports`, add:

```javascript
    // ── Agent API Settings ──────────────────────────────────────────────────

    app.get('/api/agent-api/settings', (req, res) => {
        try {
            const { getAgentApiSettings } = require('../config');
            res.json(getAgentApiSettings());
        } catch (e) {
            res.status(500).json({ error: e.message });
        }
    });

    app.put('/api/agent-api/settings', (req, res) => {
        try {
            const { saveAgentApiSettings } = require('../config');
            const body = req.body || {};

            // Validate types
            if (body.maxConcurrentSessions != null && (typeof body.maxConcurrentSessions !== 'number' || body.maxConcurrentSessions < 1 || body.maxConcurrentSessions > 20)) {
                return res.status(400).json({ error: 'maxConcurrentSessions must be 1-20' });
            }
            if (body.sessionTimeoutMs != null && (typeof body.sessionTimeoutMs !== 'number' || body.sessionTimeoutMs < 60000 || body.sessionTimeoutMs > 86400000)) {
                return res.status(400).json({ error: 'sessionTimeoutMs must be 60000-86400000' });
            }
            if (body.defaultStepSoftLimit != null && (typeof body.defaultStepSoftLimit !== 'number' || body.defaultStepSoftLimit < 10 || body.defaultStepSoftLimit > 10000)) {
                return res.status(400).json({ error: 'defaultStepSoftLimit must be 10-10000' });
            }

            const updated = saveAgentApiSettings(body);

            // Apply to running SessionManager immediately
            sessionManager.configure(body);

            res.json(updated);
        } catch (e) {
            res.status(500).json({ error: e.message });
        }
    });
```

- [ ] **Step 2: Test endpoints with curl**

```bash
curl http://localhost:3500/api/agent-api/settings
# Expected: {"enabled":true,"maxConcurrentSessions":5,"sessionTimeoutMs":1800000,"defaultStepSoftLimit":500}

curl -X PUT http://localhost:3500/api/agent-api/settings -H "Content-Type: application/json" -d '{"maxConcurrentSessions":3}'
# Expected: {"enabled":true,"maxConcurrentSessions":3,...}
```

- [ ] **Step 3: Commit**

```bash
git add src/routes/agent-api.js
git commit -m "feat(backend): add GET/PUT /api/agent-api/settings endpoints"
```

---

## Task 4: Frontend — Shared constants and types

**Files:**
- Create: `frontend/lib/agent-utils.ts`
- Create: `frontend/lib/agent-api.ts`
- Modify: `frontend/lib/config.ts`

Extract reusable constants from `agent-bridge-view.tsx` and define TypeScript types for the Agent API.

- [ ] **Step 1: Create `frontend/lib/agent-utils.ts`**

```typescript
// === Agent Hub shared constants ===
// Extracted from agent-bridge-view.tsx for reuse across Agent Hub panels.

import {
    Bot, RefreshCw, ArrowRight, ArrowLeft, AlertCircle,
    Wifi, WifiOff, Globe, MessageSquare,
} from 'lucide-react';

// ── Session state → visual config ───────────────────────────────────────

export const SESSION_STATE_CONFIG = {
    IDLE: { color: 'text-muted-foreground/50', dot: 'bg-muted-foreground/40', label: 'Offline', icon: WifiOff },
    ACTIVE: { color: 'text-emerald-400', dot: 'bg-emerald-400', label: 'Active', icon: Wifi },
    TRANSITIONING: { color: 'text-amber-400', dot: 'bg-amber-400 animate-pulse', label: 'Transitioning', icon: RefreshCw },
} as const;

// ── Transport badge colors ──────────────────────────────────────────────

export const TRANSPORT_CONFIG: Record<string, { label: string; color: string; bg: string }> = {
    discord: { label: 'Discord', color: 'text-violet-400', bg: 'bg-violet-400/10' },
    websocket: { label: 'WS', color: 'text-emerald-400', bg: 'bg-emerald-400/10' },
    'websocket-ui': { label: 'UI', color: 'text-orange-400', bg: 'bg-orange-400/10' },
    http: { label: 'HTTP', color: 'text-sky-400', bg: 'bg-sky-400/10' },
    unknown: { label: '?', color: 'text-muted-foreground', bg: 'bg-muted/10' },
};

// ── Log entry styling ───────────────────────────────────────────────────

export const LOG_COLORS: Record<string, string> = {
    system: 'text-muted-foreground/60',
    from_antigravity: 'text-sky-400',
    from_pi: 'text-violet-400',
    from_agent: 'text-sky-400',
    error: 'text-red-400',
};

export const LOG_ICONS: Record<string, typeof Bot> = {
    system: RefreshCw,
    from_antigravity: ArrowRight,
    from_pi: ArrowLeft,
    from_agent: ArrowRight,
    error: AlertCircle,
};

// ── Timestamp formatter ─────────────────────────────────────────────────

export function formatTimestamp(ts: number): string {
    return new Date(ts).toLocaleTimeString('vi-VN', {
        hour: '2-digit', minute: '2-digit', second: '2-digit',
    });
}
```

- [ ] **Step 2: Create `frontend/lib/agent-api.ts`**

```typescript
// === Agent API types and HTTP helpers ===

import { API_BASE } from './config';
import { authHeaders } from './auth';

// ── Types ───────────────────────────────────────────────────────────────

export type AgentWsState = 'disconnected' | 'connecting' | 'connected' | 'busy' | 'reconnecting' | 'error';

export interface AgentSessionInfo {
    id: string;
    state: 'IDLE' | 'ACTIVE' | 'TRANSITIONING';
    cascadeId: string | null;
    cascadeIdShort: string;
    stepCount: number;
    stepSoftLimit: number;
    isBusy: boolean;
    workspace: string;
    transport: string;
    lastActivity: number;
}

export interface AgentMessage {
    id: string;
    role: 'user' | 'agent' | 'system';
    content: string;
    timestamp: number;
    stepIndex?: number;
    stepCount?: number;
    stepType?: string;
}

export interface AgentApiSettings {
    enabled: boolean;
    maxConcurrentSessions: number;
    sessionTimeoutMs: number;
    defaultStepSoftLimit: number;
}

export interface BridgeSettings {
    discordBotToken: string;
    discordChannelId: string;
    discordGuildId: string;
    stepSoftLimit: number;
    allowedBotIds: string[];
    autoStart: boolean;
}

export interface BridgeStatus {
    state: 'IDLE' | 'ACTIVE' | 'TRANSITIONING';
    cascadeId: string | null;
    cascadeIdShort: string;
    stepCount: number;
    softLimit: number;
    log: Array<{ type: string; message: string; ts: number }>;
}

export interface AgentLogEntry {
    sessionId: string;
    transport: string;
    logType: string;
    message: string;
    timestamp: number;
}

// ── HTTP Helpers ────────────────────────────────────────────────────────

export async function fetchAgentSessions(): Promise<AgentSessionInfo[]> {
    const res = await fetch(`${API_BASE}/api/agent/sessions`, { headers: authHeaders() });
    if (!res.ok) throw new Error(`Failed to fetch sessions: ${res.status}`);
    return res.json();
}

export async function destroyAgentSession(sessionId: string): Promise<void> {
    const res = await fetch(`${API_BASE}/api/agent/${sessionId}`, {
        method: 'DELETE',
        headers: authHeaders(),
    });
    if (!res.ok) throw new Error(`Failed to destroy session: ${res.status}`);
}

export async function fetchAgentApiSettings(): Promise<AgentApiSettings> {
    const res = await fetch(`${API_BASE}/api/agent-api/settings`, { headers: authHeaders() });
    if (!res.ok) throw new Error(`Failed to fetch settings: ${res.status}`);
    return res.json();
}

export async function saveAgentApiSettings(settings: Partial<AgentApiSettings>): Promise<AgentApiSettings> {
    const res = await fetch(`${API_BASE}/api/agent-api/settings`, {
        method: 'PUT',
        headers: authHeaders(),
        body: JSON.stringify(settings),
    });
    if (!res.ok) {
        const err = await res.json().catch(() => ({ error: 'Save failed' }));
        throw new Error(err.error || `Save failed: ${res.status}`);
    }
    return res.json();
}

export async function fetchBridgeSettings(): Promise<BridgeSettings> {
    const res = await fetch(`${API_BASE}/api/agent-bridge/settings`, { headers: authHeaders() });
    if (!res.ok) throw new Error(`Failed to fetch bridge settings: ${res.status}`);
    return res.json();
}

export async function saveBridgeSettings(settings: Partial<BridgeSettings>): Promise<BridgeSettings> {
    const res = await fetch(`${API_BASE}/api/agent-bridge/settings`, {
        method: 'POST',
        headers: authHeaders(),
        body: JSON.stringify(settings),
    });
    if (!res.ok) throw new Error(`Save failed: ${res.status}`);
    return res.json();
}

export async function fetchBridgeStatus(): Promise<BridgeStatus> {
    const res = await fetch(`${API_BASE}/api/agent-bridge/status`, { headers: authHeaders() });
    if (!res.ok) throw new Error(`Failed to fetch bridge status: ${res.status}`);
    return res.json();
}

export async function startBridge(config: Record<string, unknown> = {}): Promise<BridgeStatus> {
    const res = await fetch(`${API_BASE}/api/agent-bridge/start`, {
        method: 'POST',
        headers: authHeaders(),
        body: JSON.stringify(config),
    });
    const data = await res.json();
    if (!data.ok && data.error) throw new Error(data.error);
    return data;
}

export async function stopBridge(): Promise<void> {
    await fetch(`${API_BASE}/api/agent-bridge/stop`, { method: 'POST', headers: authHeaders() });
}
```

- [ ] **Step 3: Add `getAgentWsUrl()` to `frontend/lib/config.ts`**

Add this function at the bottom of the file, before the `export const WS_URL = '';` line:

```typescript
/**
 * Agent WebSocket URL — derived from UI WS URL by appending /agent path.
 * Example: ws://localhost:3500 → ws://localhost:3500/ws/agent
 */
export async function getAgentWsUrl(): Promise<string> {
    const uiWsUrl = await getWsUrl();
    // getWsUrl() returns e.g. "ws://localhost:3500" (no path)
    // Agent WS is at /ws/agent
    return `${uiWsUrl}/ws/agent`;
}
```

- [ ] **Step 4: Verify TypeScript compiles**

Run: `cd frontend && npx next build` (or just `npx tsc --noEmit`)
Expected: No type errors from new files.

- [ ] **Step 5: Commit**

```bash
git add frontend/lib/agent-utils.ts frontend/lib/agent-api.ts frontend/lib/config.ts
git commit -m "feat(frontend): add Agent Hub shared types, API helpers, and constants"
```

---

## Task 5: Frontend — useAgentWs hook

**Files:**
- Create: `frontend/hooks/use-agent-ws.ts`

This is the most complex piece — a state machine hook that manages the Agent WS (`/ws/agent`) connection lifecycle for the Chat panel. It lives at `AgentHubView` level so it survives tab switches.

- [ ] **Step 1: Create `frontend/hooks/use-agent-ws.ts`**

```typescript
'use client';

import { useState, useCallback, useRef, useEffect } from 'react';
import { getAgentWsUrl } from '@/lib/config';
import { authWsUrl } from '@/lib/auth';
import type { AgentWsState, AgentMessage } from '@/lib/agent-api';

// ── State machine ──────────────────────────────────────────────────────
//
//   disconnected ──connect()──→ connecting ──'connected' msg──→ connected
//       ↑                         │ error                        │ send()
//       │                         ↓                              ↓
//       │                    reconnecting ←── WS close ────── busy
//       │                         │ 3 retries                    │ 'response'
//       │                         ↓                              ↓
//       ←────────────────────── error                     connected
//       ←────── disconnect()

const MAX_RETRIES = 3;
const BACKOFF = [1000, 2000, 4000];

let _msgIdCounter = 0;
function nextMsgId(): string {
    return `msg-${Date.now()}-${++_msgIdCounter}`;
}

export interface UseAgentWsReturn {
    state: AgentWsState;
    sessionId: string | null;
    cascadeId: string | null;
    workspace: string | null;
    messages: AgentMessage[];
    error: string | null;
    connect: (workspace: string) => Promise<void>;
    send: (text: string) => void;
    accept: () => void;
    reject: () => void;
    newCascade: () => void;
    disconnect: () => void;
}

export function useAgentWs(): UseAgentWsReturn {
    const [state, setState] = useState<AgentWsState>('disconnected');
    const [sessionId, setSessionId] = useState<string | null>(null);
    const [cascadeId, setCascadeId] = useState<string | null>(null);
    const [workspace, setWorkspace] = useState<string | null>(null);
    const [messages, setMessages] = useState<AgentMessage[]>([]);
    const [error, setError] = useState<string | null>(null);

    const wsRef = useRef<WebSocket | null>(null);
    const retryCountRef = useRef(0);
    const retryTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
    const workspaceRef = useRef<string | null>(null);
    const mountedRef = useRef(true);

    // Keep workspace ref in sync
    useEffect(() => { workspaceRef.current = workspace; }, [workspace]);

    // Cleanup on unmount
    useEffect(() => {
        mountedRef.current = true;
        return () => {
            mountedRef.current = false;
            if (retryTimerRef.current) clearTimeout(retryTimerRef.current);
            if (wsRef.current) {
                wsRef.current.onclose = null; // prevent reconnect on unmount
                wsRef.current.close();
                wsRef.current = null;
            }
        };
    }, []);

    const addMessage = useCallback((msg: AgentMessage) => {
        setMessages(prev => [...prev, msg]);
    }, []);

    const addSystemMessage = useCallback((content: string) => {
        addMessage({ id: nextMsgId(), role: 'system', content, timestamp: Date.now() });
    }, [addMessage]);

    const wsSend = useCallback((data: Record<string, unknown>) => {
        if (wsRef.current?.readyState === WebSocket.OPEN) {
            wsRef.current.send(JSON.stringify(data));
        }
    }, []);

    const handleMessage = useCallback((event: MessageEvent) => {
        if (!mountedRef.current) return;
        let data: Record<string, unknown>;
        try {
            data = JSON.parse(event.data as string);
        } catch { return; }

        switch (data.type) {
            case 'connected':
                setSessionId(data.sessionId as string);
                setCascadeId(data.cascadeId as string | null);
                setWorkspace(data.workspace as string);
                setState('connected');
                retryCountRef.current = 0;
                setError(null);
                addSystemMessage(`Connected to ${data.workspace} (session ${(data.sessionId as string).substring(0, 8)})`);
                break;

            case 'response':
                addMessage({
                    id: nextMsgId(),
                    role: 'agent',
                    content: data.text as string,
                    timestamp: Date.now(),
                    stepIndex: data.stepIndex as number | undefined,
                    stepCount: data.stepCount as number | undefined,
                    stepType: data.stepType as string | undefined,
                });
                setState('connected');
                break;

            case 'busy':
                if (data.isBusy) {
                    setState('busy');
                } else {
                    setState('connected');
                }
                break;

            case 'cascade_transition':
                setCascadeId(data.newId as string);
                addSystemMessage(
                    `Cascade transitioned: ${(data.oldShort as string) || '?'} → ${(data.newShort as string) || '?'}${data.reason ? ` (${data.reason})` : ''}`
                );
                break;

            case 'status_change':
                // Backend state change — informational
                break;

            case 'step_limit_warning':
                addSystemMessage(
                    `⚠️ Approaching step limit: ${data.stepCount}/${data.softLimit}`
                );
                break;

            case 'busy_rejected':
                addSystemMessage('⚠️ Agent is busy processing a previous message');
                setState('connected');
                break;

            case 'error':
                setError(data.message as string);
                addSystemMessage(`❌ Error: ${data.message}`);
                break;

            case 'disconnected':
                setState('disconnected');
                setSessionId(null);
                setCascadeId(null);
                addSystemMessage('Disconnected');
                break;

            case 'workspace_switched':
                setCascadeId(data.cascadeId as string | null);
                addSystemMessage(`Workspace switched to ${data.workspace}`);
                break;

            case 'accepted':
                addSystemMessage('✅ Code changes accepted');
                break;

            case 'rejected':
                addSystemMessage('❌ Code changes rejected');
                break;
        }
    }, [addMessage, addSystemMessage]);

    const attemptConnect = useCallback(async (ws_workspace: string) => {
        try {
            const agentUrl = await getAgentWsUrl();
            const url = authWsUrl(agentUrl);
            const ws = new WebSocket(url);
            wsRef.current = ws;

            ws.onopen = () => {
                if (!mountedRef.current) { ws.close(); return; }
                // Send connect message with transport identification
                ws.send(JSON.stringify({
                    type: 'connect',
                    workspace: ws_workspace,
                    transport: 'websocket-ui',
                }));
            };

            ws.onmessage = handleMessage;

            ws.onclose = () => {
                if (!mountedRef.current) return;
                wsRef.current = null;
                setSessionId(null);
                setCascadeId(null);

                // Attempt reconnect if we were connected (not a manual disconnect)
                setState(prev => {
                    if (prev === 'disconnected') return prev; // manual disconnect
                    if (retryCountRef.current >= MAX_RETRIES) {
                        setError('Connection lost. Reconnect failed after 3 attempts.');
                        return 'error';
                    }
                    const delay = BACKOFF[retryCountRef.current] || 4000;
                    retryCountRef.current++;
                    retryTimerRef.current = setTimeout(() => {
                        if (mountedRef.current && workspaceRef.current) {
                            attemptConnect(workspaceRef.current);
                        }
                    }, delay);
                    addSystemMessage(`Connection lost. Reconnecting (${retryCountRef.current}/${MAX_RETRIES})...`);
                    return 'reconnecting';
                });
            };

            ws.onerror = () => {
                // onclose will fire after this
            };
        } catch (e) {
            if (!mountedRef.current) return;
            setError(e instanceof Error ? e.message : 'Connection failed');
            setState('error');
        }
    }, [handleMessage, addSystemMessage]);

    const connect = useCallback(async (ws_workspace: string) => {
        // Clean up existing connection
        if (wsRef.current) {
            wsRef.current.onclose = null;
            wsRef.current.close();
            wsRef.current = null;
        }
        if (retryTimerRef.current) {
            clearTimeout(retryTimerRef.current);
            retryTimerRef.current = null;
        }
        retryCountRef.current = 0;
        setError(null);
        setState('connecting');
        setWorkspace(ws_workspace);
        workspaceRef.current = ws_workspace;
        await attemptConnect(ws_workspace);
    }, [attemptConnect]);

    const send = useCallback((text: string) => {
        if (state === 'busy') {
            addSystemMessage('⚠️ Agent is busy — wait for current response');
            return;
        }
        if (state !== 'connected') return;

        addMessage({
            id: nextMsgId(),
            role: 'user',
            content: text,
            timestamp: Date.now(),
        });
        setState('busy');
        wsSend({ type: 'send', message: text });
    }, [state, addMessage, addSystemMessage, wsSend]);

    const accept = useCallback(() => { wsSend({ type: 'accept' }); }, [wsSend]);
    const reject = useCallback(() => { wsSend({ type: 'reject' }); }, [wsSend]);

    const newCascade = useCallback(() => {
        if (workspaceRef.current) {
            wsSend({ type: 'switch_workspace', workspace: workspaceRef.current });
        }
    }, [wsSend]);

    const disconnect = useCallback(() => {
        if (retryTimerRef.current) {
            clearTimeout(retryTimerRef.current);
            retryTimerRef.current = null;
        }
        retryCountRef.current = 0;
        if (wsRef.current) {
            wsSend({ type: 'disconnect' });
            wsRef.current.onclose = null;
            wsRef.current.close();
            wsRef.current = null;
        }
        setState('disconnected');
        setSessionId(null);
        setCascadeId(null);
        setError(null);
        addSystemMessage('Disconnected');
    }, [wsSend, addSystemMessage]);

    return {
        state, sessionId, cascadeId, workspace, messages, error,
        connect, send, accept, reject, newCascade, disconnect,
    };
}
```

- [ ] **Step 2: Verify TypeScript compiles**

Run: `cd frontend && npx tsc --noEmit`
Expected: No type errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/hooks/use-agent-ws.ts
git commit -m "feat(frontend): add useAgentWs hook with explicit state machine"
```

---

## Task 6: Frontend — Sessions Panel

**Files:**
- Create: `frontend/components/agent-hub/sessions-panel.tsx`

Displays all active agent sessions as cards. Fetches initial list via HTTP on mount, then updates incrementally via WS `agent_sessions` events.

- [ ] **Step 1: Create directory**

```bash
mkdir -p frontend/components/agent-hub
```

- [ ] **Step 2: Create `frontend/components/agent-hub/sessions-panel.tsx`**

```typescript
'use client';

import { useState, useEffect, useCallback, memo } from 'react';
import { cn } from '@/lib/utils';
import { wsService } from '@/lib/ws-service';
import { Card, CardContent } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Trash2, RefreshCw, Loader2, AlertCircle, Users } from 'lucide-react';
import { SESSION_STATE_CONFIG, TRANSPORT_CONFIG, formatTimestamp } from '@/lib/agent-utils';
import { fetchAgentSessions, destroyAgentSession } from '@/lib/agent-api';
import type { AgentSessionInfo } from '@/lib/agent-api';

// ── Session Card (memoized) ─────────────────────────────────────────────

const SessionCard = memo(function SessionCard({
    session,
    onDestroy,
}: {
    session: AgentSessionInfo;
    onDestroy: (id: string) => void;
}) {
    const transport = TRANSPORT_CONFIG[session.transport] || TRANSPORT_CONFIG.unknown;
    const stateConf = SESSION_STATE_CONFIG[session.state] || SESSION_STATE_CONFIG.IDLE;
    const pct = session.stepSoftLimit > 0
        ? Math.min((session.stepCount / session.stepSoftLimit) * 100, 100)
        : 0;

    // Derive status dot based on state + busy
    let dotClass = stateConf.dot;
    let statusLabel = stateConf.label;
    if (session.state === 'ACTIVE' && session.isBusy) {
        dotClass = 'bg-amber-400 animate-pulse';
        statusLabel = 'Busy';
    }

    const elapsed = Date.now() - session.lastActivity;
    const relativeTime = elapsed < 60000 ? 'just now'
        : elapsed < 3600000 ? `${Math.floor(elapsed / 60000)}m ago`
        : `${Math.floor(elapsed / 3600000)}h ago`;

    const [confirming, setConfirming] = useState(false);

    return (
        <Card className="bg-muted/5 border-border/20">
            <CardContent className="p-3 space-y-2">
                {/* Header: ID + Transport badge + Status dot */}
                <div className="flex items-center justify-between">
                    <div className="flex items-center gap-2">
                        <span className={cn('w-2 h-2 rounded-full shrink-0', dotClass)} />
                        <span className="text-xs font-mono text-foreground/70">
                            {session.id.substring(0, 4)}…{session.id.substring(session.id.length - 4)}
                        </span>
                        <Badge variant="outline" className={cn('text-[9px] px-1.5 py-0', transport.color, transport.bg)}>
                            {transport.label}
                        </Badge>
                    </div>
                    <span className={cn('text-[10px]', stateConf.color)}>{statusLabel}</span>
                </div>

                {/* Workspace + cascade */}
                <div className="flex items-center justify-between text-[10px] text-muted-foreground/60">
                    <span>{session.workspace}</span>
                    {session.cascadeIdShort && session.cascadeIdShort !== '--------' && (
                        <span className="font-mono">#{session.cascadeIdShort}</span>
                    )}
                </div>

                {/* Step progress */}
                <div className="space-y-1">
                    <div className="flex justify-between text-[9px] text-muted-foreground/50">
                        <span>{session.stepCount}/{session.stepSoftLimit} steps</span>
                        <span>{relativeTime}</span>
                    </div>
                    <div className="w-full h-1 rounded-full bg-muted/20 overflow-hidden">
                        <div
                            className={cn(
                                'h-full rounded-full transition-all duration-500',
                                pct > 85 ? 'bg-red-400/70' : pct > 60 ? 'bg-amber-400/70' : 'bg-emerald-400/70'
                            )}
                            style={{ width: `${pct}%` }}
                        />
                    </div>
                </div>

                {/* Destroy action */}
                <div className="flex justify-end">
                    {confirming ? (
                        <div className="flex items-center gap-1.5">
                            <span className="text-[10px] text-red-400">Destroy?</span>
                            <Button size="sm" variant="destructive" className="h-6 text-[10px] px-2"
                                onClick={() => { onDestroy(session.id); setConfirming(false); }}>
                                Yes
                            </Button>
                            <Button size="sm" variant="ghost" className="h-6 text-[10px] px-2"
                                onClick={() => setConfirming(false)}>
                                No
                            </Button>
                        </div>
                    ) : (
                        <Button variant="ghost" size="icon" className="h-6 w-6"
                            onClick={() => setConfirming(true)} title="Destroy session">
                            <Trash2 className="h-3 w-3 text-muted-foreground/30 hover:text-red-400" />
                        </Button>
                    )}
                </div>
            </CardContent>
        </Card>
    );
});

// ── Main Panel ──────────────────────────────────────────────────────────

export function AgentSessionsPanel() {
    const [sessions, setSessions] = useState<AgentSessionInfo[]>([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState<string | null>(null);

    const loadSessions = useCallback(async () => {
        try {
            setError(null);
            const data = await fetchAgentSessions();
            setSessions(data);
        } catch (e) {
            setError(e instanceof Error ? e.message : 'Failed to load sessions');
        } finally {
            setLoading(false);
        }
    }, []);

    useEffect(() => {
        loadSessions();

        if (!wsService) return;

        const off = wsService.on('agent_sessions', (data) => {
            const event = data.event as string;

            switch (event) {
                case 'session_created':
                    setSessions(prev => {
                        // Avoid duplicate
                        if (prev.some(s => s.id === data.sessionId)) return prev;
                        return [...prev, {
                            id: data.sessionId as string,
                            state: (data.state as AgentSessionInfo['state']) || 'ACTIVE',
                            cascadeId: data.cascadeId as string | null,
                            cascadeIdShort: data.cascadeIdShort as string || '--------',
                            stepCount: (data.stepCount as number) || 0,
                            stepSoftLimit: (data.stepSoftLimit as number) || 500,
                            isBusy: !!data.isBusy,
                            workspace: (data.workspace as string) || '',
                            transport: (data.transport as string) || 'unknown',
                            lastActivity: (data.lastActivity as number) || Date.now(),
                        }];
                    });
                    break;

                case 'session_destroyed':
                    setSessions(prev => prev.filter(s => s.id !== data.sessionId));
                    break;

                case 'session_status_change':
                    setSessions(prev => prev.map(s => {
                        if (s.id !== data.sessionId) return s;
                        return {
                            ...s,
                            state: (data.state as AgentSessionInfo['state']) || s.state,
                            isBusy: data.isBusy != null ? !!data.isBusy : s.isBusy,
                            lastActivity: Date.now(),
                        };
                    }));
                    break;
            }
        });

        // Re-fetch on WS reconnect
        const offOpen = wsService.on('__ws_open', () => { loadSessions(); });

        return () => { off(); offOpen(); };
    }, [loadSessions]);

    const handleDestroy = useCallback(async (id: string) => {
        try {
            await destroyAgentSession(id);
            setSessions(prev => prev.filter(s => s.id !== id));
        } catch { /* WS event will handle removal */ }
    }, []);

    if (loading) {
        return (
            <div className="flex items-center justify-center h-full">
                <Loader2 className="h-5 w-5 animate-spin text-muted-foreground/30" />
            </div>
        );
    }

    if (error) {
        return (
            <div className="flex flex-col items-center justify-center h-full gap-3 px-6">
                <AlertCircle className="h-6 w-6 text-red-400/50" />
                <p className="text-xs text-red-400/70">{error}</p>
                <Button size="sm" variant="outline" onClick={loadSessions} className="text-[10px]">
                    <RefreshCw className="h-3 w-3 mr-1" /> Retry
                </Button>
            </div>
        );
    }

    if (sessions.length === 0) {
        return (
            <div className="flex flex-col items-center justify-center h-full gap-3 px-6">
                <div className="w-12 h-12 rounded-2xl bg-muted/10 flex items-center justify-center">
                    <Users className="h-6 w-6 text-muted-foreground/15" />
                </div>
                <div className="text-center space-y-1">
                    <p className="text-xs text-muted-foreground/50 font-medium">No active agent sessions</p>
                    <p className="text-[10px] text-muted-foreground/30">
                        Connect from the Chat tab or from an external agent via WebSocket/HTTP API
                    </p>
                </div>
            </div>
        );
    }

    return (
        <div className="p-3 space-y-2 overflow-y-auto h-full">
            {sessions.map(s => (
                <SessionCard key={s.id} session={s} onDestroy={handleDestroy} />
            ))}
        </div>
    );
}
```

- [ ] **Step 3: Commit**

```bash
git add frontend/components/agent-hub/sessions-panel.tsx
git commit -m "feat(frontend): add Agent Hub sessions panel with real-time WS updates"
```

---

## Task 7: Frontend — Chat Panel

**Files:**
- Create: `frontend/components/agent-hub/chat-panel.tsx`

Receives `useAgentWs` state via props (hook lives in parent `AgentHubView`). Renders workspace selector when disconnected, message list + input when connected.

- [ ] **Step 1: Create `frontend/components/agent-hub/chat-panel.tsx`**

```typescript
'use client';

import { useState, useRef, useEffect, memo } from 'react';
import { cn } from '@/lib/utils';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { ScrollArea } from '@/components/ui/scroll-area';
import {
    Wifi, WifiOff, Send, Check, X, RefreshCw, Unplug,
    Loader2, AlertCircle, MessageSquare,
} from 'lucide-react';
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import { SESSION_STATE_CONFIG } from '@/lib/agent-utils';
import type { UseAgentWsReturn } from '@/hooks/use-agent-ws';
import type { AgentMessage } from '@/lib/agent-api';

interface ChatPanelProps {
    agentWs: UseAgentWsReturn;
    workspaces: string[];
}

// ── Message bubble ──────────────────────────────────────────────────────

const MessageBubble = memo(function MessageBubble({ msg }: { msg: AgentMessage }) {
    if (msg.role === 'system') {
        return (
            <div className="flex justify-center py-1">
                <span className="text-[10px] text-muted-foreground/40 italic text-center max-w-[80%]">
                    {msg.content}
                </span>
            </div>
        );
    }

    const isUser = msg.role === 'user';

    return (
        <div className={cn('flex', isUser ? 'justify-end' : 'justify-start')}>
            <div className={cn(
                'max-w-[85%] rounded-lg px-3 py-2 text-xs',
                isUser
                    ? 'bg-primary/10 text-foreground/80'
                    : 'bg-muted/10 text-foreground/80'
            )}>
                {isUser ? (
                    <p className="whitespace-pre-wrap break-words">{msg.content}</p>
                ) : (
                    <div className="prose prose-invert prose-xs max-w-none [&>*:first-child]:mt-0 [&>*:last-child]:mb-0">
                        <ReactMarkdown remarkPlugins={[remarkGfm]}>
                            {msg.content}
                        </ReactMarkdown>
                    </div>
                )}
                {msg.stepCount != null && (
                    <div className="text-[9px] text-muted-foreground/30 mt-1">
                        Step {msg.stepIndex}/{msg.stepCount}
                    </div>
                )}
            </div>
        </div>
    );
});

// ── Main Panel ──────────────────────────────────────────────────────────

export function AgentChatPanel({ agentWs, workspaces }: ChatPanelProps) {
    const { state, sessionId, cascadeId, workspace, messages, error } = agentWs;
    const [selectedWorkspace, setSelectedWorkspace] = useState<string>('');
    const [inputText, setInputText] = useState('');
    const bottomRef = useRef<HTMLDivElement>(null);

    const isConnected = state === 'connected' || state === 'busy';
    const isBusy = state === 'busy';

    // Auto-scroll on new messages
    useEffect(() => {
        bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
    }, [messages.length]);

    const handleSend = () => {
        const text = inputText.trim();
        if (!text || isBusy) return;
        agentWs.send(text);
        setInputText('');
    };

    const handleKeyDown = (e: React.KeyboardEvent) => {
        if (e.key === 'Enter' && !e.shiftKey) {
            e.preventDefault();
            handleSend();
        }
    };

    // ── Not connected state ─────────────────────────────────────────────

    if (!isConnected && state !== 'connecting' && state !== 'reconnecting') {
        return (
            <div className="flex flex-col items-center justify-center h-full gap-4 px-6">
                <div className="w-12 h-12 rounded-2xl bg-muted/10 flex items-center justify-center">
                    <MessageSquare className="h-6 w-6 text-muted-foreground/15" />
                </div>

                <div className="text-center space-y-1">
                    <p className="text-xs text-muted-foreground/50 font-medium">Agent Chat</p>
                    <p className="text-[10px] text-muted-foreground/30">Select a workspace and connect to start</p>
                </div>

                {error && (
                    <div className="flex items-center gap-1.5 text-red-400/70">
                        <AlertCircle className="h-3 w-3" />
                        <span className="text-[10px]">{error}</span>
                    </div>
                )}

                <div className="w-full max-w-[200px] space-y-2">
                    <Select value={selectedWorkspace} onValueChange={setSelectedWorkspace}>
                        <SelectTrigger className="h-8 text-[11px]">
                            <SelectValue placeholder="Select workspace" />
                        </SelectTrigger>
                        <SelectContent>
                            {workspaces.map(ws => (
                                <SelectItem key={ws} value={ws} className="text-[11px]">{ws}</SelectItem>
                            ))}
                        </SelectContent>
                    </Select>

                    <Button
                        size="sm"
                        className="w-full h-8 text-[11px]"
                        disabled={!selectedWorkspace}
                        onClick={() => agentWs.connect(selectedWorkspace)}
                    >
                        <Wifi className="h-3 w-3 mr-1.5" />
                        Connect
                    </Button>
                </div>
            </div>
        );
    }

    // ── Connecting / Reconnecting state ─────────────────────────────────

    if (state === 'connecting' || state === 'reconnecting') {
        return (
            <div className="flex flex-col items-center justify-center h-full gap-3">
                <Loader2 className="h-5 w-5 animate-spin text-muted-foreground/30" />
                <p className="text-[10px] text-muted-foreground/40">
                    {state === 'connecting' ? 'Connecting…' : 'Reconnecting…'}
                </p>
            </div>
        );
    }

    // ── Connected state ─────────────────────────────────────────────────

    return (
        <div className="flex flex-col h-full">
            {/* Session info bar */}
            <div className="flex items-center justify-between px-3 py-1.5 border-b border-border/20 shrink-0">
                <div className="flex items-center gap-2 text-[10px] text-muted-foreground/60">
                    <span className={cn('w-1.5 h-1.5 rounded-full', isBusy ? 'bg-amber-400 animate-pulse' : 'bg-emerald-400')} />
                    <span className="font-mono">{sessionId?.substring(0, 8)}</span>
                    <span>•</span>
                    <span>{workspace}</span>
                    {cascadeId && (
                        <>
                            <span>•</span>
                            <span className="font-mono">#{cascadeId.substring(0, 8)}</span>
                        </>
                    )}
                </div>
                <span className={cn('text-[9px]', isBusy ? 'text-amber-400' : 'text-emerald-400')}>
                    {isBusy ? 'Processing…' : 'Ready'}
                </span>
            </div>

            {/* Messages */}
            <div className="flex-1 overflow-y-auto p-3 space-y-2">
                {messages.map(msg => (
                    <MessageBubble key={msg.id} msg={msg} />
                ))}
                {isBusy && messages[messages.length - 1]?.role === 'user' && (
                    <div className="flex justify-start">
                        <div className="bg-muted/10 rounded-lg px-3 py-2">
                            <Loader2 className="h-3 w-3 animate-spin text-muted-foreground/30" />
                        </div>
                    </div>
                )}
                <div ref={bottomRef} />
            </div>

            {/* Input + Controls */}
            <div className="border-t border-border/20 p-2 space-y-1.5 shrink-0">
                <div className="flex gap-1.5">
                    <Input
                        value={inputText}
                        onChange={e => setInputText(e.target.value)}
                        onKeyDown={handleKeyDown}
                        placeholder={isBusy ? 'Waiting for response…' : 'Send a message…'}
                        disabled={isBusy}
                        className="text-[11px] h-8"
                    />
                    <Button
                        size="icon"
                        className="h-8 w-8 shrink-0"
                        onClick={handleSend}
                        disabled={isBusy || !inputText.trim()}
                    >
                        <Send className="h-3 w-3" />
                    </Button>
                </div>

                {/* Session controls */}
                <div className="flex items-center gap-1">
                    <Button variant="ghost" size="sm" className="h-6 text-[9px] px-1.5"
                        onClick={agentWs.accept} title="Accept code changes">
                        <Check className="h-3 w-3 mr-0.5 text-emerald-400" /> Accept
                    </Button>
                    <Button variant="ghost" size="sm" className="h-6 text-[9px] px-1.5"
                        onClick={agentWs.reject} title="Reject code changes">
                        <X className="h-3 w-3 mr-0.5 text-red-400" /> Reject
                    </Button>
                    <Button variant="ghost" size="sm" className="h-6 text-[9px] px-1.5"
                        onClick={agentWs.newCascade} title="New cascade">
                        <RefreshCw className="h-3 w-3 mr-0.5" /> New Cascade
                    </Button>
                    <div className="flex-1" />
                    <Button variant="ghost" size="sm" className="h-6 text-[9px] px-1.5 text-red-400/60 hover:text-red-400"
                        onClick={agentWs.disconnect}>
                        <Unplug className="h-3 w-3 mr-0.5" /> Disconnect
                    </Button>
                </div>
            </div>
        </div>
    );
}
```

- [ ] **Step 2: Commit**

```bash
git add frontend/components/agent-hub/chat-panel.tsx
git commit -m "feat(frontend): add Agent Hub chat panel with message bubbles and controls"
```

---

## Task 8: Frontend — Config Panel

**Files:**
- Create: `frontend/components/agent-hub/config-panel.tsx`

Two collapsible sections: Agent API settings + Discord Bridge settings (ported from `agent-bridge-view.tsx`).

- [ ] **Step 1: Create `frontend/components/agent-hub/config-panel.tsx`**

```typescript
'use client';

import { useState, useEffect, useCallback } from 'react';
import { cn } from '@/lib/utils';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Switch } from '@/components/ui/switch';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import {
    Save, Check, ChevronDown, ChevronUp, Play, Square,
    Eye, EyeOff, Loader2, AlertCircle, Settings2, Bot,
} from 'lucide-react';
import { wsService } from '@/lib/ws-service';
import { SESSION_STATE_CONFIG } from '@/lib/agent-utils';
import {
    fetchAgentApiSettings, saveAgentApiSettings as saveApiSettings,
    fetchBridgeSettings, saveBridgeSettings as saveBridgeSettingsApi,
    fetchBridgeStatus, startBridge, stopBridge,
} from '@/lib/agent-api';
import type { AgentApiSettings, BridgeSettings, BridgeStatus } from '@/lib/agent-api';

// ── Defaults ────────────────────────────────────────────────────────────

const DEFAULT_API_SETTINGS: AgentApiSettings = {
    enabled: true, maxConcurrentSessions: 5,
    sessionTimeoutMs: 1800000, defaultStepSoftLimit: 500,
};

const DEFAULT_BRIDGE: BridgeSettings = {
    discordBotToken: '', discordChannelId: '', discordGuildId: '',
    stepSoftLimit: 500, allowedBotIds: [], autoStart: false,
};

export function AgentConfigPanel() {
    // ── Agent API Settings ──────────────────────────────────────────────
    const [api, setApi] = useState<AgentApiSettings>(DEFAULT_API_SETTINGS);
    const [apiOriginal, setApiOriginal] = useState<AgentApiSettings>(DEFAULT_API_SETTINGS);
    const [apiSaving, setApiSaving] = useState(false);
    const [apiMsg, setApiMsg] = useState('');
    const [apiOpen, setApiOpen] = useState(true);

    // ── Discord Bridge Settings ─────────────────────────────────────────
    const [bridge, setBridge] = useState<BridgeSettings>(DEFAULT_BRIDGE);
    const [bridgeOriginal, setBridgeOriginal] = useState<BridgeSettings>(DEFAULT_BRIDGE);
    const [bridgeStatus, setBridgeStatus] = useState<BridgeStatus | null>(null);
    const [bridgeSaving, setBridgeSaving] = useState(false);
    const [bridgeMsg, setBridgeMsg] = useState('');
    const [bridgeLoading, setBridgeLoading] = useState(false);
    const [bridgeOpen, setBridgeOpen] = useState(false);
    const [showToken, setShowToken] = useState(false);

    // ── Load on mount ───────────────────────────────────────────────────
    useEffect(() => {
        fetchAgentApiSettings()
            .then(d => { setApi(d); setApiOriginal(d); })
            .catch(() => {});

        fetchBridgeSettings()
            .then(d => { const s = { ...DEFAULT_BRIDGE, ...d }; setBridge(s); setBridgeOriginal(s); })
            .catch(() => {});

        fetchBridgeStatus()
            .then(setBridgeStatus)
            .catch(() => {});

        // WS updates for bridge status
        if (!wsService) return;
        const off = wsService.on('bridge_status', (data) => {
            setBridgeStatus(data as unknown as BridgeStatus);
        });
        return off;
    }, []);

    // ── API settings save ───────────────────────────────────────────────
    const hasApiChanges = JSON.stringify(api) !== JSON.stringify(apiOriginal);

    const handleSaveApi = async () => {
        setApiSaving(true);
        setApiMsg('');
        try {
            // Convert minutes → ms for backend
            const updated = await saveApiSettings(api);
            setApi(updated);
            setApiOriginal(updated);
            setApiMsg('saved');
            setTimeout(() => setApiMsg(''), 2000);
        } catch {
            setApiMsg('error');
        } finally {
            setApiSaving(false);
        }
    };

    // ── Bridge settings save ────────────────────────────────────────────
    const hasBridgeChanges = JSON.stringify(bridge) !== JSON.stringify(bridgeOriginal);

    const handleSaveBridge = async () => {
        setBridgeSaving(true);
        setBridgeMsg('');
        try {
            const updated = await saveBridgeSettingsApi(bridge);
            const s = { ...DEFAULT_BRIDGE, ...updated };
            setBridge(s);
            setBridgeOriginal(s);
            setBridgeMsg('saved');
            setTimeout(() => setBridgeMsg(''), 2000);
        } catch {
            setBridgeMsg('error');
        } finally {
            setBridgeSaving(false);
        }
    };

    const handleStartBridge = async () => {
        setBridgeLoading(true);
        try {
            const body: Record<string, unknown> = {};
            if (bridge.discordBotToken) body.discordBotToken = bridge.discordBotToken;
            if (bridge.discordChannelId) body.discordChannelId = bridge.discordChannelId;
            if (bridge.discordGuildId) body.discordGuildId = bridge.discordGuildId;
            if (bridge.stepSoftLimit) body.stepSoftLimit = bridge.stepSoftLimit;
            const data = await startBridge(body);
            setBridgeStatus(data);
        } catch { /* error handled by bridge_status WS */ }
        finally { setBridgeLoading(false); }
    };

    const handleStopBridge = async () => {
        setBridgeLoading(true);
        try {
            await stopBridge();
            const status = await fetchBridgeStatus();
            setBridgeStatus(status);
        } finally { setBridgeLoading(false); }
    };

    const bridgeState = bridgeStatus?.state || 'IDLE';
    const stateConf = SESSION_STATE_CONFIG[bridgeState as keyof typeof SESSION_STATE_CONFIG] || SESSION_STATE_CONFIG.IDLE;
    const canStartBridge = bridge.discordBotToken && bridge.discordChannelId;

    // Convert ms ↔ minutes for display
    const timeoutMinutes = Math.round(api.sessionTimeoutMs / 60000);

    return (
        <div className="p-3 space-y-3 overflow-y-auto h-full">
            {/* ── Section 1: Agent API Settings ── */}
            <Card className="bg-muted/5 border-border/20">
                <CardHeader className="p-3 pb-0 cursor-pointer" onClick={() => setApiOpen(!apiOpen)}>
                    <div className="flex items-center justify-between">
                        <CardTitle className="text-xs flex items-center gap-1.5">
                            <Settings2 className="h-3.5 w-3.5" /> Agent API
                        </CardTitle>
                        {apiOpen ? <ChevronUp className="h-3 w-3 text-muted-foreground/40" /> : <ChevronDown className="h-3 w-3 text-muted-foreground/40" />}
                    </div>
                </CardHeader>
                {apiOpen && (
                    <CardContent className="p-3 pt-2 space-y-3">
                        <div className="flex items-center justify-between">
                            <Label className="text-[10px] text-muted-foreground/70">Enable Agent API</Label>
                            <Switch checked={api.enabled} onCheckedChange={v => setApi(a => ({ ...a, enabled: v }))} className="scale-75" />
                        </div>

                        <div className="grid grid-cols-3 gap-2">
                            <div className="space-y-1">
                                <Label className="text-[10px] text-muted-foreground/70">Max Sessions</Label>
                                <Input type="number" value={api.maxConcurrentSessions} min={1} max={20}
                                    onChange={e => setApi(a => ({ ...a, maxConcurrentSessions: parseInt(e.target.value) || 1 }))}
                                    className="font-mono text-[11px] h-8" />
                            </div>
                            <div className="space-y-1">
                                <Label className="text-[10px] text-muted-foreground/70">Timeout (min)</Label>
                                <Input type="number" value={timeoutMinutes} min={1} max={1440}
                                    onChange={e => setApi(a => ({ ...a, sessionTimeoutMs: (parseInt(e.target.value) || 1) * 60000 }))}
                                    className="font-mono text-[11px] h-8" />
                            </div>
                            <div className="space-y-1">
                                <Label className="text-[10px] text-muted-foreground/70">Step Limit</Label>
                                <Input type="number" value={api.defaultStepSoftLimit} min={10} max={10000}
                                    onChange={e => setApi(a => ({ ...a, defaultStepSoftLimit: parseInt(e.target.value) || 10 }))}
                                    className="font-mono text-[11px] h-8" />
                            </div>
                        </div>

                        <div className="flex items-center justify-end gap-2">
                            {apiMsg && (
                                <span className={cn('text-[10px] font-medium', apiMsg === 'saved' ? 'text-emerald-400' : 'text-red-400')}>
                                    {apiMsg === 'saved' ? <><Check className="h-3 w-3 inline mr-0.5" />Saved</> : 'Error'}
                                </span>
                            )}
                            <Button size="sm" variant="outline" onClick={handleSaveApi}
                                disabled={apiSaving || !hasApiChanges} className="h-7 text-[10px] gap-1 px-2.5">
                                <Save className="w-3 h-3" /> {apiSaving ? 'Saving…' : 'Save'}
                            </Button>
                        </div>
                    </CardContent>
                )}
            </Card>

            {/* ── Section 2: Discord Bridge ── */}
            <Card className="bg-muted/5 border-border/20">
                <CardHeader className="p-3 pb-0 cursor-pointer" onClick={() => setBridgeOpen(!bridgeOpen)}>
                    <div className="flex items-center justify-between">
                        <CardTitle className="text-xs flex items-center gap-1.5">
                            <Bot className="h-3.5 w-3.5" /> Discord Bridge
                            <span className={cn('w-1.5 h-1.5 rounded-full ml-1', stateConf.dot)} />
                        </CardTitle>
                        <div className="flex items-center gap-1.5">
                            <span className={cn('text-[9px]', stateConf.color)}>{stateConf.label}</span>
                            {bridgeOpen ? <ChevronUp className="h-3 w-3 text-muted-foreground/40" /> : <ChevronDown className="h-3 w-3 text-muted-foreground/40" />}
                        </div>
                    </div>
                </CardHeader>
                {bridgeOpen && (
                    <CardContent className="p-3 pt-2 space-y-3">
                        {/* Bot Token */}
                        <div className="space-y-1">
                            <Label className="text-[10px] text-muted-foreground/70">Discord Bot Token</Label>
                            <div className="relative">
                                <Input type={showToken ? 'text' : 'password'} value={bridge.discordBotToken}
                                    onChange={e => setBridge(b => ({ ...b, discordBotToken: e.target.value }))}
                                    placeholder="MTQ3OTUw..." className="font-mono text-[11px] h-8 pr-8" />
                                <button type="button" onClick={() => setShowToken(!showToken)}
                                    className="absolute right-2 top-1/2 -translate-y-1/2 text-muted-foreground/40 hover:text-foreground/60 transition-colors">
                                    {showToken ? <EyeOff className="h-3 w-3" /> : <Eye className="h-3 w-3" />}
                                </button>
                            </div>
                        </div>

                        {/* Channel & Guild */}
                        <div className="grid grid-cols-2 gap-2">
                            <div className="space-y-1">
                                <Label className="text-[10px] text-muted-foreground/70">Channel ID</Label>
                                <Input value={bridge.discordChannelId}
                                    onChange={e => setBridge(b => ({ ...b, discordChannelId: e.target.value }))}
                                    placeholder="1479500166..." className="font-mono text-[11px] h-8" />
                            </div>
                            <div className="space-y-1">
                                <Label className="text-[10px] text-muted-foreground/70">Guild ID</Label>
                                <Input value={bridge.discordGuildId}
                                    onChange={e => setBridge(b => ({ ...b, discordGuildId: e.target.value }))}
                                    placeholder="1479500111..." className="font-mono text-[11px] h-8" />
                            </div>
                        </div>

                        {/* Step Limit & Allowed Bots */}
                        <div className="grid grid-cols-2 gap-2">
                            <div className="space-y-1">
                                <Label className="text-[10px] text-muted-foreground/70">Step Soft Limit</Label>
                                <Input type="number" value={bridge.stepSoftLimit}
                                    onChange={e => setBridge(b => ({ ...b, stepSoftLimit: parseInt(e.target.value) || 0 }))}
                                    min={0} max={10000} className="font-mono text-[11px] h-8" />
                            </div>
                            <div className="space-y-1">
                                <Label className="text-[10px] text-muted-foreground/70">Allowed Bot IDs</Label>
                                <Input value={bridge.allowedBotIds.join(', ')}
                                    onChange={e => setBridge(b => ({
                                        ...b, allowedBotIds: e.target.value.split(',').map(s => s.trim()).filter(Boolean),
                                    }))}
                                    placeholder="bot_id_1, bot_id_2" className="font-mono text-[11px] h-8" />
                            </div>
                        </div>

                        {/* Auto-start + Save + Start/Stop */}
                        <div className="flex items-center justify-between pt-1">
                            <div className="flex items-center gap-2">
                                <Switch checked={bridge.autoStart}
                                    onCheckedChange={v => setBridge(b => ({ ...b, autoStart: v }))}
                                    className="scale-75 origin-left" />
                                <span className="text-[10px] text-muted-foreground/60">Auto-start</span>
                            </div>
                            <div className="flex items-center gap-1.5">
                                {bridgeMsg && (
                                    <span className={cn('text-[10px] font-medium', bridgeMsg === 'saved' ? 'text-emerald-400' : 'text-red-400')}>
                                        {bridgeMsg === 'saved' ? 'Saved' : 'Error'}
                                    </span>
                                )}
                                <Button size="sm" variant="outline" onClick={handleSaveBridge}
                                    disabled={bridgeSaving || !hasBridgeChanges} className="h-7 text-[10px] gap-1 px-2">
                                    <Save className="w-3 h-3" /> Save
                                </Button>
                                {bridgeState === 'IDLE' ? (
                                    <Button size="sm" onClick={handleStartBridge}
                                        disabled={bridgeLoading || !canStartBridge} className="h-7 text-[10px] gap-1 px-2">
                                        <Play className="w-3 h-3" /> Start
                                    </Button>
                                ) : (
                                    <Button size="sm" variant="destructive" onClick={handleStopBridge}
                                        disabled={bridgeLoading} className="h-7 text-[10px] gap-1 px-2">
                                        <Square className="w-3 h-3" /> Stop
                                    </Button>
                                )}
                            </div>
                        </div>
                    </CardContent>
                )}
            </Card>
        </div>
    );
}
```

- [ ] **Step 2: Commit**

```bash
git add frontend/components/agent-hub/config-panel.tsx
git commit -m "feat(frontend): add Agent Hub config panel with API + Discord settings"
```

---

## Task 9: Frontend — Logs Panel

**Files:**
- Create: `frontend/components/agent-hub/logs-panel.tsx`

Streams log events via UI WS. Filters by transport and level. Auto-scrolls. Max 500 entries FIFO.

- [ ] **Step 1: Create `frontend/components/agent-hub/logs-panel.tsx`**

```typescript
'use client';

import { useState, useEffect, useRef, useCallback, memo } from 'react';
import { cn } from '@/lib/utils';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { wsService } from '@/lib/ws-service';
import { Trash2, ArrowDown } from 'lucide-react';
import { TRANSPORT_CONFIG, LOG_COLORS, LOG_ICONS, formatTimestamp } from '@/lib/agent-utils';
import { MessageSquare } from 'lucide-react';
import type { AgentLogEntry } from '@/lib/agent-api';

const MAX_LOG_ENTRIES = 500;

const TRANSPORT_FILTERS = ['All', 'Discord', 'WS', 'HTTP', 'UI'] as const;
const LEVEL_FILTERS = ['All', 'Info', 'Error'] as const;

function getTransportFilter(transport: string): string {
    if (transport === 'discord') return 'Discord';
    if (transport === 'websocket' || transport === 'websocket-ui') return transport === 'websocket-ui' ? 'UI' : 'WS';
    if (transport === 'http') return 'HTTP';
    return 'WS';
}

function getLevelFilter(logType: string): string {
    if (logType === 'error') return 'Error';
    return 'Info';
}

// ── Log Entry (memoized) ────────────────────────────────────────────────

const LogEntry = memo(function LogEntry({ entry }: { entry: AgentLogEntry }) {
    const transport = TRANSPORT_CONFIG[entry.transport] || TRANSPORT_CONFIG.unknown;
    const color = LOG_COLORS[entry.logType] || LOG_COLORS.system;
    const Icon = LOG_ICONS[entry.logType] || MessageSquare;
    const time = formatTimestamp(entry.timestamp);

    return (
        <div className="flex items-start gap-2 px-3 py-1.5 border-b border-border/5 hover:bg-muted/5 transition-colors">
            <span className="text-[9px] text-muted-foreground/25 font-mono shrink-0 mt-0.5 w-[52px]">{time}</span>
            <Badge variant="outline" className={cn('text-[8px] px-1 py-0 shrink-0', transport.color, transport.bg)}>
                {transport.label}
            </Badge>
            <span className="text-[9px] text-muted-foreground/30 font-mono shrink-0 w-[32px]">
                {entry.sessionId?.substring(0, 4) || '----'}
            </span>
            <Icon className={cn('w-3 h-3 mt-0.5 shrink-0', color)} />
            <p className={cn('text-[10px] break-words leading-relaxed flex-1 min-w-0', color)}>
                {entry.message}
            </p>
        </div>
    );
});

// ── Main Panel ──────────────────────────────────────────────────────────

export function AgentLogsPanel() {
    const [entries, setEntries] = useState<AgentLogEntry[]>([]);
    const [transportFilter, setTransportFilter] = useState<string>('All');
    const [levelFilter, setLevelFilter] = useState<string>('All');
    const [autoScroll, setAutoScroll] = useState(true);
    const bottomRef = useRef<HTMLDivElement>(null);
    const scrollContainerRef = useRef<HTMLDivElement>(null);

    // WS listener for log events
    useEffect(() => {
        if (!wsService) return;
        const off = wsService.on('agent_sessions', (data) => {
            if (data.event !== 'session_log') return;
            const entry: AgentLogEntry = {
                sessionId: data.sessionId as string,
                transport: data.transport as string,
                logType: data.logType as string,
                message: data.message as string,
                timestamp: (data.timestamp as number) || Date.now(),
            };
            setEntries(prev => {
                const next = [...prev, entry];
                return next.length > MAX_LOG_ENTRIES ? next.slice(-MAX_LOG_ENTRIES) : next;
            });
        });
        return off;
    }, []);

    // Auto-scroll
    useEffect(() => {
        if (autoScroll) {
            bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
        }
    }, [entries.length, autoScroll]);

    // Detect manual scroll up
    const handleScroll = useCallback(() => {
        const el = scrollContainerRef.current;
        if (!el) return;
        const atBottom = el.scrollHeight - el.scrollTop - el.clientHeight < 50;
        setAutoScroll(atBottom);
    }, []);

    const handleClear = useCallback(() => { setEntries([]); }, []);

    // Filter entries
    const filtered = entries.filter(e => {
        if (transportFilter !== 'All' && getTransportFilter(e.transport) !== transportFilter) return false;
        if (levelFilter !== 'All' && getLevelFilter(e.logType) !== levelFilter) return false;
        return true;
    });

    return (
        <div className="flex flex-col h-full">
            {/* Filters */}
            <div className="flex items-center gap-2 px-3 py-1.5 border-b border-border/20 shrink-0">
                <div className="flex items-center gap-0.5">
                    {TRANSPORT_FILTERS.map(f => (
                        <Button key={f} variant={transportFilter === f ? 'secondary' : 'ghost'}
                            size="sm" className="h-5 text-[9px] px-1.5"
                            onClick={() => setTransportFilter(f)}>
                            {f}
                        </Button>
                    ))}
                </div>
                <span className="text-muted-foreground/20">|</span>
                <div className="flex items-center gap-0.5">
                    {LEVEL_FILTERS.map(f => (
                        <Button key={f} variant={levelFilter === f ? 'secondary' : 'ghost'}
                            size="sm" className="h-5 text-[9px] px-1.5"
                            onClick={() => setLevelFilter(f)}>
                            {f}
                        </Button>
                    ))}
                </div>
                <div className="flex-1" />
                <span className="text-[9px] text-muted-foreground/30">{filtered.length}</span>
                <Button variant="ghost" size="icon" className="h-5 w-5" onClick={handleClear} title="Clear logs">
                    <Trash2 className="h-3 w-3 text-muted-foreground/30" />
                </Button>
            </div>

            {/* Log entries */}
            <div className="flex-1 overflow-y-auto" ref={scrollContainerRef} onScroll={handleScroll}>
                {filtered.length === 0 ? (
                    <div className="flex items-center justify-center h-full">
                        <p className="text-[10px] text-muted-foreground/30">No log entries</p>
                    </div>
                ) : (
                    filtered.map((e, i) => <LogEntry key={`${e.timestamp}-${i}`} entry={e} />)
                )}
                <div ref={bottomRef} />
            </div>

            {/* Scroll to bottom button */}
            {!autoScroll && (
                <div className="absolute bottom-12 right-4">
                    <Button size="icon" variant="secondary" className="h-7 w-7 rounded-full shadow-lg"
                        onClick={() => { setAutoScroll(true); bottomRef.current?.scrollIntoView({ behavior: 'smooth' }); }}>
                        <ArrowDown className="h-3 w-3" />
                    </Button>
                </div>
            )}
        </div>
    );
}
```

- [ ] **Step 2: Commit**

```bash
git add frontend/components/agent-hub/logs-panel.tsx
git commit -m "feat(frontend): add Agent Hub logs panel with transport/level filters"
```

---

## Task 10: Frontend — AgentHubView (main container)

**Files:**
- Create: `frontend/components/agent-hub-view.tsx`

Main container with shadcn Tabs. Owns `useAgentWs` hook (survives tab switches). Passes hook state to ChatPanel. Fetches workspace list for the workspace selector.

- [ ] **Step 1: Create `frontend/components/agent-hub-view.tsx`**

```typescript
'use client';

import { useState, useEffect } from 'react';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Users, MessageSquare, Settings2, ScrollText } from 'lucide-react';
import { useAgentWs } from '@/hooks/use-agent-ws';
import { AgentSessionsPanel } from '@/components/agent-hub/sessions-panel';
import { AgentChatPanel } from '@/components/agent-hub/chat-panel';
import { AgentConfigPanel } from '@/components/agent-hub/config-panel';
import { AgentLogsPanel } from '@/components/agent-hub/logs-panel';
import { API_BASE } from '@/lib/config';
import { authHeaders } from '@/lib/auth';

export function AgentHubView() {
    const agentWs = useAgentWs();
    const [workspaces, setWorkspaces] = useState<string[]>([]);

    // Fetch workspace list for the Chat panel's workspace selector
    useEffect(() => {
        (async () => {
            try {
                const res = await fetch(`${API_BASE}/api/workspaces`, { headers: authHeaders() });
                if (res.ok) {
                    const data = await res.json();
                    // data is an array of workspace objects with .name
                    const names = Array.isArray(data)
                        ? data.map((w: { name?: string; workspaceName?: string }) => w.name || w.workspaceName || '').filter(Boolean)
                        : [];
                    setWorkspaces(names);
                }
            } catch { /* silent */ }
        })();
    }, []);

    // Chat tab badge: show dot when connected
    const chatConnected = agentWs.state === 'connected' || agentWs.state === 'busy';

    return (
        <div className="flex flex-col h-full bg-background">
            {/* Header */}
            <div className="flex items-center px-4 py-2 border-b border-border/30 shrink-0">
                <span className="text-xs font-semibold text-foreground/80">Agent Hub</span>
            </div>

            {/* Tabs */}
            <Tabs defaultValue="sessions" className="flex flex-col flex-1 min-h-0">
                <TabsList className="w-full justify-start rounded-none border-b border-border/20 bg-transparent h-8 px-2">
                    <TabsTrigger value="sessions" className="text-[10px] h-6 gap-1 px-2 data-[state=active]:bg-muted/10">
                        <Users className="h-3 w-3" /> Sessions
                    </TabsTrigger>
                    <TabsTrigger value="chat" className="text-[10px] h-6 gap-1 px-2 data-[state=active]:bg-muted/10 relative">
                        <MessageSquare className="h-3 w-3" /> Chat
                        {chatConnected && (
                            <span className="absolute top-1 right-1 w-1.5 h-1.5 rounded-full bg-emerald-400" />
                        )}
                    </TabsTrigger>
                    <TabsTrigger value="config" className="text-[10px] h-6 gap-1 px-2 data-[state=active]:bg-muted/10">
                        <Settings2 className="h-3 w-3" /> Config
                    </TabsTrigger>
                    <TabsTrigger value="logs" className="text-[10px] h-6 gap-1 px-2 data-[state=active]:bg-muted/10">
                        <ScrollText className="h-3 w-3" /> Logs
                    </TabsTrigger>
                </TabsList>

                <TabsContent value="sessions" className="flex-1 min-h-0 m-0">
                    <AgentSessionsPanel />
                </TabsContent>

                <TabsContent value="chat" className="flex-1 min-h-0 m-0">
                    <AgentChatPanel agentWs={agentWs} workspaces={workspaces} />
                </TabsContent>

                <TabsContent value="config" className="flex-1 min-h-0 m-0">
                    <AgentConfigPanel />
                </TabsContent>

                <TabsContent value="logs" className="flex-1 min-h-0 m-0 relative">
                    <AgentLogsPanel />
                </TabsContent>
            </Tabs>
        </div>
    );
}
```

- [ ] **Step 2: Commit**

```bash
git add frontend/components/agent-hub-view.tsx
git commit -m "feat(frontend): add AgentHubView main container with tabs"
```

---

## Task 11: Frontend — Wire into page.tsx and app-sidebar.tsx

**Files:**
- Modify: `frontend/app/page.tsx`
- Modify: `frontend/components/app-sidebar.tsx`

Replace `showBridge`/`AgentBridgeView` with `showAgentHub`/`AgentHubView`. Rename sidebar button.

- [ ] **Step 1: Update `frontend/app/page.tsx`**

Make these changes in order:

**1a.** Change the import (line 28):
```typescript
// FROM:
import { AgentBridgeView } from '@/components/agent-bridge-view';
// TO:
import { AgentHubView } from '@/components/agent-hub-view';
```

**1b.** Rename state variable (line 154):
```typescript
// FROM:
const [showBridge, setShowBridge] = useState(() => getStoredValue('antigravity-show-bridge', false));
// TO:
const [showAgentHub, setShowAgentHub] = useState(() => getStoredValue('antigravity-show-agent-hub', false));
```

**1c.** Update localStorage persistence (line 175):
```typescript
// FROM:
useEffect(() => { localStorage.setItem('antigravity-show-bridge', JSON.stringify(showBridge)); }, [showBridge]);
// TO:
useEffect(() => { localStorage.setItem('antigravity-show-agent-hub', JSON.stringify(showAgentHub)); }, [showAgentHub]);
```

**1d.** Update resetPanels (line 213):
```typescript
// FROM:
setShowBridge(false);
// TO:
setShowAgentHub(false);
```

**1e.** Update handleShowBridge → handleShowAgentHub (lines 280-286):
```typescript
// FROM:
const handleShowBridge = useCallback(() => {
    selectConversation(null);
    resetPanels();
    setActiveWorkspace(null);
    setShowBridge(true);
}, [selectConversation, resetPanels]);
// TO:
const handleShowAgentHub = useCallback(() => {
    selectConversation(null);
    resetPanels();
    setActiveWorkspace(null);
    setShowAgentHub(true);
}, [selectConversation, resetPanels]);
```

**1f.** Update view logic (lines 421-422) — replace all `showBridge` with `showAgentHub`:
```typescript
const showConversationList = detected && !showChat && !showAccountInfo && !showSettings && !showLogs && !showAgentHub && !showSourceControl && !showResources && activeWorkspace !== null;
const showWelcome = !detected || (!showChat && !showConversationList && !showAccountInfo && !showSettings && !showLogs && !showAgentHub && !showSourceControl && !showResources);
```

**1g.** Update sidebar callback (line 439):
```typescript
// FROM:
onShowBridge={handleShowBridge}
// TO:
onShowAgentHub={handleShowAgentHub}
```

**1h.** Update rendering (lines 621-623):
```typescript
// FROM:
<div className={detected && showBridge ? 'flex flex-col flex-1 min-h-0 overflow-hidden' : 'hidden'}>
    <AgentBridgeView />
</div>
// TO:
<div className={detected && showAgentHub ? 'flex flex-col flex-1 min-h-0 overflow-hidden' : 'hidden'}>
    <AgentHubView />
</div>
```

- [ ] **Step 2: Update `frontend/components/app-sidebar.tsx`**

**2a.** Rename the callback prop (line 62):
```typescript
// FROM:
onShowBridge: () => void
// TO:
onShowAgentHub: () => void
```

**2b.** Update the menu item (lines 504-507):
```typescript
// FROM:
<DropdownMenuItem onClick={onShowBridge}>
    <Bot className="mr-2 h-4 w-4" />
    <span>Agent Bridge</span>
</DropdownMenuItem>
// TO:
<DropdownMenuItem onClick={onShowAgentHub}>
    <Bot className="mr-2 h-4 w-4" />
    <span>Agent Hub</span>
</DropdownMenuItem>
```

- [ ] **Step 3: Verify build**

Run: `cd frontend && npx next build`
Expected: Build succeeds with no errors. (There may be warnings — that's OK.)

- [ ] **Step 4: Commit**

```bash
git add frontend/app/page.tsx frontend/components/app-sidebar.tsx
git commit -m "feat(frontend): wire AgentHubView into page and sidebar, replacing AgentBridgeView"
```

---

## Task 12: Smoke test — manual verification

This is a manual verification task. Start both servers and verify the Agent Hub renders and functions.

- [ ] **Step 1: Start backend**

```bash
cd src && node server.js
```
Expected: Backend starts on port 3500.

- [ ] **Step 2: Start frontend**

```bash
cd frontend && npm run dev
```
Expected: Frontend starts on port 3000.

- [ ] **Step 3: Open Agent Hub**

Navigate to `http://localhost:3000`. Click the sidebar menu → "Agent Hub".

Verify:
- Tabs render: Sessions, Chat, Config, Logs
- Sessions tab shows empty state (or existing sessions if any are running)
- Config tab shows Agent API settings and Discord Bridge settings (collapsible)
- Logs tab shows "No log entries"

- [ ] **Step 4: Test Chat connection (if backend has LS instance available)**

Switch to Chat tab → select a workspace → click Connect.

Verify:
- State changes to connecting → connected
- Session info bar shows session ID and workspace
- Send a test message → input disables → response appears

- [ ] **Step 5: Test Config save**

In Config tab, change Max Sessions to 3 → click Save.

Verify:
- "Saved" message appears
- Curl `http://localhost:3500/api/agent-api/settings` returns `maxConcurrentSessions: 3`

- [ ] **Step 6: Commit final cleanup (if any fixes needed)**

```bash
git add -A
git commit -m "fix: Agent Hub smoke test fixes"
```

---

## Summary

| Task | Files | Description |
|------|-------|-------------|
| 1 | `agent-session-manager.js` | Backend: wire session events → broadcast |
| 2 | `ws-agent.js` | Backend: accept transport field |
| 3 | `agent-api.js` | Backend: settings endpoints |
| 4 | `agent-utils.ts`, `agent-api.ts`, `config.ts` | Shared constants, types, helpers |
| 5 | `use-agent-ws.ts` | Agent WS state machine hook |
| 6 | `sessions-panel.tsx` | Sessions panel |
| 7 | `chat-panel.tsx` | Chat panel |
| 8 | `config-panel.tsx` | Config panel (API + Discord) |
| 9 | `logs-panel.tsx` | Logs panel |
| 10 | `agent-hub-view.tsx` | Main container with tabs |
| 11 | `page.tsx`, `app-sidebar.tsx` | Wire into app |
| 12 | — | Manual smoke test |

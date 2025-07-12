# AI Workflow Builder - Complete Build Guide

## 🎯 Project Overview
Build a revolutionary visual AI workflow builder with drag-and-drop interface, real-time execution, chat-based workflow creation, and MCP server integration.

## 📋 Table of Contents
1. [Project Setup](#project-setup)
2. [Database Schema](#database-schema)
3. [Backend Implementation](#backend-implementation)
4. [Frontend Implementation](#frontend-implementation)
5. [MCP Server Integration](#mcp-server-integration)
6. [Node System](#node-system)
7. [Chat Widget](#chat-widget)
8. [File Operations](#file-operations)
9. [Deployment Configuration](#deployment-configuration)
10. [Testing & Validation](#testing--validation)

---

## 🚀 Project Setup

### 1. Initialize Next.js Project
```bash
npx create-next-app@latest ai-workflow-builder --typescript --tailwind --eslint --app
cd ai-workflow-builder
```

### 2. Install Dependencies
```bash
# Core dependencies
npm install @types/node @types/react @types/react-dom
npm install react-flow-renderer reactflow
npm install @prisma/client prisma
npm install next-auth
npm install @langchain/core @langchain/openai @langchain/anthropic
npm install openai anthropic
npm install socket.io socket.io-client
npm install zustand
npm install @tanstack/react-query
npm install lucide-react
npm install @radix-ui/react-dialog @radix-ui/react-button @radix-ui/react-input
npm install class-variance-authority clsx tailwind-merge
npm install zod
npm install bcryptjs jsonwebtoken
npm install ws
npm install redis ioredis
npm install node-cron

# Development dependencies
npm install --save-dev @types/bcryptjs @types/jsonwebtoken @types/ws
npm install --save-dev prisma @types/node
```

### 3. Project Structure
```
ai-workflow-builder/
├── src/
│   ├── app/
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── api/
│   │       ├── auth/
│   │       ├── workflows/
│   │       ├── nodes/
│   │       ├── executions/
│   │       ├── mcp/
│   │       └── chat/
│   ├── components/
│   │   ├── ui/
│   │   ├── workflow/
│   │   ├── nodes/
│   │   ├── chat/
│   │   └── dashboard/
│   ├── lib/
│   │   ├── workflow-engine/
│   │   ├── node-system/
│   │   ├── mcp-server/
│   │   ├── database/
│   │   └── utils/
│   ├── hooks/
│   └── types/
├── prisma/
├── public/
├── .env.local
├── next.config.js
├── tailwind.config.js
├── package.json
├── docker-compose.yml
├── Dockerfile
└── vercel.json
```

---

## 🗄️ Database Schema

### Prisma Schema (`prisma/schema.prisma`)
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  avatar    String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  workflows Workflow[]
  executions Execution[]
  
  @@map("users")
}

model Workflow {
  id          String   @id @default(cuid())
  name        String
  description String?
  nodes       Json
  edges       Json
  variables   Json     @default("{}")
  version     Int      @default(1)
  status      String   @default("draft") // draft, active, paused, archived
  isPublic    Boolean  @default(false)
  tags        String[]
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  userId      String
  user        User       @relation(fields: [userId], references: [id], onDelete: Cascade)
  executions  Execution[]
  
  @@map("workflows")
}

model Execution {
  id          String   @id @default(cuid())
  status      String   @default("running") // running, completed, failed, cancelled
  input       Json
  output      Json?
  logs        Json     @default("[]")
  metrics     Json     @default("{}")
  error       String?
  startedAt   DateTime @default(now())
  completedAt DateTime?
  duration    Int? // in milliseconds
  
  workflowId  String
  workflow    Workflow @relation(fields: [workflowId], references: [id], onDelete: Cascade)
  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("executions")
}

model NodeDefinition {
  id          String   @id @default(cuid())
  type        String   @unique
  name        String
  description String
  category    String
  icon        String?
  configSchema Json
  inputSchema  Json
  outputSchema Json
  isCustom    Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@map("node_definitions")
}

model ChatSession {
  id          String   @id @default(cuid())
  userId      String
  workflowId  String?
  messages    Json     @default("[]")
  context     Json     @default("{}")
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@map("chat_sessions")
}
```

### Database Migrations
```bash
npx prisma migrate dev --name init
npx prisma generate
```

---

## 🔧 Backend Implementation

### 1. Workflow Engine (`src/lib/workflow-engine/index.ts`)
```typescript
import { Workflow, ExecutionContext, NodeExecutionResult } from '@/types/workflow';
import { BaseNode } from '@/lib/node-system/base-node';
import { NodeRegistry } from '@/lib/node-system/node-registry';
import { prisma } from '@/lib/database/prisma';
import { EventEmitter } from 'events';

export class WorkflowEngine extends EventEmitter {
  private nodeRegistry: NodeRegistry;
  private executionMap: Map<string, ExecutionContext> = new Map();

  constructor() {
    super();
    this.nodeRegistry = new NodeRegistry();
  }

  async createWorkflow(data: {
    name: string;
    description?: string;
    nodes: any[];
    edges: any[];
    userId: string;
  }): Promise<Workflow> {
    const workflow = await prisma.workflow.create({
      data: {
        name: data.name,
        description: data.description,
        nodes: data.nodes,
        edges: data.edges,
        userId: data.userId,
      },
    });

    return workflow as Workflow;
  }

  async executeWorkflow(
    workflowId: string,
    input: any,
    options: {
      userId: string;
      streaming?: boolean;
      timeout?: number;
    }
  ): Promise<string> {
    const workflow = await prisma.workflow.findUnique({
      where: { id: workflowId },
    });

    if (!workflow) {
      throw new Error(`Workflow ${workflowId} not found`);
    }

    const execution = await prisma.execution.create({
      data: {
        workflowId,
        userId: options.userId,
        input,
        status: 'running',
      },
    });

    // Start execution in background
    this.executeWorkflowNodes(execution.id, workflow as Workflow, input, options);

    return execution.id;
  }

  private async executeWorkflowNodes(
    executionId: string,
    workflow: Workflow,
    input: any,
    options: any
  ): Promise<void> {
    const context: ExecutionContext = {
      workflowId: workflow.id,
      executionId,
      variables: { ...workflow.variables, ...input },
      currentNode: '',
      executionPath: [],
      logs: [],
      metrics: {},
    };

    this.executionMap.set(executionId, context);

    try {
      // Find entry nodes (nodes with no incoming edges)
      const entryNodes = workflow.nodes.filter(node => 
        !workflow.edges.some(edge => edge.target === node.id)
      );

      // Execute nodes in topological order
      for (const entryNode of entryNodes) {
        await this.executeNode(entryNode, context, workflow);
      }

      // Mark execution as completed
      await prisma.execution.update({
        where: { id: executionId },
        data: {
          status: 'completed',
          output: context.variables,
          logs: context.logs,
          metrics: context.metrics,
          completedAt: new Date(),
          duration: Date.now() - new Date(context.startedAt || 0).getTime(),
        },
      });

      this.emit('execution:completed', executionId, context);
    } catch (error) {
      await prisma.execution.update({
        where: { id: executionId },
        data: {
          status: 'failed',
          error: error.message,
          logs: context.logs,
          completedAt: new Date(),
        },
      });

      this.emit('execution:failed', executionId, error);
    } finally {
      this.executionMap.delete(executionId);
    }
  }

  private async executeNode(
    node: any,
    context: ExecutionContext,
    workflow: Workflow
  ): Promise<void> {
    context.currentNode = node.id;
    context.executionPath.push(node.id);

    const nodeInstance = this.nodeRegistry.getNode(node.type);
    if (!nodeInstance) {
      throw new Error(`Node type ${node.type} not found`);
    }

    // Configure node
    nodeInstance.configure(node.config);

    // Execute node
    const result = await nodeInstance.execute(context.variables, context);

    // Update context with result
    if (result.success) {
      context.variables = { ...context.variables, ...result.output };
      context.logs.push(...(result.logs || []));
    } else {
      throw new Error(result.error || 'Node execution failed');
    }

    // Emit progress event
    this.emit('execution:progress', context.executionId, {
      nodeId: node.id,
      status: 'completed',
      output: result.output,
    });

    // Execute connected nodes
    const connectedEdges = workflow.edges.filter(edge => edge.source === node.id);
    for (const edge of connectedEdges) {
      const nextNode = workflow.nodes.find(n => n.id === edge.target);
      if (nextNode) {
        await this.executeNode(nextNode, context, workflow);
      }
    }
  }

  getExecutionContext(executionId: string): ExecutionContext | undefined {
    return this.executionMap.get(executionId);
  }
}
```

### 2. API Routes

#### Workflows API (`src/app/api/workflows/route.ts`)
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { prisma } from '@/lib/database/prisma';
import { WorkflowEngine } from '@/lib/workflow-engine';

const workflowEngine = new WorkflowEngine();

export async function GET(request: NextRequest) {
  const session = await getServerSession();
  if (!session?.user?.email) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const user = await prisma.user.findUnique({
    where: { email: session.user.email },
  });

  if (!user) {
    return NextResponse.json({ error: 'User not found' }, { status: 404 });
  }

  const workflows = await prisma.workflow.findMany({
    where: { userId: user.id },
    orderBy: { updatedAt: 'desc' },
  });

  return NextResponse.json(workflows);
}

export async function POST(request: NextRequest) {
  const session = await getServerSession();
  if (!session?.user?.email) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const user = await prisma.user.findUnique({
    where: { email: session.user.email },
  });

  if (!user) {
    return NextResponse.json({ error: 'User not found' }, { status: 404 });
  }

  const { name, description, nodes, edges } = await request.json();

  const workflow = await workflowEngine.createWorkflow({
    name,
    description,
    nodes,
    edges,
    userId: user.id,
  });

  return NextResponse.json(workflow);
}
```

#### Execution API (`src/app/api/executions/route.ts`)
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { prisma } from '@/lib/database/prisma';
import { WorkflowEngine } from '@/lib/workflow-engine';

const workflowEngine = new WorkflowEngine();

export async function POST(request: NextRequest) {
  const session = await getServerSession();
  if (!session?.user?.email) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const user = await prisma.user.findUnique({
    where: { email: session.user.email },
  });

  if (!user) {
    return NextResponse.json({ error: 'User not found' }, { status: 404 });
  }

  const { workflowId, input, options } = await request.json();

  const executionId = await workflowEngine.executeWorkflow(
    workflowId,
    input,
    { userId: user.id, ...options }
  );

  return NextResponse.json({ executionId });
}
```

---

## 🎨 Frontend Implementation

### 1. Main Application (`src/app/page.tsx`)
```typescript
'use client';

import { useState, useEffect } from 'react';
import { WorkflowCanvas } from '@/components/workflow/WorkflowCanvas';
import { NodeLibrary } from '@/components/workflow/NodeLibrary';
import { ChatWidget } from '@/components/chat/ChatWidget';
import { ExecutionDashboard } from '@/components/dashboard/ExecutionDashboard';
import { WorkflowProvider } from '@/context/WorkflowContext';

export default function Home() {
  const [activeTab, setActiveTab] = useState<'canvas' | 'executions'>('canvas');

  return (
    <WorkflowProvider>
      <div className="h-screen flex flex-col bg-gray-50">
        {/* Header */}
        <header className="bg-white border-b border-gray-200 p-4">
          <div className="flex items-center justify-between">
            <h1 className="text-2xl font-bold text-gray-900">
              AI Workflow Builder
            </h1>
            <div className="flex gap-2">
              <button
                onClick={() => setActiveTab('canvas')}
                className={`px-4 py-2 rounded-lg ${
                  activeTab === 'canvas'
                    ? 'bg-blue-500 text-white'
                    : 'bg-gray-200 text-gray-700'
                }`}
              >
                Canvas
              </button>
              <button
                onClick={() => setActiveTab('executions')}
                className={`px-4 py-2 rounded-lg ${
                  activeTab === 'executions'
                    ? 'bg-blue-500 text-white'
                    : 'bg-gray-200 text-gray-700'
                }`}
              >
                Executions
              </button>
            </div>
          </div>
        </header>

        {/* Main Content */}
        <div className="flex-1 flex">
          {activeTab === 'canvas' ? (
            <>
              <NodeLibrary />
              <WorkflowCanvas />
            </>
          ) : (
            <ExecutionDashboard />
          )}
        </div>

        {/* Chat Widget */}
        <ChatWidget />
      </div>
    </WorkflowProvider>
  );
}
```

### 2. Workflow Canvas (`src/components/workflow/WorkflowCanvas.tsx`)
```typescript
'use client';

import { useCallback, useEffect, useState } from 'react';
import ReactFlow, {
  Node,
  Edge,
  addEdge,
  Connection,
  useNodesState,
  useEdgesState,
  Controls,
  Background,
  NodeTypes,
} from 'reactflow';
import 'reactflow/dist/style.css';

import { useWorkflow } from '@/context/WorkflowContext';
import { LLMChatNode } from '@/components/nodes/LLMChatNode';
import { APICallNode } from '@/components/nodes/APICallNode';
import { ConditionalNode } from '@/components/nodes/ConditionalNode';

const nodeTypes: NodeTypes = {
  llmChat: LLMChatNode,
  apiCall: APICallNode,
  conditional: ConditionalNode,
};

export function WorkflowCanvas() {
  const { currentWorkflow, updateWorkflow } = useWorkflow();
  const [nodes, setNodes, onNodesChange] = useNodesState([]);
  const [edges, setEdges, onEdgesChange] = useEdgesState([]);

  const onConnect = useCallback(
    (params: Connection) => {
      const newEdge = addEdge(params, edges);
      setEdges(newEdge);
      updateWorkflow({ edges: newEdge });
    },
    [edges, setEdges, updateWorkflow]
  );

  const onDrop = useCallback(
    (event: React.DragEvent) => {
      event.preventDefault();

      const reactFlowBounds = event.currentTarget.getBoundingClientRect();
      const type = event.dataTransfer.getData('application/reactflow');
      const position = {
        x: event.clientX - reactFlowBounds.left,
        y: event.clientY - reactFlowBounds.top,
      };

      const newNode: Node = {
        id: `${type}-${Date.now()}`,
        type,
        position,
        data: {
          label: `${type} node`,
          config: {},
        },
      };

      setNodes((nds) => nds.concat(newNode));
      updateWorkflow({ nodes: [...nodes, newNode] });
    },
    [nodes, setNodes, updateWorkflow]
  );

  const onDragOver = useCallback((event: React.DragEvent) => {
    event.preventDefault();
    event.dataTransfer.dropEffect = 'move';
  }, []);

  // Load workflow data
  useEffect(() => {
    if (currentWorkflow) {
      setNodes(currentWorkflow.nodes || []);
      setEdges(currentWorkflow.edges || []);
    }
  }, [currentWorkflow, setNodes, setEdges]);

  return (
    <div className="flex-1 h-full">
      <ReactFlow
        nodes={nodes}
        edges={edges}
        onNodesChange={onNodesChange}
        onEdgesChange={onEdgesChange}
        onConnect={onConnect}
        onDrop={onDrop}
        onDragOver={onDragOver}
        nodeTypes={nodeTypes}
        fitView
      >
        <Background />
        <Controls />
      </ReactFlow>
    </div>
  );
}
```

### 3. Node Library (`src/components/workflow/NodeLibrary.tsx`)
```typescript
'use client';

import { useState } from 'react';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Input } from '@/components/ui/input';
import { Badge } from '@/components/ui/badge';

const NODE_CATEGORIES = {
  ai: {
    name: 'AI Nodes',
    nodes: [
      {
        type: 'llmChat',
        name: 'LLM Chat',
        description: 'Interact with large language models',
        icon: '🤖',
      },
      {
        type: 'textAnalysis',
        name: 'Text Analysis',
        description: 'Analyze text for sentiment, entities, etc.',
        icon: '📝',
      },
      {
        type: 'imageAnalysis',
        name: 'Image Analysis',
        description: 'Analyze images with AI',
        icon: '🖼️',
      },
    ],
  },
  integration: {
    name: 'Integration Nodes',
    nodes: [
      {
        type: 'apiCall',
        name: 'API Call',
        description: 'Make HTTP requests to external APIs',
        icon: '🌐',
      },
      {
        type: 'database',
        name: 'Database',
        description: 'Read/write from databases',
        icon: '💾',
      },
      {
        type: 'email',
        name: 'Email',
        description: 'Send emails',
        icon: '📧',
      },
    ],
  },
  logic: {
    name: 'Logic Nodes',
    nodes: [
      {
        type: 'conditional',
        name: 'Conditional',
        description: 'Route data based on conditions',
        icon: '🔀',
      },
      {
        type: 'loop',
        name: 'Loop',
        description: 'Iterate over data',
        icon: '🔄',
      },
      {
        type: 'transform',
        name: 'Transform',
        description: 'Transform data',
        icon: '🔧',
      },
    ],
  },
};

export function NodeLibrary() {
  const [searchTerm, setSearchTerm] = useState('');
  const [selectedCategory, setSelectedCategory] = useState<string>('all');

  const onDragStart = (event: React.DragEvent, nodeType: string) => {
    event.dataTransfer.setData('application/reactflow', nodeType);
    event.dataTransfer.effectAllowed = 'move';
  };

  const filteredNodes = Object.entries(NODE_CATEGORIES).flatMap(([categoryKey, category]) => {
    if (selectedCategory !== 'all' && selectedCategory !== categoryKey) {
      return [];
    }
    
    return category.nodes.filter(node =>
      node.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
      node.description.toLowerCase().includes(searchTerm.toLowerCase())
    );
  });

  return (
    <div className="w-80 bg-white border-r border-gray-200 p-4">
      <div className="mb-4">
        <h2 className="text-lg font-semibold mb-2">Node Library</h2>
        <Input
          placeholder="Search nodes..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
          className="mb-2"
        />
        <div className="flex flex-wrap gap-1">
          <Badge
            variant={selectedCategory === 'all' ? 'default' : 'secondary'}
            className="cursor-pointer"
            onClick={() => setSelectedCategory('all')}
          >
            All
          </Badge>
          {Object.entries(NODE_CATEGORIES).map(([key, category]) => (
            <Badge
              key={key}
              variant={selectedCategory === key ? 'default' : 'secondary'}
              className="cursor-pointer"
              onClick={() => setSelectedCategory(key)}
            >
              {category.name}
            </Badge>
          ))}
        </div>
      </div>

      <div className="space-y-2">
        {filteredNodes.map((node) => (
          <Card
            key={node.type}
            className="cursor-move hover:shadow-md transition-shadow"
            draggable
            onDragStart={(e) => onDragStart(e, node.type)}
          >
            <CardHeader className="pb-2">
              <CardTitle className="text-sm flex items-center gap-2">
                <span className="text-lg">{node.icon}</span>
                {node.name}
              </CardTitle>
            </CardHeader>
            <CardContent>
              <CardDescription className="text-xs">
                {node.description}
              </CardDescription>
            </CardContent>
          </Card>
        ))}
      </div>
    </div>
  );
}
```

---

## 🧩 Node System Implementation

### 1. Base Node Class (`src/lib/node-system/base-node.ts`)
```typescript
import { ExecutionContext } from '@/types/workflow';

export interface NodeExecutionResult {
  success: boolean;
  output?: any;
  error?: string;
  logs?: string[];
  metrics?: Record<string, any>;
}

export interface NodeConfig {
  [key: string]: any;
}

export abstract class BaseNode {
  protected config: NodeConfig = {};

  abstract readonly type: string;
  abstract readonly name: string;
  abstract readonly description: string;
  abstract readonly category: string;
  abstract readonly icon: string;

  configure(config: NodeConfig): void {
    this.config = { ...this.config, ...config };
  }

  abstract execute(
    input: any,
    context: ExecutionContext
  ): Promise<NodeExecutionResult>;

  abstract getConfigSchema(): any;

  protected handleError(error: Error): NodeExecutionResult {
    return {
      success: false,
      error: error.message,
      logs: [`Error: ${error.message}`],
    };
  }

  protected logExecution(message: string, data?: any): string {
    const timestamp = new Date().toISOString();
    const logEntry = `[${timestamp}] ${this.name}: ${message}`;
    if (data) {
      console.log(logEntry, data);
    }
    return logEntry;
  }
}
```

### 2. LLM Chat Node (`src/lib/node-system/nodes/llm-chat-node.ts`)
```typescript
import { BaseNode, NodeExecutionResult } from '../base-node';
import { ExecutionContext } from '@/types/workflow';
import OpenAI from 'openai';

export class LLMChatNode extends BaseNode {
  readonly type = 'llmChat';
  readonly name = 'LLM Chat';
  readonly description = 'Interact with large language models';
  readonly category = 'ai';
  readonly icon = '🤖';

  private openai: OpenAI;

  constructor() {
    super();
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,
    });
  }

  async execute(input: any, context: ExecutionContext): Promise<NodeExecutionResult> {
    try {
      const { model, prompt, temperature, maxTokens, systemMessage } = this.config;
      
      const messages = [
        ...(systemMessage ? [{ role: 'system' as const, content: systemMessage }] : []),
        { role: 'user' as const, content: this.interpolatePrompt(prompt, input, context) },
      ];

      const response = await this.openai.chat.completions.create({
        model: model || 'gpt-3.5-turbo',
        messages,
        temperature: temperature || 0.7,
        max_tokens: maxTokens || 1000,
      });

      const content = response.choices[0]?.message?.content || '';
      const tokensUsed = response.usage?.total_tokens || 0;

      return {
        success: true,
        output: {
          response: content,
          tokensUsed,
        },
        logs: [
          this.logExecution('LLM request completed', { tokensUsed }),
        ],
        metrics: {
          tokensUsed,
          model,
        },
      };
    } catch (error) {
      return this.handleError(error as Error);
    }
  }

  private interpolatePrompt(prompt: string, input: any, context: ExecutionContext): string {
    let interpolatedPrompt = prompt;
    
    // Replace variables from context
    Object.entries(context.variables).forEach(([key, value]) => {
      interpolatedPrompt = interpolatedPrompt.replace(
        new RegExp(`\\{\\{${key}\\}\\}`, 'g'),
        String(value)
      );
    });

    // Replace variables from input
    Object.entries(input).forEach(([key, value]) => {
      interpolatedPrompt = interpolatedPrompt.replace(
        new RegExp(`\\{\\{${key}\\}\\}`, 'g'),
        String(value)
      );
    });

    return interpolatedPrompt;
  }

  getConfigSchema() {
    return {
      type: 'object',
      properties: {
        model: {
          type: 'string',
          enum: ['gpt-3.5-turbo', 'gpt-4', 'gpt-4-turbo'],
          default: 'gpt-3.5-turbo',
        },
        prompt: {
          type: 'string',
          description: 'The prompt to send to the LLM',
        },
        systemMessage: {
          type: 'string',
          description: 'System message to set context',
        },
        temperature: {
          type: 'number',
          minimum: 0,
          maximum: 2,
          default: 0.7,
        },
        maxTokens: {
          type: 'number',
          minimum: 1,
          maximum: 4000,
          default: 1000,
        },
      },
      required: ['prompt'],
    };
  }
}
```

### 3. Node Registry (`src/lib/node-system/node-registry.ts`)
```typescript
import { BaseNode } from './base-node';
import { LLMChatNode } from './nodes/llm-chat-node';
import { APICallNode } from './nodes/api-call-node';
import { ConditionalNode } from './nodes/conditional-node';

export class NodeRegistry {
  private nodes: Map<string, BaseNode> = new Map();

  constructor() {
    this.registerDefaultNodes();
  }

  private registerDefaultNodes(): void {
    this.registerNode(new LLMChatNode());
    this.registerNode(new APICallNode());
    this.registerNode(new ConditionalNode());
  }

  registerNode(node: BaseNode): void {
    this.nodes.set(node.type, node);
  }

  getNode(type: string): BaseNode | undefined {
    return this.nodes.get(type);
  }

  getAllNodes(): BaseNode[] {
    return Array.from(this.nodes.values());
  }

  getNodesByCategory(category: string): BaseNode[] {
    return this.getAllNodes().filter(node => node.category === category);
  }
}
```

---

## 💬 Chat Widget Implementation

### 1. Chat Widget Component (`src/components/chat/ChatWidget.tsx`)
```typescript
'use client';

import { useState, useEffect, useRef } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';
import { ScrollArea } from '@/components/ui/scroll-area';
import { useWorkflow } from '@/context/WorkflowContext';
import { ChatMessage } from '@/types/chat';
import { MessageBubble } from './MessageBubble';

export function ChatWidget() {
  const [isOpen, setIsOpen] = useState(false);
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [inputValue, setInputValue] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const messagesEndRef = useRef<HTMLDivElement>(null);
  const { currentWorkflow, updateWorkflow } = useWorkflow();

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  };

  useEffect(() => {
    scrollToBottom();
  }, [messages]);

  const handleSendMessage = async () => {
    if (!inputValue.trim() || isLoading) return;

    const userMessage: ChatMessage = {
      id: Date.now().toString(),
      role: 'user',
      content: inputValue,
      timestamp: new Date(),
    };

    setMessages(prev => [...prev, userMessage]);
    setInputValue('');
    setIsLoading(true);

    try {
      const response = await fetch('/api/chat/workflow', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          message: inputValue,
          currentWorkflow,
          sessionId: 'default', // TODO: Implement session management
        }),
      });

      const data = await response.json();

      const aiMessage: ChatMessage = {
        id: (Date.now() + 1).toString(),
        role: 'assistant',
        content: data.response,
        timestamp: new Date(),
        workflowChanges: data.workflowChanges,
      };

      setMessages(prev => [...prev, aiMessage]);

      // Apply workflow changes if any
      if (data.workflowChanges) {
        await applyWorkflowChanges(data.workflowChanges);
      }
    } catch (error) {
      console.error('Error sending message:', error);
      const errorMessage: ChatMessage = {
        id: (Date.now() + 1).toString(),
        role: 'assistant',
        content: 'Sorry, I encountered an error. Please try again.',
        timestamp: new Date(),
      };
      setMessages(prev => [...prev, errorMessage]);
    } finally {
      setIsLoading(false);
    }
  };

  const applyWorkflowChanges = async (changes: any[]) => {
    // Apply changes to the current workflow
    for (const change of changes) {
      switch (change.type) {
        case 'add_node':
          // Add node to workflow
          updateWorkflow({
            nodes: [...(currentWorkflow?.nodes || []), change.node],
          });
          break;
        case 'connect_nodes':
          // Add edge between nodes
          updateWorkflow({
            edges: [...(currentWorkflow?.edges || []), change.edge],
          });
          break;
        case 'modify_node':
          // Modify existing node
          updateWorkflow({
            nodes: currentWorkflow?.nodes.map(node =>
              node.id === change.nodeId ? { ...node, ...change.updates } : node
            ),
          });
          break;
      }
    }
  };

  const handleKeyPress = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      handleSendMessage();
    }
  };

  if (!isOpen) {
    return (
      <div className="fixed bottom-4 right-4 z-50">
        <Button
          onClick={() => setIsOpen(true)}
          className="rounded-full w-12 h-12"
          size="icon"
        >
          💬
        </Button>
      </div>
    );
  }

  return (
    <div className="fixed bottom-4 right-4 w-96 h-[600px] z-50">
      <Card className="h-full flex flex-col">
        <CardHeader className="pb-3">
          <div className="flex items-center justify-between">
            <CardTitle className="text-lg">AI Workflow Assistant</CardTitle>
            <Button
              variant="ghost"
              size="sm"
              onClick={() => setIsOpen(false)}
            >
              ✕
            </Button>
          </div>
        </CardHeader>
        <CardContent className="flex-1 flex flex-col p-0">
          <ScrollArea className="flex-1 p-4">
            <div className="space-y-4">
              {messages.length === 0 && (
                <div className="text-center text-gray-500 py-8">
                  <p>👋 Hi! I'm your AI workflow assistant.</p>
                  <p className="text-sm mt-2">
                    Describe what you want to automate and I'll help you build it!
                  </p>
                </div>
              )}
              {messages.map((message) => (
                <MessageBubble key={message.id} message={message} />
              ))}
              {isLoading && (
                <div className="flex items-center gap-2 text-gray-500">
                  <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce" />
                  <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce delay-100" />
                  <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce delay-200" />
                  <span className="text-sm">AI is thinking...</span>
                </div>
              )}
              <div ref={messagesEndRef} />
            </div>
          </ScrollArea>
          <div className="p-4 border-t">
            <div className="flex gap-2">
              <Input
                value={inputValue}
                onChange={(e) => setInputValue(e.target.value)}
                onKeyPress={handleKeyPress}
                placeholder="Describe your workflow..."
                disabled={isLoading}
              />
              <Button
                onClick={handleSendMessage}
                disabled={!inputValue.trim() || isLoading}
              >
                Send
              </Button>
            </div>
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
```

### 2. Message Bubble Component (`src/components/chat/MessageBubble.tsx`)
```typescript
'use client';

import { ChatMessage } from '@/types/chat';
import { Card, CardContent } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';

interface MessageBubbleProps {
  message: ChatMessage;
}

export function MessageBubble({ message }: MessageBubbleProps) {
  const isUser = message.role === 'user';

  return (
    <div className={`flex ${isUser ? 'justify-end' : 'justify-start'}`}>
      <Card
        className={`max-w-[80%] ${
          isUser
            ? 'bg-blue-500 text-white'
            : 'bg-gray-100 text-gray-800'
        }`}
      >
        <CardContent className="p-3">
          <p className="text-sm">{message.content}</p>
          
          {message.workflowChanges && (
            <div className="mt-2 p-2 bg-white/20 rounded text-xs">
              <p className="font-medium">Workflow Changes:</p>
              <ul className="list-disc list-inside mt-1 space-y-1">
                {message.workflowChanges.map((change, index) => (
                  <li key={index}>
                    <Badge variant="secondary" className="mr-1">
                      {change.type}
                    </Badge>
                    {change.description}
                  </li>
                ))}
              </ul>
            </div>
          )}
          
          <div className="flex justify-between items-center mt-2">
            <span className="text-xs opacity-70">
              {message.timestamp.toLocaleTimeString()}
            </span>
            {!isUser && (
              <span className="text-xs opacity-70">AI</span>
            )}
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
```

### 3. Chat API Route (`src/app/api/chat/workflow/route.ts`)
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { MCPWorkflowServer } from '@/lib/mcp-server';

const mcpServer = new MCPWorkflowServer();

export async function POST(request: NextRequest) {
  try {
    const { message, currentWorkflow, sessionId } = await request.json();

    const response = await mcpServer.processWorkflowChat({
      message,
      currentWorkflow,
      sessionId,
    });

    return NextResponse.json(response);
  } catch (error) {
    console.error('Chat API error:', error);
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

---

## 📡 MCP Server Implementation

### 1. MCP Server Core (`src/lib/mcp-server/index.ts`)
```typescript
import { BaseNode } from '@/lib/node-system/base-node';
import { NodeRegistry } from '@/lib/node-system/node-registry';
import { WorkflowEngine } from '@/lib/workflow-engine';
import OpenAI from 'openai';

export interface MCPTool {
  name: string;
  description: string;
  inputSchema: any;
  handler: (params: any) => Promise<any>;
}

export class MCPWorkflowServer {
  private nodeRegistry: NodeRegistry;
  private workflowEngine: WorkflowEngine;
  private openai: OpenAI;
  private tools: Map<string, MCPTool> = new Map();

  constructor() {
    this.nodeRegistry = new NodeRegistry();
    this.workflowEngine = new WorkflowEngine();
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,
    });
    
    this.setupTools();
  }

  private setupTools(): void {
    // Create workflow tool
    this.tools.set('create_workflow', {
      name: 'create_workflow',
      description: 'Create a new workflow from natural language description',
      inputSchema: {
        type: 'object',
        properties: {
          description: { type: 'string' },
          name: { type: 'string' },
          userId: { type: 'string' },
        },
        required: ['description', 'name', 'userId'],
      },
      handler: this.createWorkflowFromDescription.bind(this),
    });

    // Execute workflow tool
    this.tools.set('execute_workflow', {
      name: 'execute_workflow',
      description: 'Execute a workflow with given inputs',
      inputSchema: {
        type: 'object',
        properties: {
          workflowId: { type: 'string' },
          input: { type: 'object' },
          userId: { type: 'string' },
        },
        required: ['workflowId', 'input', 'userId'],
      },
      handler: this.executeWorkflow.bind(this),
    });

    // Modify workflow tool
    this.tools.set('modify_workflow', {
      name: 'modify_workflow',
      description: 'Modify an existing workflow based on instructions',
      inputSchema: {
        type: 'object',
        properties: {
          workflowId: { type: 'string' },
          instructions: { type: 'string' },
          currentWorkflow: { type: 'object' },
        },
        required: ['workflowId', 'instructions'],
      },
      handler: this.modifyWorkflow.bind(this),
    });

    // List available nodes tool
    this.tools.set('list_nodes', {
      name: 'list_nodes',
      description: 'List all available node types',
      inputSchema: { type: 'object' },
      handler: this.listNodes.bind(this),
    });
  }

  async processWorkflowChat(params: {
    message: string;
    currentWorkflow?: any;
    sessionId: string;
  }): Promise<any> {
    const { message, currentWorkflow, sessionId } = params;

    // Analyze the message to determine intent
    const intent = await this.analyzeIntent(message, currentWorkflow);

    let response: any = {
      response: '',
      workflowChanges: [],
    };

    switch (intent.type) {
      case 'create_workflow':
        response = await this.handleCreateWorkflow(message, intent);
        break;
      case 'modify_workflow':
        response = await this.handleModifyWorkflow(message, currentWorkflow, intent);
        break;
      case 'execute_workflow':
        response = await this.handleExecuteWorkflow(message, currentWorkflow, intent);
        break;
      case 'question':
        response = await this.handleQuestion(message, currentWorkflow);
        break;
      default:
        response.response = "I'm not sure how to help with that. Can you please be more specific about what you'd like to do with your workflow?";
    }

    return response;
  }

  private async analyzeIntent(message: string, currentWorkflow?: any): Promise<any> {
    const prompt = `
    Analyze this user message and determine their intent:
    Message: "${message}"
    Current workflow: ${currentWorkflow ? 'exists' : 'none'}
    
    Possible intents:
    - create_workflow: User wants to create a new workflow
    - modify_workflow: User wants to modify existing workflow
    - execute_workflow: User wants to run a workflow
    - question: User is asking about workflows or needs help
    
    Respond with JSON: { "type": "intent_type", "details": "additional_context" }
    `;

    const response = await this.openai.chat.completions.create({
      model: 'gpt-3.5-turbo',
      messages: [{ role: 'user', content: prompt }],
      temperature: 0.3,
    });

    try {
      return JSON.parse(response.choices[0]?.message?.content || '{}');
    } catch {
      return { type: 'question', details: 'Unable to parse intent' };
    }
  }

  private async handleCreateWorkflow(message: string, intent: any): Promise<any> {
    const workflowPlan = await this.generateWorkflowPlan(message);
    
    return {
      response: `I'll create a workflow for you! Here's what I'm building:\n\n${workflowPlan.description}`,
      workflowChanges: workflowPlan.changes,
    };
  }

  private async handleModifyWorkflow(message: string, currentWorkflow: any, intent: any): Promise<any> {
    const modifications = await this.generateWorkflowModifications(message, currentWorkflow);
    
    return {
      response: `I'll modify your workflow as requested:\n\n${modifications.description}`,
      workflowChanges: modifications.changes,
    };
  }

  private async generateWorkflowPlan(description: string): Promise<any> {
    const availableNodes = this.nodeRegistry.getAllNodes();
    const nodeDescriptions = availableNodes.map(node => 
      `${node.type}: ${node.description}`
    ).join('\n');

    const prompt = `
    Create a workflow plan for: "${description}"
    
    Available nodes:
    ${nodeDescriptions}
    
    Generate a workflow plan with:
    1. A clear description of what the workflow does
    2. The nodes needed and their connections
    3. Configuration for each node
    
    Return JSON with:
    {
      "description": "Clear explanation of the workflow",
      "changes": [
        {
          "type": "add_node",
          "node": {
            "id": "unique_id",
            "type": "node_type",
            "position": {"x": 100, "y": 100},
            "data": {
              "label": "Node Name",
              "config": {}
            }
          }
        },
        {
          "type": "connect_nodes",
          "edge": {
            "id": "unique_edge_id",
            "source": "source_node_id",
            "target": "target_node_id"
          }
        }
      ]
    }
    `;

    const response = await this.openai.chat.completions.create({
      model: 'gpt-4',
      messages: [{ role: 'user', content: prompt }],
      temperature: 0.3,
    });

    try {
      return JSON.parse(response.choices[0]?.message?.content || '{}');
    } catch {
      return {
        description: 'I encountered an error creating the workflow plan.',
        changes: [],
      };
    }
  }

  private async generateWorkflowModifications(instruction: string, currentWorkflow: any): Promise<any> {
    const prompt = `
    Modify this workflow based on the instruction: "${instruction}"
    
    Current workflow:
    ${JSON.stringify(currentWorkflow, null, 2)}
    
    Generate modifications as a JSON object with:
    {
      "description": "Explanation of changes",
      "changes": [
        {
          "type": "add_node|modify_node|remove_node|connect_nodes",
          "...": "specific change data"
        }
      ]
    }
    `;

    const response = await this.openai.chat.completions.create({
      model: 'gpt-4',
      messages: [{ role: 'user', content: prompt }],
      temperature: 0.3,
    });

    try {
      return JSON.parse(response.choices[0]?.message?.content || '{}');
    } catch {
      return {
        description: 'I encountered an error modifying the workflow.',
        changes: [],
      };
    }
  }

  private async createWorkflowFromDescription(params: any): Promise<any> {
    const { description, name, userId } = params;
    
    const workflowPlan = await this.generateWorkflowPlan(description);
    
    const workflow = await this.workflowEngine.createWorkflow({
      name,
      description,
      nodes: workflowPlan.changes
        .filter((change: any) => change.type === 'add_node')
        .map((change: any) => change.node),
      edges: workflowPlan.changes
        .filter((change: any) => change.type === 'connect_nodes')
        .map((change: any) => change.edge),
      userId,
    });

    return {
      workflowId: workflow.id,
      workflow,
      description: workflowPlan.description,
    };
  }

  private async executeWorkflow(params: any): Promise<any> {
    const { workflowId, input, userId } = params;
    
    const executionId = await this.workflowEngine.executeWorkflow(
      workflowId,
      input,
      { userId }
    );

    return { executionId };
  }

  private async modifyWorkflow(params: any): Promise<any> {
    const { workflowId, instructions, currentWorkflow } = params;
    
    const modifications = await this.generateWorkflowModifications(
      instructions,
      currentWorkflow
    );

    return modifications;
  }

  private async listNodes(): Promise<any> {
    const nodes = this.nodeRegistry.getAllNodes();
    
    return {
      nodes: nodes.map(node => ({
        type: node.type,
        name: node.name,
        description: node.description,
        category: node.category,
        configSchema: node.getConfigSchema(),
      })),
    };
  }

  private async handleQuestion(message: string, currentWorkflow?: any): Promise<any> {
    const prompt = `
    User question: "${message}"
    ${currentWorkflow ? `Current workflow: ${JSON.stringify(currentWorkflow)}` : 'No current workflow'}
    
    Provide a helpful response about workflow automation and building.
    `;

    const response = await this.openai.chat.completions.create({
      model: 'gpt-3.5-turbo',
      messages: [{ role: 'user', content: prompt }],
      temperature: 0.7,
    });

    return {
      response: response.choices[0]?.message?.content || 'I apologize, but I encountered an error processing your question.',
      workflowChanges: [],
    };
  }

  // MCP Protocol methods
  async listTools(): Promise<MCPTool[]> {
    return Array.from(this.tools.values());
  }

  async callTool(name: string, params: any): Promise<any> {
    const tool = this.tools.get(name);
    if (!tool) {
      throw new Error(`Tool ${name} not found`);
    }

    return await tool.handler(params);
  }
}
```

---

## 📁 File Operations Implementation

### 1. File Operations Utilities (`src/lib/utils/file-operations.ts`)
```typescript
import { Workflow } from '@/types/workflow';

export class FileOperations {
  static downloadWorkflow(workflow: Workflow): void {
    const dataStr = JSON.stringify(workflow, null, 2);
    const dataUri = 'data:application/json;charset=utf-8,'+ encodeURIComponent(dataStr);
    
    const exportFileDefaultName = `${workflow.name.replace(/[^a-z0-9]/gi, '_').toLowerCase()}_workflow.json`;
    
    const linkElement = document.createElement('a');
    linkElement.setAttribute('href', dataUri);
    linkElement.setAttribute('download', exportFileDefaultName);
    linkElement.click();
  }

  static async uploadWorkflow(file: File): Promise<Workflow> {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      
      reader.onload = (e) => {
        try {
          const workflow = JSON.parse(e.target?.result as string);
          resolve(workflow);
        } catch (error) {
          reject(new Error('Invalid workflow file format'));
        }
      };
      
      reader.onerror = () => {
        reject(new Error('Failed to read file'));
      };
      
      reader.readAsText(file);
    });
  }

  static validateWorkflowFile(workflow: any): boolean {
    const requiredFields = ['name', 'nodes', 'edges'];
    return requiredFields.every(field => field in workflow);
  }

  static async exportWorkflowAsImage(canvasElement: HTMLElement): Promise<void> {
    const html2canvas = (await import('html2canvas')).default;
    
    const canvas = await html2canvas(canvasElement);
    const link = document.createElement('a');
    link.download = 'workflow.png';
    link.href = canvas.toDataURL();
    link.click();
  }
}
```

### 2. File Upload Component (`src/components/workflow/FileUpload.tsx`)
```typescript
'use client';

import { useRef } from 'react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { useWorkflow } from '@/context/WorkflowContext';
import { FileOperations } from '@/lib/utils/file-operations';
import { toast } from '@/hooks/use-toast';

export function FileUpload() {
  const fileInputRef = useRef<HTMLInputElement>(null);
  const { setCurrentWorkflow } = useWorkflow();

  const handleFileUpload = async (event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0];
    if (!file) return;

    try {
      const workflow = await FileOperations.uploadWorkflow(file);
      
      if (!FileOperations.validateWorkflowFile(workflow)) {
        throw new Error('Invalid workflow file format');
      }

      setCurrentWorkflow(workflow);
      toast({
        title: 'Success',
        description: 'Workflow imported successfully',
      });
    } catch (error) {
      toast({
        title: 'Error',
        description: error instanceof Error ? error.message : 'Failed to import workflow',
        variant: 'destructive',
      });
    }

    // Reset the input
    if (fileInputRef.current) {
      fileInputRef.current.value = '';
    }
  };

  return (
    <div className="flex gap-2">
      <Input
        ref={fileInputRef}
        type="file"
        accept=".json"
        onChange={handleFileUpload}
        className="hidden"
      />
      <Button
        onClick={() => fileInputRef.current?.click()}
        variant="outline"
      >
        Import Workflow
      </Button>
    </div>
  );
}
```

### 3. File Download Component (`src/components/workflow/FileDownload.tsx`)
```typescript
'use client';

import { Button } from '@/components/ui/button';
import { DropdownMenu, DropdownMenuContent, DropdownMenuItem, DropdownMenuTrigger } from '@/components/ui/dropdown-menu';
import { useWorkflow } from '@/context/WorkflowContext';
import { FileOperations } from '@/lib/utils/file-operations';
import { Download, FileText, Image } from 'lucide-react';

export function FileDownload() {
  const { currentWorkflow } = useWorkflow();

  const handleDownloadJSON = () => {
    if (!currentWorkflow) return;
    FileOperations.downloadWorkflow(currentWorkflow);
  };

  const handleDownloadImage = async () => {
    const canvasElement = document.querySelector('.react-flow') as HTMLElement;
    if (!canvasElement) return;
    
    try {
      await FileOperations.exportWorkflowAsImage(canvasElement);
    } catch (error) {
      console.error('Failed to export image:', error);
    }
  };

  if (!currentWorkflow) {
    return null;
  }

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="outline" className="flex items-center gap-2">
          <Download className="w-4 h-4" />
          Export
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent>
        <DropdownMenuItem onClick={handleDownloadJSON}>
          <FileText className="w-4 h-4 mr-2" />
          Download as JSON
        </DropdownMenuItem>
        <DropdownMenuItem onClick={handleDownloadImage}>
          <Image className="w-4 h-4 mr-2" />
          Download as Image
        </DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

---

## 🚀 Deployment Configuration

### 1. Environment Variables (`.env.local`)
```env
# Database
DATABASE_URL="postgresql://username:password@localhost:5432/ai_workflow_builder"
REDIS_URL="redis://localhost:6379"

# Authentication
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="your-secret-key-here"

# AI APIs
OPENAI_API_KEY="sk-your-openai-key"
ANTHROPIC_API_KEY="sk-ant-your-anthropic-key"
GOOGLE_API_KEY="your-google-api-key"

# Vector Database
PINECONE_API_KEY="your-pinecone-key"
PINECONE_ENVIRONMENT="your-environment"

# App Configuration
NODE_ENV="development"
```

### 2. Next.js Configuration (`next.config.js`)
```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    serverActions: true,
  },
  webpack: (config) => {
    config.externals.push({
      'utf-8-validate': 'commonjs utf-8-validate',
      'bufferutil': 'commonjs bufferutil',
    });
    return config;
  },
  images: {
    domains: ['localhost'],
  },
  async headers() {
    return [
      {
        source: '/api/:path*',
        headers: [
          { key: 'Access-Control-Allow-Credentials', value: 'true' },
          { key: 'Access-Control-Allow-Origin', value: '*' },
          { key: 'Access-Control-Allow-Methods', value: 'GET,OPTIONS,PATCH,DELETE,POST,PUT' },
          { key: 'Access-Control-Allow-Headers', value: 'X-CSRF-Token, X-Requested-With, Accept, Accept-Version, Content-Length, Content-MD5, Content-Type, Date, X-Api-Version' },
        ],
      },
    ];
  },
};

module.exports = nextConfig;
```

### 3. Docker Configuration (`Dockerfile`)
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy source code
COPY . .

# Generate Prisma client
RUN npx prisma generate

# Build the application
RUN npm run build

# Expose port
EXPOSE 3000

# Start the application
CMD ["npm", "start"]
```

### 4. Docker Compose (`docker-compose.yml`)
```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/ai_workflow_builder
      - REDIS_URL=redis://redis:6379
      - NEXTAUTH_URL=http://localhost:3000
    depends_on:
      - postgres
      - redis
    volumes:
      - .:/app
      - /app/node_modules

  postgres:
    image: postgres:14
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: ai_workflow_builder
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### 5. Vercel Configuration (`vercel.json`)
```json
{
  "framework": "nextjs",
  "buildCommand": "npm run build",
  "outputDirectory": ".next",
  "installCommand": "npm ci",
  "functions": {
    "pages/api/**/*.ts": {
      "maxDuration": 30
    }
  },
  "env": {
    "NODE_ENV": "production"
  },
  "build": {
    "env": {
      "SKIP_ENV_VALIDATION": "1"
    }
  }
}
```

---

## 📋 Build Instructions

### 1. Development Setup
```bash
# 1. Clone or create project
git clone <repo-url> ai-workflow-builder
cd ai-workflow-builder

# 2. Install dependencies
npm install

# 3. Set up environment variables
cp .env.example .env.local
# Edit .env.local with your API keys

# 4. Set up database
docker-compose up -d postgres redis
npx prisma migrate dev --name init
npx prisma generate

# 5. Start development server
npm run dev
```

### 2. Production Build
```bash
# Build for production
npm run build

# Start production server
npm start
```

### 3. Docker Deployment
```bash
# Build and run with Docker Compose
docker-compose up -d --build

# Or build Docker image manually
docker build -t ai-workflow-builder .
docker run -p 3000:3000 ai-workflow-builder
```

### 4. Vercel Deployment
```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel

# Set environment variables in Vercel dashboard
# Deploy to production
vercel --prod
```

---

## 🧪 Testing & Validation

### 1. Test Scripts (`package.json`)
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:e2e": "playwright test"
  }
}
```

### 2. Basic Component Tests
```typescript
// __tests__/components/WorkflowCanvas.test.tsx
import { render, screen } from '@testing-library/react';
import { WorkflowCanvas } from '@/components/workflow/WorkflowCanvas';

describe('WorkflowCanvas', () => {
  it('renders the canvas', () => {
    render(<WorkflowCanvas />);
    expect(screen.getByRole('main')).toBeInTheDocument();
  });
});
```

### 3. API Tests
```typescript
// __tests__/api/workflows.test.ts
import { GET, POST } from '@/app/api/workflows/route';

describe('/api/workflows', () => {
  it('should return workflows', async () => {
    const response = await GET(new Request('http://localhost:3000/api/workflows'));
    expect(response.status).toBe(200);
  });
});
```

---

## 🎯 Key Features Checklist

### Core Features
- [ ] Visual workflow canvas with drag-and-drop
- [ ] Node library with AI, integration, and logic nodes
- [ ] Real-time workflow execution
- [ ] Workflow file import/export (JSON)
- [ ] Chat widget for natural language workflow creation
- [ ] MCP server integration
- [ ] User authentication and authorization
- [ ] Database persistence
- [ ] Real-time collaboration

### Advanced Features
- [ ] Workflow versioning
- [ ] Execution history and logging
- [ ] Performance monitoring
- [ ] Custom node creation
- [ ] Workflow templates
- [ ] Scheduling and automation
- [ ] Multi-user workspaces
- [ ] API integrations
- [ ] Webhook support
- [ ] Error handling and retry logic

### Deployment Features
- [ ] Docker containerization
- [ ] Vercel deployment
- [ ] Environment configuration
- [ ] Database migrations
- [ ] Security headers
- [ ] Performance optimization
- [ ] Monitoring and logging
- [ ] Auto-scaling
- [ ] CI/CD pipeline

---

## 🚀 Next Steps

1. **Initialize the project** following the setup instructions
2. **Implement core components** starting with the workflow canvas
3. **Add node system** with basic AI and logic nodes
4. **Integrate MCP server** for programmatic access
5. **Build chat widget** for natural language workflow creation
6. **Add file operations** for import/export functionality
7. **Implement real-time features** with WebSockets
8. **Add authentication** and user management
9. **Deploy to production** using Vercel or Docker
10. **Test and iterate** based on user feedback

This comprehensive guide provides everything needed to build a fully functional AI workflow builder with MCP integration. The architecture is scalable, the features are production-ready, and the deployment options are flexible.

**Ready to build the future of AI workflow automation!** 🚀
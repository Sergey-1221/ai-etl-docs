# 🎨 Frontend Development Guide

## Overview

The AI ETL Assistant frontend is built with Next.js 14, React 18, and TypeScript, providing a modern, responsive interface for pipeline management.

## Technology Stack

- **Framework**: Next.js 14 (App Router)
- **UI Library**: React 18
- **Language**: TypeScript 5
- **Styling**: Tailwind CSS + shadcn/ui
- **State Management**: Zustand
- **Data Fetching**: TanStack Query
- **Forms**: React Hook Form + Zod
- **Charts**: Recharts
- **DAG Editor**: React Flow
- **Animations**: Framer Motion
- **Icons**: Lucide React

## Project Structure

```
frontend/
├── app/                     # Next.js App Router
│   ├── (auth)/             # Auth pages (login, register)
│   ├── (dashboard)/        # Main app pages
│   │   ├── dashboard/      # Dashboard
│   │   ├── pipelines/      # Pipeline management
│   │   ├── studio/         # Pipeline editor
│   │   └── settings/       # Settings
│   ├── api/                # API routes
│   ├── layout.tsx          # Root layout
│   └── page.tsx            # Landing page
├── components/             # React components
│   ├── ui/                # shadcn/ui components
│   ├── layout/            # Layout components
│   ├── pipelines/         # Pipeline components
│   ├── dashboard/         # Dashboard widgets
│   └── shared/            # Shared components
├── hooks/                  # Custom React hooks
├── lib/                    # Utility functions
├── services/              # API service layer
├── store/                 # Zustand stores
├── styles/                # Global styles
└── types/                 # TypeScript types
```

## Component Development

### Creating Components

#### Basic Component

```typescript
// components/pipelines/PipelineCard.tsx
import { FC } from 'react';
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Pipeline } from '@/types/pipeline';

interface PipelineCardProps {
  pipeline: Pipeline;
  onSelect?: (pipeline: Pipeline) => void;
}

export const PipelineCard: FC<PipelineCardProps> = ({
  pipeline,
  onSelect
}) => {
  return (
    <Card
      className="hover:shadow-lg transition-shadow cursor-pointer"
      onClick={() => onSelect?.(pipeline)}
    >
      <CardHeader>
        <CardTitle className="flex items-center justify-between">
          {pipeline.name}
          <Badge variant={pipeline.status === 'active' ? 'success' : 'secondary'}>
            {pipeline.status}
          </Badge>
        </CardTitle>
      </CardHeader>
      <CardContent>
        <p className="text-sm text-muted-foreground">
          {pipeline.description}
        </p>
        <div className="mt-4 flex items-center gap-4 text-xs">
          <span>Created: {new Date(pipeline.createdAt).toLocaleDateString()}</span>
          <span>Runs: {pipeline.runCount}</span>
        </div>
      </CardContent>
    </Card>
  );
};
```

#### Server Component (Next.js 14)

```typescript
// app/(dashboard)/pipelines/page.tsx
import { Suspense } from 'react';
import { getPipelines } from '@/services/pipeline';
import { PipelineList } from '@/components/pipelines/PipelineList';
import { PipelineListSkeleton } from '@/components/pipelines/PipelineListSkeleton';

export default async function PipelinesPage() {
  const pipelines = await getPipelines();

  return (
    <div className="container mx-auto py-8">
      <h1 className="text-3xl font-bold mb-6">Pipelines</h1>
      <Suspense fallback={<PipelineListSkeleton />}>
        <PipelineList pipelines={pipelines} />
      </Suspense>
    </div>
  );
}
```

### Using shadcn/ui Components

```typescript
// components/pipelines/CreatePipelineDialog.tsx
import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@/components/ui/dialog';
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Button } from '@/components/ui/button';
import { PlusCircle } from 'lucide-react';

const formSchema = z.object({
  name: z.string().min(2).max(50),
  description: z.string().min(10).max(500),
});

export function CreatePipelineDialog() {
  const [open, setOpen] = useState(false);

  const form = useForm<z.infer<typeof formSchema>>({
    resolver: zodResolver(formSchema),
    defaultValues: {
      name: '',
      description: '',
    },
  });

  async function onSubmit(values: z.infer<typeof formSchema>) {
    // API call to create pipeline
    console.log(values);
    setOpen(false);
    form.reset();
  }

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        <Button>
          <PlusCircle className="mr-2 h-4 w-4" />
          Create Pipeline
        </Button>
      </DialogTrigger>
      <DialogContent className="sm:max-w-[425px]">
        <DialogHeader>
          <DialogTitle>Create New Pipeline</DialogTitle>
        </DialogHeader>
        <Form {...form}>
          <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
            <FormField
              control={form.control}
              name="name"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Name</FormLabel>
                  <FormControl>
                    <Input placeholder="My Pipeline" {...field} />
                  </FormControl>
                  <FormDescription>
                    Give your pipeline a descriptive name.
                  </FormDescription>
                  <FormMessage />
                </FormItem>
              )}
            />
            <FormField
              control={form.control}
              name="description"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Description</FormLabel>
                  <FormControl>
                    <Textarea
                      placeholder="Describe what your pipeline does..."
                      {...field}
                    />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />
            <Button type="submit">Create</Button>
          </form>
        </Form>
      </DialogContent>
    </Dialog>
  );
}
```

## State Management with Zustand

### Creating a Store

```typescript
// store/pipelineStore.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { Pipeline } from '@/types/pipeline';

interface PipelineState {
  pipelines: Pipeline[];
  selectedPipeline: Pipeline | null;
  isLoading: boolean;
  error: string | null;

  // Actions
  setPipelines: (pipelines: Pipeline[]) => void;
  selectPipeline: (pipeline: Pipeline | null) => void;
  addPipeline: (pipeline: Pipeline) => void;
  updatePipeline: (id: string, updates: Partial<Pipeline>) => void;
  deletePipeline: (id: string) => void;
  setLoading: (isLoading: boolean) => void;
  setError: (error: string | null) => void;
}

export const usePipelineStore = create<PipelineState>()(
  devtools(
    persist(
      (set) => ({
        pipelines: [],
        selectedPipeline: null,
        isLoading: false,
        error: null,

        setPipelines: (pipelines) => set({ pipelines }),
        selectPipeline: (pipeline) => set({ selectedPipeline: pipeline }),
        addPipeline: (pipeline) =>
          set((state) => ({ pipelines: [...state.pipelines, pipeline] })),
        updatePipeline: (id, updates) =>
          set((state) => ({
            pipelines: state.pipelines.map((p) =>
              p.id === id ? { ...p, ...updates } : p
            ),
          })),
        deletePipeline: (id) =>
          set((state) => ({
            pipelines: state.pipelines.filter((p) => p.id !== id),
          })),
        setLoading: (isLoading) => set({ isLoading }),
        setError: (error) => set({ error }),
      }),
      {
        name: 'pipeline-storage',
      }
    )
  )
);
```

### Using the Store

```typescript
// components/pipelines/PipelineManager.tsx
import { useEffect } from 'react';
import { usePipelineStore } from '@/store/pipelineStore';
import { fetchPipelines } from '@/services/pipeline';

export function PipelineManager() {
  const {
    pipelines,
    isLoading,
    error,
    setPipelines,
    setLoading,
    setError,
  } = usePipelineStore();

  useEffect(() => {
    loadPipelines();
  }, []);

  async function loadPipelines() {
    setLoading(true);
    try {
      const data = await fetchPipelines();
      setPipelines(data);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div>
      {pipelines.map((pipeline) => (
        <div key={pipeline.id}>{pipeline.name}</div>
      ))}
    </div>
  );
}
```

## Data Fetching with TanStack Query

### Setup Query Client

```typescript
// app/providers.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // 1 minute
            refetchOnWindowFocus: false,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

### Using Queries

```typescript
// hooks/usePipelines.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { pipelineService } from '@/services/pipeline';

export function usePipelines() {
  return useQuery({
    queryKey: ['pipelines'],
    queryFn: pipelineService.getAll,
  });
}

export function usePipeline(id: string) {
  return useQuery({
    queryKey: ['pipeline', id],
    queryFn: () => pipelineService.getById(id),
    enabled: !!id,
  });
}

export function useCreatePipeline() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: pipelineService.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['pipelines'] });
    },
  });
}
```

## React Flow DAG Editor

```typescript
// components/dag-editor/DagEditor.tsx
import ReactFlow, {
  Node,
  Edge,
  Controls,
  Background,
  MiniMap,
  useNodesState,
  useEdgesState,
  addEdge,
  Connection,
} from 'reactflow';
import 'reactflow/dist/style.css';

const initialNodes: Node[] = [
  {
    id: '1',
    type: 'input',
    data: { label: 'Data Source' },
    position: { x: 250, y: 0 },
  },
  {
    id: '2',
    data: { label: 'Transform' },
    position: { x: 250, y: 100 },
  },
  {
    id: '3',
    type: 'output',
    data: { label: 'Data Target' },
    position: { x: 250, y: 200 },
  },
];

const initialEdges: Edge[] = [
  { id: 'e1-2', source: '1', target: '2' },
  { id: 'e2-3', source: '2', target: '3' },
];

export function DagEditor() {
  const [nodes, setNodes, onNodesChange] = useNodesState(initialNodes);
  const [edges, setEdges, onEdgesChange] = useEdgesState(initialEdges);

  const onConnect = (params: Connection) => {
    setEdges((eds) => addEdge(params, eds));
  };

  return (
    <div style={{ width: '100%', height: '500px' }}>
      <ReactFlow
        nodes={nodes}
        edges={edges}
        onNodesChange={onNodesChange}
        onEdgesChange={onEdgesChange}
        onConnect={onConnect}
        fitView
      >
        <Controls />
        <MiniMap />
        <Background variant="dots" gap={12} size={1} />
      </ReactFlow>
    </div>
  );
}
```

## API Service Layer

```typescript
// services/api.ts
import axios from 'axios';
import { getSession } from 'next-auth/react';

const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor for auth
api.interceptors.request.use(async (config) => {
  const session = await getSession();
  if (session?.accessToken) {
    config.headers.Authorization = `Bearer ${session.accessToken}`;
  }
  return config;
});

// Response interceptor for error handling
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Redirect to login
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

```typescript
// services/pipeline.ts
import api from './api';
import { Pipeline, PipelineCreate } from '@/types/pipeline';

export const pipelineService = {
  async getAll(): Promise<Pipeline[]> {
    const { data } = await api.get('/api/v1/pipelines');
    return data;
  },

  async getById(id: string): Promise<Pipeline> {
    const { data } = await api.get(`/api/v1/pipelines/${id}`);
    return data;
  },

  async create(pipeline: PipelineCreate): Promise<Pipeline> {
    const { data } = await api.post('/api/v1/pipelines', pipeline);
    return data;
  },

  async generate(description: string): Promise<Pipeline> {
    const { data } = await api.post('/api/v1/pipelines:generate', {
      description,
    });
    return data;
  },

  async deploy(id: string): Promise<void> {
    await api.post(`/api/v1/pipelines/${id}:deploy`);
  },

  async run(id: string, params?: any): Promise<void> {
    await api.post(`/api/v1/pipelines/${id}:run`, { params });
  },
};
```

## Testing

### Component Testing

```typescript
// __tests__/components/PipelineCard.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { PipelineCard } from '@/components/pipelines/PipelineCard';

describe('PipelineCard', () => {
  const mockPipeline = {
    id: '1',
    name: 'Test Pipeline',
    description: 'Test description',
    status: 'active',
    createdAt: '2024-01-01',
    runCount: 5,
  };

  it('renders pipeline information', () => {
    render(<PipelineCard pipeline={mockPipeline} />);

    expect(screen.getByText('Test Pipeline')).toBeInTheDocument();
    expect(screen.getByText('Test description')).toBeInTheDocument();
    expect(screen.getByText('active')).toBeInTheDocument();
    expect(screen.getByText(/Runs: 5/)).toBeInTheDocument();
  });

  it('calls onSelect when clicked', () => {
    const onSelect = jest.fn();
    render(<PipelineCard pipeline={mockPipeline} onSelect={onSelect} />);

    fireEvent.click(screen.getByText('Test Pipeline'));
    expect(onSelect).toHaveBeenCalledWith(mockPipeline);
  });
});
```

## Performance Optimization

### Code Splitting

```typescript
// Dynamic imports for heavy components
import dynamic from 'next/dynamic';

const DagEditor = dynamic(() => import('@/components/dag-editor/DagEditor'), {
  ssr: false,
  loading: () => <div>Loading editor...</div>,
});

const MonacoEditor = dynamic(() => import('@monaco-editor/react'), {
  ssr: false,
});
```

### Image Optimization

```typescript
import Image from 'next/image';

export function Logo() {
  return (
    <Image
      src="/logo.png"
      alt="AI ETL Assistant"
      width={200}
      height={50}
      priority // Load immediately
      placeholder="blur" // Show blur while loading
      blurDataURL="..." // Base64 encoded placeholder
    />
  );
}
```

### Memoization

```typescript
import { memo, useMemo, useCallback } from 'react';

const ExpensiveComponent = memo(({ data }) => {
  const processedData = useMemo(() => {
    // Expensive computation
    return data.map(item => ({
      ...item,
      computed: heavyComputation(item),
    }));
  }, [data]);

  const handleClick = useCallback((id) => {
    // Handle click
  }, []);

  return <div>...</div>;
});
```

## Styling Best Practices

### Using Tailwind CSS

```typescript
// Use component variants with cva
import { cva, type VariantProps } from 'class-variance-authority';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  }
);
```

## Related Documentation

- [Development Setup](./setup.md)
- [Testing Guide](./testing.md)
- [API Integration](../api/rest-api.md)
- [Deployment](../deployment/docker.md)

---

[← Back to Development](./README.md) | [Backend Guide →](./backend.md)
# PER-LERP - Personalized Learning-Path Generator

## Project Overview

PER-LERP is an AI-powered education platform that creates individualized learning experiences by orchestrating multiple specialized AI agents. The system uses PocketFlow for orchestration, Google A2A protocol for agent communication, and supports multiple LLM providers with user-configurable model selection. The UI is built with Next.js and ShadCN/UI for a modern, accessible user experience.

### Tech Stack Summary
- **Frontend**: Next.js 14 + ShadCN/UI + Tailwind CSS
- **Backend**: Node.js + TypeScript + Express
- **Database**: PostgreSQL with Apache AGE (graph) + pg_vector (embeddings)
- **Orchestration**: PocketFlow + Google A2A Protocol
- **AI**: Multi-LLM support (OpenAI, Anthropic, Google, Local)
- **Infrastructure**: Docker, Kubernetes, AWS

## Project Structure

```
PER-LERP/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   ├── security-scan.yml
│   │   └── deploy.yml
│   └── CODEOWNERS
├── packages/
│   ├── core/                 # Core business logic and shared utilities
│   │   ├── src/
│   │   │   ├── agents/      # Agent base classes and interfaces
│   │   │   ├── llm/         # Multi-LLM router and integrations
│   │   │   ├── a2a/         # A2A protocol implementation
│   │   │   └── types/       # Shared TypeScript types
│   │   └── package.json
│   ├── agents/              # Individual agent implementations
│   │   ├── goal-setter/
│   │   ├── knowledge-assessor/
│   │   ├── curriculum-generator/
│   │   ├── resource-finder/
│   │   ├── progress-tracker/
│   │   ├── quiz-generator/
│   │   ├── ai-mentor/
│   │   └── git-sync/
│   ├── orchestrator/        # PocketFlow orchestration
│   │   ├── src/
│   │   │   ├── flows/
│   │   │   ├── nodes/
│   │   │   └── store/
│   │   └── package.json
│   ├── api/                 # REST API and WebSocket server
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   ├── middleware/
│   │   │   ├── services/
│   │   │   └── websocket/
│   │   └── package.json
│   ├── web/                 # Next.js web application
│   │   ├── app/
│   │   ├── components/
│   │   │   ├── ui/         # ShadCN/UI components
│   │   │   └── features/   # Feature-specific components
│   │   ├── hooks/
│   │   ├── lib/
│   │   └── package.json
│   └── mobile/              # React Native mobile app (Phase 4)
├── infrastructure/
│   ├── terraform/           # Infrastructure as Code
│   ├── kubernetes/          # K8s manifests
│   └── docker/              # Dockerfiles
├── database/
│   ├── migrations/          # SQL migrations
│   ├── seeds/              # Seed data
│   └── schema.sql          # Database schema
├── scripts/
│   ├── setup-dev.sh        # Development setup
│   ├── run-tests.sh        # Test runner
│   └── deploy.sh           # Deployment script
├── tests/
│   ├── unit/               # Unit tests
│   ├── integration/        # Integration tests
│   ├── e2e/               # End-to-end tests
│   └── fixtures/          # Test data
├── docs/
│   ├── api/               # API documentation
│   ├── architecture/      # Architecture diagrams
│   └── agents/            # Agent specifications
├── .env.example
├── .gitignore
├── .eslintrc.js
├── .prettierrc
├── jest.config.js
├── tsconfig.json
├── package.json
├── pnpm-workspace.yaml
└── README.md
```

## Development Phases

### Phase 1: MVP Foundation (Weeks 1-6)

#### Sprint 1-2: Architecture Setup
```bash
# Initialize monorepo with pnpm
pnpm init
pnpm add -D typescript @types/node eslint prettier jest @types/jest

# Setup core package structure
mkdir -p packages/{core,agents,orchestrator,api,web}/src
mkdir -p infrastructure/{terraform,kubernetes,docker}
mkdir -p database/{migrations,seeds}

# Initialize TypeScript configuration
npx tsc --init

# Setup ESLint and Prettier
npx eslint --init
echo '{"semi": true, "singleQuote": true, "tabWidth": 2}' > .prettierrc

```bash
# Setup ShadCN/UI for the web package
cd packages/web
npx create-next-app@latest . --typescript --tailwind --app
npx shadcn-ui@latest init

# When prompted, choose:
# - TypeScript: Yes
# - Style: Default
# - Base color: Slate
# - CSS variables: Yes

# Install additional dependencies
pnpm add @radix-ui/react-slot class-variance-authority clsx tailwind-merge
pnpm add -D @types/react @types/react-dom

cd ../..

# Initialize git with conventional commits
git init
git add .
git commit -m "feat: initial project structure with ShadCN/UI"
```
```

#### Sprint 3-4: Core Functionality

1. **Implement core types and interfaces**:
```typescript
// packages/core/src/types/index.ts
export interface User {
  id: string;
  email: string;
  profile: UserProfile;
  aiPreferences: AIPreferences;
  gitConfig?: GitConfig;
  subscription: Subscription;
}

export interface LearningGoal {
  id: string;
  userId: string;
  title: string;
  description: string;
  targetDate: Date;
  status: 'active' | 'paused' | 'completed';
}

export interface A2ATaskRequest {
  taskId: string;
  clientAgent: string;
  targetAgent: string;
  skill: string;
  parameters: Record<string, any>;
  context?: TaskContext;
}
```

2. **Setup database with migrations**:
```sql
-- database/init-extensions.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS age;
CREATE EXTENSION IF NOT EXISTS vector;

-- Load AGE extension
LOAD 'age';
SET search_path = ag_catalog, "$user", public;

-- database/migrations/001_initial_schema.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    profile JSONB NOT NULL,
    ai_preferences JSONB NOT NULL DEFAULT '{}',
    git_config JSONB DEFAULT NULL,
    subscription JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_active_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Add indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at);

-- Create graph for learning paths
SELECT create_graph('learning_graph');
```

3. **Implement base agent class**:
```typescript
// packages/core/src/agents/BaseAgent.ts
export abstract class BaseA2AAgent {
  protected agentCard: A2AAgentCard;
  
  abstract handleA2ARequest(request: A2ATaskRequest): Promise<A2ATaskResponse>;
  
  async registerWithA2A(): Promise<void> {
    // Register agent with A2A protocol
  }
}
```

#### Sprint 5-6: Agent Implementation

Create individual agents following the PRD specifications:

```typescript
// packages/agents/goal-setter/src/index.ts
import { BaseA2AAgent } from '@PER-LERP/core';

export class GoalSetterAgent extends BaseA2AAgent {
  constructor() {
    super();
    this.agentCard = {
      name: 'goal-setter',
      version: '1.0.0',
      description: 'Helps users define clear learning goals',
      endpoint: 'https://api.plpg.ai/a2a/agents/goal-setter',
      skills: [
        {
          name: 'set_goal',
          description: 'Create a new learning goal',
          parameters: { /* ... */ }
        }
      ]
    };
  }
  
  async handleA2ARequest(request: A2ATaskRequest): Promise<A2ATaskResponse> {
    // Implementation
  }
}
```

### Phase 2: Beta Features (Weeks 7-10)

#### Sprint 7-8: A2A Protocol & MCP Integration

1. **Implement A2A protocol client**:
```typescript
// packages/core/src/a2a/client.ts
export class A2AClient {
  async sendTask(request: A2ATaskRequest): Promise<A2ATaskResponse> {
    // Implement A2A protocol communication
  }
  
  async discoverAgents(capability: string): Promise<A2AAgentCard[]> {
    // Implement service discovery
  }
}
```

2. **Setup MCP server connections**:
```typescript
// packages/agents/resource-finder/src/mcp-client.ts
export class MCPClient {
  async searchYouTube(query: string): Promise<Resource[]> {
    // Implement YouTube MCP integration
  }
  
  async searchGitHub(query: string): Promise<Resource[]> {
    // Implement GitHub MCP integration
  }
}
```

3. **Implement UI with ShadCN components**:
```tsx
// packages/web/app/dashboard/page.tsx
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Progress } from '@/components/ui/progress'
import { Badge } from '@/components/ui/badge'

export default function Dashboard() {
  return (
    <div className="container mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">Your Learning Journey</h1>
      
      <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
        <Card>
          <CardHeader>
            <CardTitle>Current Goal</CardTitle>
            <CardDescription>Learn Python for Data Science</CardDescription>
          </CardHeader>
          <CardContent>
            <Progress value={65} className="mb-2" />
            <p className="text-sm text-muted-foreground">65% Complete</p>
          </CardContent>
        </Card>
        
        <Card>
          <CardHeader>
            <CardTitle>Today's Focus</CardTitle>
          </CardHeader>
          <CardContent>
            <Badge variant="secondary">NumPy Arrays</Badge>
            <Button className="w-full mt-4">Continue Learning</Button>
          </CardContent>
        </Card>
      </div>
    </div>
  )
}
```

```bash
# Add commonly used ShadCN components
cd packages/web
npx shadcn-ui@latest add card button progress badge dialog form input select
npx shadcn-ui@latest add alert toast table tabs dropdown-menu
cd ../..
```

#### Sprint 9-10: Prompt Management & Multi-LLM

1. **Implement LLM router**:
```typescript
// packages/core/src/llm/router.ts
export class LLMRouter {
  async selectModel(
    agent: string,
    task: TaskContext,
    userPrefs: UserLLMPreferences
  ): Promise<SelectedModel> {
    // Implement model selection logic
  }
}
```

2. **Create prompt management system**:
```typescript
// packages/core/src/prompts/manager.ts
export class PromptManager {
  async getPrompt(agentId: string, version?: string): Promise<PromptTemplate> {
    // Retrieve prompt template
  }
  
  async evolvePrompt(
    basePrompt: PromptTemplate,
    context: TuningContext
  ): Promise<EvolvedPrompt> {
    // Implement auto-tuning
  }
}
```

### Phase 3: Launch Preparation (Weeks 11-12)

#### Sprint 11-12: Polish & Testing

1. **Comprehensive testing suite**:
```typescript
// tests/integration/agents.test.ts
describe('Agent Communication', () => {
  it('should successfully communicate via A2A protocol', async () => {
    const goalSetter = new GoalSetterAgent();
    const assessor = new KnowledgeAssessorAgent();
    
    const request = createA2ARequest('assess_knowledge', { /* ... */ });
    const response = await assessor.handleA2ARequest(request);
    
    expect(response.status).toBe('completed');
  });
});
```

2. **Performance optimization**:
- Implement caching strategies
- Add request batching
- Optimize database queries
- Setup CDN for static assets

## Git Workflow & Best Practices

### Branch Strategy

```bash
# Main branches
main              # Production-ready code
develop           # Integration branch
release/v*        # Release preparation

# Feature branches
feature/agent-*   # Agent development
feature/ui-*      # Frontend features
fix/bug-*         # Bug fixes
chore/*           # Maintenance tasks

# Example workflow
git checkout -b feature/agent-goal-setter
# ... make changes ...
git add .
git commit -m "feat(agent): implement goal-setter agent with A2A support"
git push origin feature/agent-goal-setter
# Create PR to develop
```

### Commit Convention

Follow Conventional Commits specification:

```bash
# Format: <type>(<scope>): <subject>

# Types:
feat     # New feature
fix      # Bug fix
docs     # Documentation
style    # Code style (formatting, semicolons, etc)
refactor # Code refactoring
test     # Adding tests
chore    # Maintenance tasks

# Examples:
git commit -m "feat(auth): add OAuth2 integration for GitHub"
git commit -m "fix(agent): resolve A2A timeout issues"
git commit -m "docs(api): update WebSocket event documentation"
git commit -m "test(llm): add unit tests for model router"
```

### Code Review Process

1. All code must be reviewed before merging
2. Require at least one approval
3. Run automated tests on all PRs
4. Check code coverage (minimum 80%)
5. Verify documentation is updated

## Testing Strategy

### Unit Testing

```typescript
// packages/agents/goal-setter/src/__tests__/goal-setter.test.ts
describe('GoalSetterAgent', () => {
  let agent: GoalSetterAgent;
  
  beforeEach(() => {
    agent = new GoalSetterAgent();
  });
  
  describe('handleA2ARequest', () => {
    it('should create a valid learning goal', async () => {
      const request = createMockA2ARequest('set_goal', {
        title: 'Learn Python',
        description: 'Master Python for data science'
      });
      
      const response = await agent.handleA2ARequest(request);
      
      expect(response.status).toBe('completed');
      expect(response.result).toHaveProperty('goalId');
    });
  });
});
```

```typescript
// packages/web/components/__tests__/dashboard.test.tsx
import { render, screen } from '@testing-library/react'
import { Card } from '@/components/ui/card'
import Dashboard from '@/app/dashboard/page'

describe('Dashboard', () => {
  it('renders learning progress card', () => {
    render(<Dashboard />)
    
    expect(screen.getByText('Your Learning Journey')).toBeInTheDocument()
    expect(screen.getByText('Current Goal')).toBeInTheDocument()
    expect(screen.getByText('65% Complete')).toBeInTheDocument()
  })
  
  it('displays ShadCN progress component', () => {
    const { container } = render(<Dashboard />)
    
    const progressBar = container.querySelector('[role="progressbar"]')
    expect(progressBar).toHaveAttribute('aria-valuenow', '65')
  })
})
```

### Integration Testing

```typescript
// tests/integration/learning-flow.test.ts
describe('Learning Flow Integration', () => {
  it('should complete full learning path creation', async () => {
    // Test the complete flow from goal setting to curriculum generation
  });
});
```

### E2E Testing

```typescript
// tests/e2e/user-journey.test.ts
describe('User Learning Journey', () => {
  it('should allow user to create goal and receive curriculum', async () => {
    // Test complete user journey through UI
  });
});
```

## Linting & Code Quality

### ESLint Configuration

```javascript
// .eslintrc.js
module.exports = {
  parser: '@typescript-eslint/parser',
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:react/recommended',
    'plugin:react-hooks/recommended',
    'next/core-web-vitals',
    'plugin:prettier/recommended',
  ],
  rules: {
    '@typescript-eslint/explicit-function-return-type': 'error',
    '@typescript-eslint/no-explicit-any': 'error',
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
    'no-console': ['warn', { allow: ['warn', 'error'] }],
    'react/prop-types': 'off', // TypeScript handles this
    'react/react-in-jsx-scope': 'off', // Next.js handles this
  },
  settings: {
    react: {
      version: 'detect',
    },
  },
};
```

### Pre-commit Hooks

```json
// package.json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
    }
  },
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write",
      "jest --bail --findRelatedTests"
    ],
    "*.{json,md}": ["prettier --write"],
    "*.css": ["prettier --write"],
    "packages/web/components/**/*.tsx": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

## Environment Configuration

### Development Environment

```bash
# .env.development
NODE_ENV=development
DATABASE_URL=postgresql://dev:dev@localhost:5432/PER-LERP_dev
REDIS_URL=redis://localhost:6379

# LLM Provider Keys
OPENAI_API_KEY=sk-dev-xxx
ANTHROPIC_API_KEY=sk-ant-dev-xxx
GOOGLE_AI_API_KEY=xxx

# A2A Configuration
A2A_REGISTRY_URL=http://localhost:8080
A2A_AUTH_TOKEN=dev-token

# Git Integration
GITHUB_CLIENT_ID=xxx
GITHUB_CLIENT_SECRET=xxx
```

### ShadCN Configuration

```json
// packages/web/components.json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "default",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.ts",
    "css": "app/globals.css",
    "baseColor": "slate",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils"
  }
}
```

```javascript
// packages/web/tailwind.config.ts
import type { Config } from 'tailwindcss'

const config: Config = {
  darkMode: ["class"],
  content: [
    './pages/**/*.{ts,tsx}',
    './components/**/*.{ts,tsx}',
    './app/**/*.{ts,tsx}',
    './src/**/*.{ts,tsx}',
  ],
  theme: {
    container: {
      center: true,
      padding: "2rem",
      screens: {
        "2xl": "1400px",
      },
    },
    extend: {
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
        secondary: {
          DEFAULT: "hsl(var(--secondary))",
          foreground: "hsl(var(--secondary-foreground))",
        },
        destructive: {
          DEFAULT: "hsl(var(--destructive))",
          foreground: "hsl(var(--destructive-foreground))",
        },
        muted: {
          DEFAULT: "hsl(var(--muted))",
          foreground: "hsl(var(--muted-foreground))",
        },
        accent: {
          DEFAULT: "hsl(var(--accent))",
          foreground: "hsl(var(--accent-foreground))",
        },
        popover: {
          DEFAULT: "hsl(var(--popover))",
          foreground: "hsl(var(--popover-foreground))",
        },
        card: {
          DEFAULT: "hsl(var(--card))",
          foreground: "hsl(var(--card-foreground))",
        },
      },
      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
      keyframes: {
        "accordion-down": {
          from: { height: "0" },
          to: { height: "var(--radix-accordion-content-height)" },
        },
        "accordion-up": {
          from: { height: "var(--radix-accordion-content-height)" },
          to: { height: "0" },
        },
      },
      animation: {
        "accordion-down": "accordion-down 0.2s ease-out",
        "accordion-up": "accordion-up 0.2s ease-out",
      },
    },
  },
  plugins: [require("tailwindcss-animate")],
}
export default config
```

### Docker Development Setup

```dockerfile
# docker/Dockerfile.postgres
FROM postgres:16

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    postgresql-server-dev-16 \
    && rm -rf /var/lib/apt/lists/*

# Install Apache AGE
RUN git clone https://github.com/apache/age.git \
    && cd age \
    && make install \
    && cd .. \
    && rm -rf age

# Install pg_vector
RUN git clone --branch v0.6.0 https://github.com/pgvector/pgvector.git \
    && cd pgvector \
    && make \
    && make install \
    && cd .. \
    && rm -rf pgvector

# docker/Dockerfile.dev
FROM node:20-alpine

WORKDIR /app

# Install pnpm
RUN npm install -g pnpm

# Copy workspace files
COPY pnpm-workspace.yaml ./
COPY package.json ./
COPY packages ./packages

# Install dependencies
RUN pnpm install

# Ensure ShadCN/UI dependencies are installed
RUN cd packages/web && pnpm install

# Start development server
CMD ["pnpm", "dev"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    build:
      context: .
      dockerfile: docker/Dockerfile.postgres
    environment:
      POSTGRES_DB: PER-LERP_dev
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init-extensions.sql:/docker-entrypoint-initdb.d/init.sql
    command: >
      postgres 
      -c shared_preload_libraries='age,vector'
      -c search_path='ag_catalog,"$user",public'

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  app:
    build:
      context: .
      dockerfile: docker/Dockerfile.dev
    ports:
      - "3000:3000"  # Web
      - "4000:4000"  # API
    environment:
      DATABASE_URL: postgresql://dev:dev@postgres:5432/PER-LERP_dev
      REDIS_URL: redis://redis:6379
    volumes:
      - .:/app
      - /app/node_modules
    depends_on:
      - postgres
      - redis

volumes:
  postgres_data:
```

## Deployment Strategy

### CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          
      - name: Install pnpm
        run: npm install -g pnpm
        
      - name: Install dependencies
        run: pnpm install
        
      - name: Run linting
        run: pnpm lint
        
      - name: Run type checking
        run: pnpm typecheck
        
      - name: Run tests
        run: pnpm test:ci
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/test
          
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
```

### Production Deployment

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Docker images
        run: |
          docker build -t PER-LERP/postgres -f docker/Dockerfile.postgres .
          docker build -t PER-LERP/api -f docker/Dockerfile.api .
          docker build -t PER-LERP/web -f docker/Dockerfile.web .
          
      - name: Push to registry
        run: |
          echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
          docker push PER-LERP/postgres
          docker push PER-LERP/api
          docker push PER-LERP/web
          
      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f infrastructure/kubernetes/
          kubectl set image deployment/postgres postgres=PER-LERP/postgres:${{ github.sha }}
          kubectl set image deployment/api api=PER-LERP/api:${{ github.sha }}
          kubectl set image deployment/web web=PER-LERP/web:${{ github.sha }}
```

## Security Checklist

- [ ] Environment variables properly secured
- [ ] Database connections use SSL
- [ ] API rate limiting implemented
- [ ] Input validation on all endpoints
- [ ] CORS properly configured
- [ ] Authentication tokens expire appropriately
- [ ] Audit logging for sensitive operations
- [ ] Regular dependency updates
- [ ] Security headers implemented
- [ ] SQL injection protection
- [ ] XSS prevention measures
- [ ] CSRF token validation

## Performance Monitoring

### Key Metrics to Track

```typescript
// packages/api/src/middleware/metrics.ts
export const metricsMiddleware = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    
    metrics.histogram('http_request_duration_ms', duration, {
      method: req.method,
      route: req.route?.path,
      status: res.statusCode,
    });
  });
  
  next();
};
```

### Monitoring Setup

- Use Prometheus for metrics collection
- Grafana for visualization
- Set up alerts for:
  - Response time > 2s
  - Error rate > 1%
  - AI cost per user > $5/month
  - Database connection pool exhaustion
  - Memory usage > 80%
  - Graph query time > 200ms
  - Vector search latency > 100ms
  - Graph traversal depth > 10 levels

## Development Commands

```json
// package.json scripts
{
  "scripts": {
    "dev": "pnpm --parallel dev",
    "build": "pnpm -r build",
    "test": "pnpm -r test",
    "test:ci": "pnpm -r test -- --coverage",
    "lint": "eslint . --ext .ts,.tsx",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "typecheck": "tsc --noEmit",
    "format": "prettier --write '**/*.{ts,tsx,json,md}'",
    "db:migrate": "cd database && pnpm migrate",
    "db:seed": "cd database && pnpm seed",
    "docker:dev": "docker-compose up",
    "docker:build": "docker-compose build",
    "clean": "pnpm -r clean && rm -rf node_modules",
    "ui:add": "cd packages/web && npx shadcn-ui@latest add",
    "ui:theme": "cd packages/web && npx shadcn-ui@latest theme"
  }
}
```

## Troubleshooting Guide

### Common Issues

1. **A2A Connection Failures**
   - Check agent registration status
   - Verify A2A protocol endpoints
   - Review authentication tokens

2. **LLM Rate Limits**
   - Implement exponential backoff
   - Use fallback models
   - Cache common responses

3. **Database Performance**
   - Add appropriate indexes
   - Use connection pooling
   - Implement query optimization
   - Optimize graph traversals with Apache AGE
   - Tune pg_vector for similarity searches

4. **Memory Leaks**
   - Profile with Node.js --inspect
   - Check for event listener cleanup
   - Monitor heap usage

5. **ShadCN/UI Issues**
   - Ensure Tailwind config includes component paths
   - Check theme variables in globals.css
   - Verify component imports from @/components/ui
   - Run `pnpm ui:add <component>` for missing components

## Success Criteria

- [ ] All core agents implemented and tested
- [ ] A2A protocol fully integrated
- [ ] Multi-LLM support working
- [ ] Git repository sync functional
- [ ] ShadCN/UI components integrated and themed
- [ ] 80% test coverage achieved
- [ ] Response time < 2s for all endpoints
- [ ] Security audit passed
- [ ] Documentation complete
- [ ] Beta users onboarded
- [ ] Production deployment stable

---

**Document Version**: 1.2
**Last Updated**: January 2025
**Owner**: PER-LERP Technical Team
**Changes**: Added ShadCN/UI integration, graph database setup with Apache AGE and pg_vector
# Ditto Internal Frontend Architecture

## Overview

The Ditto Internal Frontend is a comprehensive internal tools platform built for managing matchmaking operations, user data, communications, and analytics. This is a React-based single-page application (SPA) that provides various tools for internal team members to manage the Ditto dating platform.

## Tech Stack

### Core Framework
- **React 18.3.1** - UI library for building component-based interfaces
- **TypeScript 5.6.2** - Type-safe JavaScript with static typing
- **Vite 6.0.5** - Modern build tool and dev server with hot module replacement (HMR)
- **React Router 7.1.3** - Client-side routing for navigation

### State Management
The application uses multiple state management approaches:
- **Jotai 2.11.1** - Atomic state management for global UI state (atoms)
- **Zustand 5.0.5** - Lightweight state management (used for AI chat store)
- **React Context API** - For larger feature contexts (UserContext, CanvaContext, etc.)
- **SWR 2.3.3** - Data fetching and caching library

### UI Components & Styling
- **Tailwind CSS 3.4.17** - Utility-first CSS framework
- **Radix UI** - Headless UI component primitives (dialogs, dropdowns, etc.)
- **shadcn/ui** - Pre-built accessible components built on Radix UI
- **Framer Motion 12.6.5** - Animation library
- **Lucide React** - Icon library

### Data Visualization
- **Chart.js 4.4.7** - Canvas-based charting
- **Recharts 2.15.1** - React charting library
- **ECharts 5.6.0** - Advanced data visualization
- **Nivo Funnel** - Funnel chart components

### Backend Integration
- **Axios 1.7.9** - HTTP client for API requests
- **Supabase 2.48.1** - Backend-as-a-Service for authentication and data
- **MeiliSearch** - Fast search engine integration

### AI & Advanced Features
- **OpenAI SDK 4.83.0** - AI integration
- **AI SDK (@ai-sdk/react, @ai-sdk/openai)** - Vercel AI SDK for chat interfaces
- **Model Context Protocol (@modelcontextprotocol/sdk)** - MCP integration
- **Monaco Editor** - Code editor component (VS Code editor)

### Other Notable Libraries
- **React Hook Form 7.54.2** - Form management
- **Zod 3.24.1** - Schema validation
- **Day.js 1.11.13** & **date-fns 4.1.0** - Date manipulation
- **Lodash 4.17.21** - Utility functions
- **Immutable.js 5.0.3** - Immutable data structures

## Project Structure

```
ditto-internal-frontend/
├── src/
│   ├── api/                    # API client and endpoint definitions
│   │   ├── models/            # API model types organized by domain
│   │   ├── api-client.ts      # Axios instance with interceptors
│   │   └── *.api.ts           # Individual API endpoint files
│   ├── atoms/                  # Jotai atoms for global state
│   ├── components/             # Reusable React components
│   │   ├── Common/            # Shared components across features
│   │   ├── ui/                # shadcn/ui components
│   │   ├── animate-ui/        # Animation components
│   │   ├── Sidebar/           # Navigation sidebar
│   │   └── *.tsx              # Other shared components
│   ├── config/                 # Configuration files
│   ├── consts/                 # Constants and enums
│   ├── contexts/               # React Context providers
│   ├── helpers/                # Helper functions
│   ├── hooks/                  # Custom React hooks
│   ├── lib/                    # Library utilities (Supabase, IndexedDB)
│   ├── models/                 # TypeScript type definitions
│   ├── pages/                  # Page components (route components)
│   │   ├── Home/              # Landing page
│   │   ├── Matchmaker/        # Manual matchmaking interface
│   │   ├── MatchmakerV2/      # Updated matchmaker
│   │   ├── MatchingStatus/    # Match tracking
│   │   ├── MessageSender/     # Bulk messaging
│   │   ├── Analytics/         # Analytics dashboards
│   │   ├── AIChat/            # AI chat interface
│   │   └── ...                # Many other feature pages
│   ├── services/               # Business logic services
│   ├── store/                  # Zustand stores
│   ├── styles/                 # Global styles
│   ├── types/                  # Additional TypeScript types
│   ├── utils/                  # Utility functions
│   ├── main.tsx               # Application entry point
│   └── index.scss             # Global styles entry
├── public/                     # Static assets
├── .env                        # Environment variables
├── vite.config.ts             # Vite configuration
├── tsconfig.json              # TypeScript configuration
├── tailwind.config.js         # Tailwind CSS configuration
└── package.json               # Dependencies and scripts
```

## Architecture Patterns

### 1. Routing Architecture

The application uses React Router with a protected route pattern:

```
/ (root) → Redirects to last visited page or /home
├── /login → Public login page
└── Protected Routes (requires authentication)
    ├── /home → Dashboard/landing page
    ├── /matchmaker → Manual matchmaking
    ├── /matchmakerv2 → Updated matchmaker
    ├── /match-status → Match tracking
    ├── /analytics → Analytics dashboard
    ├── /message-sender → Bulk messaging
    ├── /template-editor → Email template editor
    ├── /ai-chat → AI chat interface
    └── ... (20+ other routes)
```

**Key Files:**
- `src/main.tsx` - Route definitions and app initialization
- `src/components/ProtectedRoute.tsx` - Authentication wrapper with provider hierarchy

### 2. Authentication Flow

Authentication is token-based using localStorage:

1. User logs in via `/login` page
2. Token is stored in `localStorage.getItem("token")`
3. `ProtectedRoute` component checks for token presence
4. API client (`src/api/api-client.ts`) automatically adds token to all requests via Axios interceptor
5. 401 responses trigger automatic logout and redirect to login

### 3. State Management Strategy

The application uses a **multi-layered state management approach**:

#### Layer 1: Jotai Atoms (UI State)
Located in `src/atoms/`, these manage simple global UI state:
- `command-menu.atom.ts` - Command menu open/closed
- `profile-drawer.atom.ts` - Profile drawer state
- `filter.atom.ts` - Filter state
- `selected-user.atom.ts` - Currently selected user

**Usage:**
```typescript
import { useAtom } from 'jotai';
import { commandMenuAtom } from '@/atoms/command-menu.atom';

const [isOpen, setIsOpen] = useAtom(commandMenuAtom);
```

#### Layer 2: React Context (Feature State)
Located in `src/contexts/`, these manage complex feature-level state:

- **UserContext** - Central user data management with IndexedDB caching
- **MatchingStatusContext** - Matching status and operations
- **CanvaContext** - Canva integration state
- **EnvContext** - Environment configuration (dev/prod)

**Usage:**
```typescript
import { useUsers } from '@/contexts/UserContext';

const { users, getById, updateUser } = useUsers();
```

#### Layer 3: Zustand Stores (Complex State)
Located in `src/store/`, currently used for:
- **aiChatStore** - AI chat threads, messages, and MCP configuration

**Usage:**
```typescript
import { aiChatStore } from '@/store/aiChatStore';

const threadList = aiChatStore((state) => state.threadList);
```

#### Layer 4: SWR (Server State)
Used for data fetching with automatic caching and revalidation:
```typescript
import useSWR from 'swr';

const { data, error } = useSWR('/api/endpoint', fetcher);
```

### 4. API Architecture

The API layer is organized by domain with a centralized client:

**API Client (`src/api/api-client.ts`):**
- Axios instance with base URL configuration
- Request interceptor: Adds authentication token
- Response interceptor: Handles errors, shows toasts, manages 401 redirects

**API Organization:**
```
src/api/
├── api-client.ts              # Configured Axios instance
├── user.api.ts                # User-related endpoints
├── matching.api.ts            # Matching endpoints
├── email.api.ts               # Email endpoints
├── models/                    # Type definitions for API responses
│   ├── user/
│   ├── profile/
│   ├── chat/
│   └── ...
└── ...
```

**Environment Configuration (`src/consts/api.ts`):**
- Supports dev/prod environment switching
- Base URLs for different services:
  - `API_INTERNAL_BASE_URL` - Main internal API
  - `API_BASE_URL` - Public API
  - `SCHEJ_BASE_URL` - Scheduling service
  - `MEILI_URL` - Search service

### 5. Data Caching Strategy

The application implements sophisticated client-side caching using IndexedDB:

**UserContext Caching (`src/lib/user-db.ts`):**
1. **Initial Load**: Checks IndexedDB for cached user data
2. **Full Sync**: If no cache or environment changed, fetches all users from API
3. **Incremental Sync**: If cache exists, only fetches updated users
4. **Metadata Tracking**: Stores last sync timestamps and counts
5. **Environment Isolation**: Separate caches for dev/prod environments

This approach significantly improves performance by:
- Reducing API calls
- Enabling instant page loads with cached data
- Background updates without blocking UI

### 6. Component Architecture

The application follows a **feature-based component organization**:

#### Shared Components (`src/components/`)
- **ui/** - Base UI primitives (buttons, dialogs, inputs)
- **Common/** - Shared feature components (EmailPreview, MatchingStatus)
- **Sidebar/** - Navigation components
- **ProfileDetail/** - User profile drawer
- **GlobalCommandMenu/** - Command palette (Cmd+K)

#### Page Components (`src/pages/`)
Each page is self-contained with its own:
- Main page component (`*.page.tsx`)
- Sub-components (`components/` subdirectory)
- Page-specific hooks (`hooks/` subdirectory)
- Utilities (`utils/` subdirectory)
- Types (`types/` subdirectory)

**Example: MatchmakerV2 Page**
```
src/pages/MatchmakerV2/
├── MatchmakerV2.page.tsx      # Main page component
├── components/                 # Page-specific components
│   ├── Filters/
│   └── Badges/
├── hooks/                      # Page-specific hooks
└── utils/                      # Page-specific utilities
```

### 7. Provider Hierarchy

The application wraps components in a specific provider hierarchy (from `ProtectedRoute.tsx`):

```
<SidebarProvider>
  <EnvProvider>
    <CanvaProvider>
      <UserProvider>
        <MatchingStatusProvider>
          <TooltipProvider>
            {/* Page Content */}
          </TooltipProvider>
        </MatchingStatusProvider>
      </UserProvider>
    </CanvaProvider>
  </EnvProvider>
</SidebarProvider>
```

This hierarchy ensures:
1. Environment configuration is available first
2. User data loads before matching status
3. All providers are available to child components

### 8. Type Safety

The application uses TypeScript extensively:

**Type Organization:**
- `src/types/` - General types (chat, MCP, etc.)
- `src/models/` - Domain models (user, matching, email, etc.)
- `src/api/models/` - API response types organized by domain

**Type Patterns:**
- Interface definitions for all data structures
- Enum types for constants
- Generic types for reusable patterns
- Zod schemas for runtime validation

## Key Features & Pages

### Matchmaking Tools
1. **Matchmaker V2** (`/matchmakerv2`) - Primary matchmaking interface with filtering and user cards
2. **AI Matchmaker** (`/ai-matchmaker`) - AI-powered match suggestions
3. **User Labeler** (`/user-labeler`) - Tag and categorize users
4. **Matching Status** (`/match-status`) - Track ongoing matches with detailed status
5. **Matching Status by School** (`/matching-status-by-school`) - School-specific tracking

### Operations Tools
1. **Chat Dashboard** (`/chat-dashboard`) - Monitor SMS conversations
2. **Message Sender** (`/message-sender`) - Send bulk emails/SMS
3. **Template Editor** (`/template-editor`) - Edit message templates with Mustache syntax
4. **Email History** (`/email-history`) - View sent email history
5. **Email Approvals** (`/email-approvals`) - Approve outgoing emails

### Analytics
1. **Analytics** (`/analytics-sigma`) - Main analytics dashboard
2. **Analytics Legacy** (`/analytics`) - Original analytics page

### Scheduling
1. **Schedules** (`/schedules`) - Manage event schedules with calendar view
2. **Dates** (`/dates`) - Track confirmed dates

### Developer Tools
1. **Prompt Editor** (`/prompt-editor`) - Edit AI prompts with version control
2. **Asset Editor** (`/asset-editor`) - Manage AI model assets
3. **Ditto Number Config** (`/ditto-number-config`) - Configure phone number settings
4. **AI Chat** (`/ai-chat`) - Internal AI chat interface with MCP support

### Content Creation
1. **Poster Maker** (`/poster-maker`) - Create event posters

### User Management
1. **Users** (`/users`) - Browse and manage user profiles
2. **Feedbacks** (`/feedbacks`) - View user feedback

## Development Workflow

### Running the Application

```bash
# Install dependencies
pnpm install

# Start development server (runs on http://127.0.0.1:5173)
pnpm dev

# Build for production
pnpm build

# Run linting and type checking
pnpm lint

# Preview production build
pnpm preview
```

### Environment Configuration

The application supports dev/prod environment switching:

1. **Via localStorage**: Set `localStorage.setItem('env', 'dev')` or `'prod'`
2. **Via NODE_ENV**: Automatically uses `process.env.NODE_ENV`

Environment affects:
- API base URLs (dev.api.ditto.ai vs api.ditto.ai)
- MeiliSearch index (profile_dev vs profile)
- Data isolation in IndexedDB

### Path Aliases

The application uses TypeScript path aliases for cleaner imports:

```typescript
// Instead of: import { Button } from '../../components/ui/button'
import { Button } from '@/components/ui/button'
```

Configuration in `tsconfig.json`:
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

## Important Patterns & Conventions

### 1. File Naming
- **Pages**: `*.page.tsx` (e.g., `Home.page.tsx`)
- **Components**: PascalCase (e.g., `UserCard.tsx`)
- **API files**: `*.api.ts` (e.g., `user.api.ts`)
- **Models**: `*.model.ts` (e.g., `user-profile.model.ts`)
- **Atoms**: `*.atom.ts` (e.g., `command-menu.atom.ts`)
- **Hooks**: `use*.ts` (e.g., `useUsers.ts`)

### 2. Component Patterns

**Functional Components with TypeScript:**
```typescript
import { FC } from 'react';

interface MyComponentProps {
  title: string;
  onAction: () => void;
}

const MyComponent: FC<MyComponentProps> = ({ title, onAction }) => {
  return <div>{title}</div>;
};

export default MyComponent;
```

### 3. API Call Pattern

```typescript
import { ApiClient } from '@/api/api-client';
import { toast } from 'sonner';

export const fetchUsers = async () => {
  try {
    const response = await ApiClient.get('/users');
    return response.data;
  } catch (error) {
    // Error handling is automatic via interceptor
    // But you can add custom handling here
    throw error;
  }
};
```

### 4. Toast Notifications

The application uses Sonner for toast notifications:

```typescript
import { toast } from 'sonner';

// Success
toast.success('Operation completed');

// Error
toast.error('Something went wrong', {
  description: 'Error details here'
});

// Info
toast.info('Information message');
```

### 5. Form Handling

Forms use React Hook Form with Zod validation:

```typescript
import { useForm } from 'react-hook-form';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
});

type FormData = z.infer<typeof schema>;

const MyForm = () => {
  const { register, handleSubmit } = useForm<FormData>();
  
  const onSubmit = (data: FormData) => {
    // Handle form submission
  };
  
  return <form onSubmit={handleSubmit(onSubmit)}>...</form>;
};
```

## Performance Considerations

### 1. User Data Loading
- Uses IndexedDB for client-side caching
- Implements incremental updates to minimize API calls
- Shows loading states with progress indicators
- Lazy loads user data on demand

### 2. Code Splitting
- React Router automatically code-splits by route
- Large pages load independently
- Reduces initial bundle size

### 3. Virtualization
- Uses `react-window` and `react-virtualized` for large lists
- Only renders visible items in long lists
- Improves performance with thousands of users

### 4. Memoization
- Uses `useMemo` and `useCallback` for expensive computations
- Immutable.js for efficient data structure updates

## Security Considerations

1. **Authentication**: Token-based with automatic logout on 401
2. **API Security**: All requests include Bearer token
3. **Environment Variables**: Sensitive keys in `.env` file
4. **CORS**: Backend handles CORS configuration
5. **Input Validation**: Zod schemas validate user input

## Common Tasks for New Engineers

### Adding a New Page

1. Create page component in `src/pages/NewFeature/NewFeature.page.tsx`
2. Add route in `src/main.tsx`:
```typescript
<Route path="/new-feature" element={<NewFeature />} />
```
3. Add navigation item in `src/components/Sidebar/Sidebar.tsx`
4. Create any page-specific components in `src/pages/NewFeature/components/`

### Adding a New API Endpoint

1. Create API file: `src/api/new-feature.api.ts`
2. Define types in `src/api/models/new-feature/`
3. Use ApiClient for requests:
```typescript
import { ApiClient } from '@/api/api-client';

export const getFeatureData = async () => {
  const response = await ApiClient.get('/feature');
  return response.data;
};
```

### Adding Global State

1. For simple UI state: Create Jotai atom in `src/atoms/`
2. For complex feature state: Create Context in `src/contexts/`
3. For persistent state: Use Zustand store in `src/store/`

### Styling Components

1. Use Tailwind utility classes for most styling
2. Use shadcn/ui components for common UI patterns
3. Create custom components in `src/components/ui/` if needed
4. Use `cn()` utility for conditional classes:
```typescript
import { cn } from '@/lib/utils';

<div className={cn('base-class', isActive && 'active-class')} />
```

## Troubleshooting

### Common Issues

1. **"User data not loaded" toast**: Click "Load Now" to fetch user data from API
2. **401 Unauthorized**: Token expired, will auto-redirect to login
3. **Environment mismatch**: Check localStorage env setting matches desired environment
4. **Type errors**: Run `pnpm lint` to see all TypeScript errors
5. **Build errors**: Clear node_modules and reinstall: `rm -rf node_modules && pnpm install`

### Debugging Tips

1. **React DevTools**: Install browser extension to inspect component tree
2. **Network Tab**: Monitor API calls and responses
3. **Console Logs**: Check browser console for errors
4. **IndexedDB**: Use browser DevTools to inspect cached data
5. **localStorage**: Check for token and env settings

## Additional Resources

- **Documentation**: https://siyuan.vibee.ai
- **Server Status**: https://status.ditt.ai
- **Supabase Dashboard**: Access via VITE_SUPABASE_URL
- **MeiliSearch**: Search functionality powered by MeiliSearch

## Architecture Decisions

### Why Multiple State Management Solutions?

1. **Jotai**: Lightweight for simple global UI state (modals, drawers)
2. **Context**: Best for feature-scoped state with complex logic (users, matching)
3. **Zustand**: Persistent state with middleware support (AI chat)
4. **SWR**: Server state with automatic revalidation

### Why IndexedDB for User Data?

- Handles large datasets (3000+ users)
- Persists across sessions
- Enables offline-first experience
- Faster than repeated API calls

### Why Vite over Create React App?

- Faster development server with instant HMR
- Better build performance
- Modern tooling with native ESM support
- Smaller bundle sizes

## Future Considerations

As the application grows, consider:

1. **Code splitting**: Further split large pages into smaller chunks
2. **Service workers**: Add offline support and caching
3. **Testing**: Add unit and integration tests
4. **Monitoring**: Add error tracking (Sentry) and analytics
5. **Documentation**: Keep this document updated with new features
6. **Performance monitoring**: Track bundle size and load times

---

**Last Updated**: November 2025
**Maintained By**: Ditto Engineering Team

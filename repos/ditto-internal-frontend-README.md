# Ditto Internal Frontend

A comprehensive internal tools platform for managing matchmaking operations, user data, communications, and analytics for the Ditto dating platform.

## Quick Start

```bash
# Install dependencies
pnpm install

# Start development server
pnpm dev

# Build for production
pnpm build

# Run linting and type checking
pnpm lint
```

The application runs on `http://127.0.0.1:5173` by default.

## Tech Stack

- **React 18.3.1** + **TypeScript 5.6.2** - Type-safe component development
- **Vite 6.0.5** - Fast build tool and dev server
- **React Router 7.1.3** - Client-side routing
- **Tailwind CSS 3.4.17** - Utility-first styling
- **Radix UI** + **shadcn/ui** - Accessible component library
- **Supabase** - Authentication and backend services
- **MeiliSearch** - Fast search functionality

### State Management
- **Jotai** - Atomic UI state
- **Zustand** - Persistent state
- **React Context** - Feature-level state
- **SWR** - Server state with caching

### Data Visualization
- Chart.js, Recharts, ECharts, Nivo

## Project Structure

```
src/
├── api/          # API clients and models
├── atoms/        # Jotai global state atoms
├── components/   # Shared components + UI library
├── contexts/     # React Context providers
├── pages/        # Route components (20+ features)
├── store/        # Zustand stores
├── hooks/        # Custom React hooks
├── lib/          # Utilities (Supabase, IndexedDB)
├── models/       # TypeScript type definitions
└── utils/        # Helper functions
```

## Key Features

### Matchmaking Tools
- **Matchmaker V2** - Primary matchmaking interface
- **AI Matchmaker** - AI-powered match suggestions
- **User Labeler** - Tag and categorize users
- **Matching Status** - Track ongoing matches
- **Matching Status by School** - School-specific tracking

### Operations Tools
- **Chat Dashboard** - Monitor SMS conversations
- **Message Sender** - Send bulk emails/SMS
- **Template Editor** - Edit message templates
- **Email History** - View sent email history

### Analytics & Insights
- **Analytics Dashboard** - Platform statistics and metrics
- **Schedules** - Manage event schedules
- **Dates** - Track confirmed dates

### Developer Tools
- **Prompt Editor** - Edit AI prompts with version control
- **Asset Editor** - Manage AI model assets
- **Ditto Number Config** - Configure phone settings
- **AI Chat** - Internal AI chat interface with MCP support

## Environment Configuration

The application supports dev/prod environment switching:

```javascript
// Via localStorage
localStorage.setItem('env', 'dev') // or 'prod'

// Via NODE_ENV (automatic)
process.env.NODE_ENV
```

Environment affects:
- API base URLs (`dev.api.ditto.ai` vs `api.ditto.ai`)
- MeiliSearch index (`profile_dev` vs `profile`)
- Data isolation in IndexedDB

## Authentication

Token-based authentication using localStorage:
1. Login via `/login` page
2. Token stored in `localStorage`
3. Automatic token injection in API requests
4. Auto-logout on 401 responses

## Development Guidelines

### Path Aliases
Use `@/` for cleaner imports:
```typescript
import { Button } from '@/components/ui/button'
```

### File Naming Conventions
- Pages: `*.page.tsx`
- Components: `PascalCase.tsx`
- API files: `*.api.ts`
- Models: `*.model.ts`
- Atoms: `*.atom.ts`
- Hooks: `use*.ts`

### Component Pattern
```typescript
import { FC } from 'react';

interface MyComponentProps {
  title: string;
}

const MyComponent: FC<MyComponentProps> = ({ title }) => {
  return <div>{title}</div>;
};

export default MyComponent;
```

## Performance Features

- **IndexedDB Caching** - Client-side caching for 3000+ user profiles
- **Incremental Sync** - Only fetch updated data
- **Code Splitting** - Route-based automatic splitting
- **Virtualization** - Efficient rendering of large lists
- **Lazy Loading** - On-demand data loading

## Architecture Documentation

For detailed architecture information, including:
- Complete routing architecture
- State management strategies
- API patterns and conventions
- Component organization
- Security considerations
- Common tasks for new engineers
- Troubleshooting guide

See **[ARCHITECTURE.md](./ARCHITECTURE.md)** for comprehensive documentation.

## Resources

- **Documentation**: https://siyuan.vibee.ai
- **Server Status**: https://status.ditt.ai
- **Supabase Dashboard**: Access via `VITE_SUPABASE_URL`

## Troubleshooting

### Common Issues

**"User data not loaded" toast**
- Click "Load Now" to fetch user data from API

**401 Unauthorized**
- Token expired, will auto-redirect to login

**Environment mismatch**
- Check `localStorage` env setting matches desired environment

**Type errors**
- Run `pnpm lint` to see all TypeScript errors

**Build errors**
- Clear and reinstall: `rm -rf node_modules && pnpm install`

## Contributing

This is an internal tool for the Ditto team. For questions or issues, refer to the internal documentation or contact the engineering team.

---

**Last Updated**: November 2025

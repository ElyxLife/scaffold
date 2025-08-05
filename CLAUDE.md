# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Backend Development (Python FastAPI)
- **Install dependencies**: `cd backend && uv sync`
- **Run server locally**: `cd backend && ./run_local.sh` (loads .env.local and sets PYTHONPATH)
- **Alternative run**: `cd backend && uv run uvicorn app:app --reload`
- **Run tests**: `cd backend && uv run pytest`
- **Run tests with coverage**: `uv run pytest --cov=backend` (from project root)
- **Database migrations**: `cd backend && uv run alembic upgrade head`
- **Lint check**: Configure via ruff in pyproject.toml (line-length 120, Python 3.13)
- **Type check**: mypy configured in pyproject.toml

### Frontend Development (React/Vite)
- **Install dependencies**: `cd ui && npm install`
- **Run dev server**: `cd ui && npm run dev` (usually runs on port 5173)
- **Build for production**: `cd ui && npm run build`
- **Run tests**: `cd ui && npm run test`
- **Lint check**: `cd ui && npm run lint`

### Infrastructure & Deployment
- **Authentication Setup**: Before first deployment, configure Google OAuth:
  1. Go to [Google Cloud Console - Credentials](https://console.cloud.google.com/apis/credentials)
  2. Create OAuth 2.0 Client ID (Web application)
  3. Add redirect URI: `https://{project-id}.firebaseapp.com/__/auth/handler`
  4. Update `terraform.tfvars.[environment]` with client_id
  5. Update `terraform.auto.tfvars.secrets.[environment]` with client_secret
- **Service Account Setup**: `cd terraform && ./setup-terraform-auth.sh [PROJECT_ID] [USER_EMAIL]`
  - Optional: Sets up service account authentication for Terraform
  - Fixes potential OAuth client conflicts during terraform operations
- **Deploy to GCP**: `cd terraform && ./deploy.sh [environment] [version-tag]`
  - Examples: `./deploy.sh prod`, `./deploy.sh staging v1.0.1`
- **Run database migrations on Cloud**: `cd terraform && ./run_migrations.sh PROJECT_ID`
- **Build and push Docker image**: `cd terraform && ./build_and_push_image.sh VERSION_TAG PROJECT_ID`
- **Manual Terraform commands**: When running terraform directly (instead of deploy.sh), always include both variable files:
  - `terraform plan -var-file=terraform.tfvars.[environment] -var-file=terraform.auto.tfvars.secrets.[environment]`
  - `terraform apply -var-file=terraform.tfvars.[environment] -var-file=terraform.auto.tfvars.secrets.[environment]`

## Architecture Overview

### Core Components
- **Backend** (`backend/`): FastAPI application handling WhatsApp webhooks and API endpoints
- **Frontend** (`ui/`): React/TypeScript UI for concierge team management
- **Infrastructure** (`terraform/`): Terraform modules for GCP deployment
- **Cloud Function** (`cloud_comm_pubsub/`): Pub/Sub webhook handling

### Key Backend Files
- `app.py`: Main FastAPI application with all API endpoints
- `whatsapp_handler.py`: Core logic for processing WhatsApp messages and sending responses
- `models.py`: SQLAlchemy database models
- `config.py`: Configuration management with environment variables
- `db_manager.py`: Database connection and session management

### Database Architecture
Uses PostgreSQL with SQLAlchemy ORM. Key tables:
- `users`: WhatsApp users with phone numbers
- `groups`: Internal groups (representing DM threads with WhatsApp users)
- `group_members`: Many-to-many relationship between users and groups
- `messages`: All messages with source tracking (whatsapp_user, ui_concierge, system_auto)
- `concierge_team_members`: Staff who can respond through the UI
- `media_attachments`: File attachments stored in Google Cloud Storage

### Message Flow
1. WhatsApp user sends message → Meta webhook → Backend webhook endpoint
2. `whatsapp_handler.py` processes message, determines group assignment
3. Message stored in database, auto-responses generated
4. Messages distributed to all group members via WhatsApp Cloud API
5. Real-time updates sent to UI via Server-Sent Events

### Frontend Architecture
- **Framework**: React 19 with TypeScript, Vite build tool
- **UI Library**: shadcn/ui components with Tailwind CSS
- **State Management**: React Context for current user selection
- **Routing**: React Router for navigation
- **Real-time Updates**: Server-Sent Events for live message updates
- **API Communication**: Axios for HTTP requests

## Key Development Patterns

### Backend Development
- Use uv for dependency management (not pip/poetry)
- SQLAlchemy models with proper type hints
- FastAPI with async/await patterns
- Environment-based configuration via config.py
- Comprehensive test coverage with pytest

### Frontend Development
- TypeScript for all components
- shadcn/ui for consistent UI components
- Tailwind CSS for styling
- Path aliases: `@/*` points to `src/`
- Component organization: pages/, components/, services/, hooks/

### Database Operations
- Use Alembic for schema migrations
- Proper foreign key relationships and constraints
- Transaction management for multi-table operations
- Connection pooling via SQLAlchemy

### Error Handling
- Comprehensive error logging in whatsapp_handler.py
- Request signature validation for webhooks (bypassable for tests)
- Graceful degradation for failed WhatsApp API calls
- System messages for delivery failures

## Environment Configuration

### Required Environment Variables
```bash
# Database
DATABASE_URL="postgresql://user:password@host:port/database"

# WhatsApp Cloud API
WHATSAPP_ACCESS_TOKEN="your_access_token"
WHATSAPP_PHONE_NUMBER_ID="your_phone_number_id"
WHATSAPP_WEBHOOK_VERIFY_TOKEN="your_verify_token"

# Google Cloud
PROJECT_ID="your-gcp-project"
GCS_BUCKET_NAME="your-media-bucket"

# Application
DEBUG=true
PORT=8000
CORS_ALLOWED_ORIGINS="http://localhost:3000,http://localhost:5173"
```

### Secret Management
- Sensitive values (API keys, tokens, passwords) are stored in `terraform.auto.tfvars.secrets` (gitignored)
- Use `terraform.auto.tfvars.secrets.example` as a template for required secrets
- Non-sensitive configuration goes in `terraform.tfvars`
- Terraform reads secrets from `.tfvars.secrets` and creates/manages them in Google Secret Manager
- **Never commit actual secret values to git**

## Testing Guidelines

### Backend Testing
- Tests use in-memory SQLite for isolation
- Mock WhatsApp API calls in unit tests
- Use pytest fixtures for database setup
- Request signature validation can be bypassed with monkeypatch
- Current coverage: ~74% overall

### Frontend Testing
- Vitest with React Testing Library
- Component testing with proper mocking
- API client testing with axios mocking

## Known Issues & Technical Debt

1. **SQLAlchemy Type Hints**: Persistent mypy errors in app.py and whatsapp_handler.py related to SQLAlchemy type checking
2. **Markdown Conversion**: Flaky conversion between Tiptap HTML and WhatsApp Markdown in frontend
3. **Firebase Config Files**: .firebaserc and firebase.json are auto-generated by Terraform - do not edit manually

## Infrastructure Notes

- **Deployment**: Fully automated via Terraform with GCP Cloud Run, Cloud SQL, and Firebase Hosting
- **State Management**: Terraform uses remote state stored in GCS bucket (see backend.tf)
- **Docker**: Backend containerized with multi-stage builds
- **Secrets**: Managed via Google Secret Manager, referenced in Terraform
- **Monitoring**: Basic Cloud Run logging and monitoring enabled

## Secrets and Authentication

### Zero GitHub Secrets Required! 🎉

This project requires **ZERO GitHub repository secrets** thanks to modern security practices:

**Project IDs** (and **Project Numbers**) → Read from `terraform.tfvars.staging` and `terraform.tfvars.prod`  \
**CI Note**: GitHub Actions also extracts `project_number` from these files; it is **not** referenced directly in Terraform code, so we intentionally keep it only in the tfvars.
**Authentication** → Handled securely via Workload Identity Federation  
**Meta API Secrets** → Stored in Google Secret Manager  
**Database Passwords** → Auto-generated by Terraform  

### How Secrets Are Managed

#### 🔐 Workload Identity Federation
- **GitHub Actions** authenticates using OIDC tokens (no service account keys!)
- **Short-lived tokens** that expire after each workflow run
- **Repository-specific** authentication for enhanced security
- **Automatic rotation** - no manual key management required

#### 🗝️ Meta API Secrets (WhatsApp Integration)
- **Local Development**: Read from `terraform.auto.tfvars.secrets.staging/prod` files (never committed)
- **Initial Deployment**: Terraform stores secrets in Google Secret Manager
- **Runtime**: Applications read secrets via Terraform data sources
- **CI/CD**: GitHub Actions automatically reads from Secret Manager

#### 🔑 Database Credentials
- **Fully Automated**: Terraform generates secure 32-character passwords using `random_password` resource
- **Persistent**: Passwords stored in Terraform state and reused across deployments
- **Secure**: Passwords never appear in logs, scripts, or deployment processes
- **No Manual Management**: Zero password rotation or storage required

#### 📁 Configuration File Management

**✅ Committed to Repository:**
```bash
terraform.tfvars.staging        # Project configuration
terraform.tfvars.prod          # Project configuration
```

**❌ Never Committed (Local Only):**
```bash
terraform.auto.tfvars.secrets.staging    # Meta API secrets for initial setup
terraform.auto.tfvars.secrets.prod      # Meta API secrets for initial setup
```

**🔒 Stored in Google Secret Manager:**
```bash
META_WABA_ACCESS_TOKEN    # Automatically managed after initial deployment
META_VERIFY_TOKEN        # Automatically managed after initial deployment
META_APP_SECRET         # Automatically managed after initial deployment
```

### Security Benefits

- 🔐 **Zero Long-lived Keys**: No service account keys stored anywhere
- ⏰ **Auto-expiring Tokens**: Authentication tokens expire after each run
- 🎯 **Repository-specific**: Only your specific repository can authenticate
- 📊 **Better Audit Trail**: Detailed identity information in GCP logs
- 🏗️ **Future-proof**: Industry standard for cloud authentication

**Result**: Reduced from 10+ secrets to **ZERO secrets** while significantly improving security! 🚀

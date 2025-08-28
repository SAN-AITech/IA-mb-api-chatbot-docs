# Development Guide

## Local Development Setup

### Prerequisites

#### **Required Software**

- **Python 3.13** - Backend runtime
- **UV Package Manager** - Python dependency management (enterprise JFrog repository)
- **Node.js 22.14+** - Frontend runtime
- **Angular CLI** - Frontend development tools
- **AWS CLI** - AWS service access
- **Git** - Version control

#### **Installation Commands**

```bash
# Install UV (if not already installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install Node.js (via nvm recommended)
nvm install 22.14
nvm use 22.14

# Install Angular CLI
npm install -g @angular/cli

# Verify installations
uv --version
node --version
ng version
aws --version
```

### **Project Setup**

#### **1. Clone Repository**

```bash
git clone <repository-url>
cd IA-mb-api-chatbot
```

#### **2. Backend Setup (Python)**

```bash
# Install Python dependencies from enterprise JFrog repository
uv sync

# Verify dependencies
uv tree
```

#### **3. Frontend Setup (Angular)**

```bash
# Navigate to web directory
cd web

# Install Node.js dependencies
npm install

# Verify Angular setup
ng version

# Return to project root
cd ..
```

### **Environment Configuration**

#### **AWS Credentials**

```bash
# Configure AWS CLI (required for DynamoDB and Bedrock access)
aws configure

# Or use environment variables
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key
export AWS_DEFAULT_REGION=eu-west-1
```

#### **Application Environment Variables**

```bash
# Backend configuration
export AWS_DEFAULT_REGION=eu-west-1
export INTERACTIONS_TABLE=chatbot-interactions-dev
export SESSIONS_TABLE=chatbot-sessions-dev
export BEDROCK_MODEL_ID=anthropic.claude-3-sonnet-20240229-v1:0
export BEDROCK_REGION=eu-west-1

# Optional: Debug logging
export LOG_LEVEL=DEBUG
```

#### **Configuration Files**

```bash
# Development configuration
config/dev.yaml       # Backend settings for development
web/proxy.conf.json   # Frontend proxy configuration for local development
```

## Running the Application

### **Development Mode (Recommended)**

#### **Terminal 1: Backend Server**

```bash
# Set environment variables
export AWS_DEFAULT_REGION=eu-west-1
export INTERACTIONS_TABLE=chatbot-interactions-dev
export SESSIONS_TABLE=chatbot-sessions-dev

# Start backend with UV
uv run python chatbot_api/run.py
```

**Backend will be available at:** `http://localhost:8000`

**API Documentation:** `http://localhost:8000/docs` (Swagger UI)

#### **Terminal 2: Frontend Development Server**

```bash
# Navigate to web directory
cd web

# Start Angular development server
ng serve --verbose
```

**Frontend will be available at:** `http://localhost:4200`

**Proxy Configuration:** Automatically proxies `/chatbot/api/*` to backend

### **Production Mode**

```bash
# Use provided production script
./start-server.sh
```

This script includes:

- SSL certificate configuration
- Production environment settings
- Proper logging setup

## Development Workflow

### **Code Structure**

```text
IA-mb-api-chatbot/
├── chatbot_api/           # Python FastAPI backend
│   ├── controllers/       # API endpoint handlers
│   │   ├── conversation.py    # Conversation management
│   │   ├── sessions.py        # Session handling
│   │   └── health.py          # Health checks
│   ├── services/          # Business logic
│   │   ├── agent.py           # LangGraph agent integration
│   │   ├── aws.py             # DynamoDB operations
│   │   └── conversation.py    # Conversation processing
│   ├── domain/            # Data models and DTOs
│   │   ├── requests/          # Request models
│   │   └── responses/         # Response models
│   ├── main.py           # FastAPI application setup
│   └── run.py            # Application entry point
├── web/                  # Angular frontend
│   ├── src/app/         # Angular application
│   │   ├── services/        # API communication
│   │   ├── components/      # UI components
│   │   └── models/          # TypeScript interfaces
│   ├── proxy.conf.json  # Development proxy setup
│   └── package.json     # Node.js dependencies
├── config/              # Environment configurations
│   ├── dev.yaml         # Development settings
│   └── qa.yaml          # QA environment settings
├── test/                # Unit and integration tests
└── docs/                # Technical documentation
```

### **Development Commands**

#### **Backend Development**

```bash
# Run backend server
uv run python chatbot_api/run.py

# Run specific module
uv run python -m chatbot_api.main

# Run tests
uv run pytest test/

# Check code formatting
uv run black chatbot_api/
uv run flake8 chatbot_api/

# Type checking
uv run mypy chatbot_api/
```

#### **Frontend Development**

```bash
cd web

# Development server
ng serve --verbose

# Build for production
ng build --prod

# Run tests
ng test

# Run linting
ng lint

# Check for updates
ng update
```

### **Dependency Management**

#### **Python Dependencies (UV)**

```bash
# Add new dependency
uv add package-name

# Add development dependency
uv add --dev package-name

# Update dependencies
uv sync

# Show dependency tree
uv tree

# Export requirements (if needed)
uv export --format requirements-txt > requirements.txt
```

#### **Node.js Dependencies**

```bash
cd web

# Add new dependency
npm install package-name

# Add development dependency
npm install --save-dev package-name

# Update dependencies
npm update

# Audit for vulnerabilities
npm audit
```

## Testing

### **Backend Testing**

```bash
# Run all tests
uv run pytest test/

# Run specific test file
uv run pytest test/services/test_conversation.py

# Run with coverage
uv run pytest test/ --cov=chatbot_api

# Run integration tests
uv run pytest test/ -m integration
```

### **Frontend Testing**

```bash
cd web

# Run unit tests
ng test

# Run e2e tests
ng e2e

# Generate test coverage
ng test --code-coverage
```

## Debugging

### **Backend Debugging**

```bash
# Enable debug logging
export LOG_LEVEL=DEBUG

# Run with Python debugger
uv run python -m pdb chatbot_api/run.py

# Check AWS connectivity
aws dynamodb describe-table --table-name chatbot-interactions-dev
aws bedrock list-foundation-models --region eu-west-1
```

### **Frontend Debugging**

```bash
# Angular CLI debug information
ng version

# Verbose output
ng serve --verbose

# Check proxy configuration
cat web/proxy.conf.json
```

### **Common Issues**

#### **AWS Connection Issues**

```bash
# Verify AWS credentials
aws sts get-caller-identity

# Check IAM permissions
aws iam get-user

# Test DynamoDB access
aws dynamodb list-tables
```

#### **UV Package Issues**

```bash
# Clear UV cache
uv cache clean

# Reinstall dependencies
uv sync --force

# Check for conflicts
uv tree --show-conflicts
```

#### **CORS Issues**

```bash
# Check frontend proxy configuration
cat web/proxy.conf.json

# Verify backend CORS settings in config/dev.yaml
```

## Advanced Development

### **Hot Reloading**

Both backend and frontend support hot reloading during development:

- **Backend**: FastAPI automatically reloads on code changes
- **Frontend**: Angular CLI watches for changes and rebuilds

### **API Development**

```bash
# Interactive API documentation
http://localhost:8000/docs          # Swagger UI
http://localhost:8000/redoc         # ReDoc

# Test API endpoints
curl -X POST http://localhost:8000/chatbot/api/v1/sessions \
  -H "Content-Type: application/json" \
  -d '{"clientId": "test-client"}'
```

### **Database Development**

```bash
# Local DynamoDB development (if needed)
aws dynamodb create-table --cli-input-json file://table-definition.json

# Query development data
aws dynamodb scan --table-name chatbot-interactions-dev --max-items 10
```

## Performance Monitoring

### **Backend Monitoring**

- **Metrics**: Available at `/health` endpoint
- **Logs**: Structured JSON logging to stdout
- **Tracing**: OpenTelemetry integration for distributed tracing

### **Frontend Monitoring**

- **Build Analysis**: `ng build --stats-json` followed by webpack-bundle-analyzer
- **Performance**: Chrome DevTools Lighthouse audits
- **Network**: Monitor API calls in browser DevTools

## Troubleshooting

### **Common Problems**

1. **Port Already in Use**

   ```bash
   # Backend (port 8000)
   lsof -ti:8000 | xargs kill
   
   # Frontend (port 4200)
   lsof -ti:4200 | xargs kill
   ```

2. **AWS Permission Denied**

   ```bash
   # Check IAM policy attached to your user/role
   aws iam list-attached-user-policies --user-name your-username
   ```

3. **UV Installation Issues**

   ```bash
   # Reinstall UV
   curl -LsSf https://astral.sh/uv/install.sh | sh
   source ~/.bashrc
   ```

4. **Angular CLI Issues**

   ```bash
   # Clear npm cache
   npm cache clean --force
   
   # Reinstall Angular CLI
   npm uninstall -g @angular/cli
   npm install -g @angular/cli@latest
   ```

This development guide provides everything needed to set up and work with the IA MB API Chatbot locally using the actual tooling (UV, Angular CLI) and deployment methods.

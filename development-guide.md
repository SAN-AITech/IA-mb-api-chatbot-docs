[← Back to Documentation Home](README.md)

# Development Guide

## Local Development Setup

### **Prerequisites**

#### **Required Software**

- **Python 3.11+** - Backend runtime
- **UV Package Manager** - Python dependency management (enterprise JFrog repository)
- **AWS CLI** - AWS service access
- **Git** - Version control

#### **Installation Commands**

```bash
# Install UV (if not already installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Verify installations
uv --version
aws --version
```

## Project Setup

### **1. Clone Repository**

```bash
git clone <repository-url>
cd IA-mb-api-chatbot
```

### **2. Obtain JFrog API Token**

**Required for accessing enterprise Python packages:**

1. **Navigate to JFrog Portal**: Go to [https://gluoneurope.jfrog.io/ui/login](https://gluoneurope.jfrog.io/ui/login)
2. **Login**: Use SAML SSO with your corporate credentials
3. **Generate Token**:
   - Click your user icon (top right)
   - Select "Edit Profile"
   - Click "Generate an Identity Token"
   - **Important**: Copy and save the token securely
4. **Token Expiration**: Tokens expire every 3 months - set a calendar reminder

### **3. Backend Setup (Python)**

```bash
# Set JFrog credentials (required before uv sync)
export UV_INDEX_PRIVATE_REGISTRY_USERNAME=x756900@opendigitalservices.com
export UV_INDEX_PRIVATE_REGISTRY_PASSWORD=<your-jfrog-token>

# Install Python dependencies from enterprise JFrog repository
uv sync

# Verify dependencies
uv tree
```

## Environment Configuration

### **AWS Parameter Store (Primary)**

Configuration automatically loads from AWS Parameter Store using `ok-config`:

```bash
# No manual configuration needed - parameters load automatically on startup
# Explore your parameters at:
# https://eu-west-1.console.aws.amazon.com/systems-manager/parameters/?region=eu-west-1&tab=Table
```

**Parameter Store Benefits:**

- **Centralized Configuration**: All environments managed in one place
- **Secure Storage**: Sensitive values encrypted at rest
- **Automatic Loading**: No manual environment setup required
- **Service Integration**: Integrates with SERVICE_NAME environment variable

### **Optional .env Override (Development Only)**

Create a `.env` file to override specific Parameter Store values during development:

```bash
# .env file (optional) - only overrides specific parameters
AWS_REGION=eu-west-1
SERVER_PORT=8082
SERVICE_NAME=mb-api-chatbot-mc

# All other parameters (database tables, models, etc.) still load from Parameter Store
```

**Important**: Only parameters listed in `.env` override Parameter Store. All unlisted parameters continue loading from Parameter Store automatically.

## Running the Application

### **Development Authentication (Important Note)**

**In development mode, JWT authentication is completely bypassed:**

- **No NGINX OIDC**: Direct frontend → FastAPI communication
- **Hardcoded Client ID**: Uses `'clientid'` from frontend interceptor
- **No Token Validation**: FastAPI accepts any client ID header
- **Simplified Flow**: Perfect for development and testing

This means you can develop and test without dealing with OAuth/JWT complexity.

### **Development Mode (Recommended)**

### **Complete Environment Setup**

```bash
# Required: JFrog repository access
export UV_INDEX_PRIVATE_REGISTRY_USERNAME=x756900@opendigitalservices.com
export UV_INDEX_PRIVATE_REGISTRY_PASSWORD=<your-jfrog-token>

# Install dependencies
uv sync

# Start server (configuration loads automatically from AWS Parameter Store)
uv run src/ia_mb_api_chatbot/run.py
```

**Access Points:**

- **Frontend UI**: `http://localhost:8082/gui` (integrated NiceGUI interface)
- **API Documentation**: `http://localhost:8082/docs` (Swagger UI)
- **API Testing**: `http://localhost:8082/redoc` (ReDoc format)
- **Health Check**: `http://localhost:8082/health`

**Configuration Notes:**

- **Parameter Store**: Configuration loads automatically - no manual setup required
- **JFrog Token**: Obtain from [JFrog Portal](https://gluoneurope.jfrog.io/ui/login) - expires every 3 months
- **AWS Credentials**: Required for Parameter Store and service access

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
├── src/ia_mb_api_chatbot/ # Python FastAPI backend
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
│   ├── frontend/          # Integrated NiceGUI frontend
│   ├── main.py            # FastAPI application setup
│   └── run.py             # Application entry point
├── config/                # Environment configurations
│   ├── dev.yaml           # Development settings
│   └── qa.yaml            # QA environment settings
├── tests/                 # Unit and integration tests
└── docs/                  # Technical documentation
```

### **Development Commands**

### **Backend Development**

```bash
# Run backend server (correct path)
uv run src/ia_mb_api_chatbot/run.py

# Alternative: Run specific module
uv run python -m src.ia_mb_api_chatbot.run

# Run tests
uv run pytest tests/

# Check code formatting
uv run black src/
uv run flake8 src/

# Type checking
uv run mypy src/
```

### **Dependency Management**

### **Python Dependencies (UV)**

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

## Testing

### **Backend Testing**

```bash
# Run all tests
uv run pytest tests/

# Run specific test file
uv run pytest tests/services/test_conversation.py

# Run with coverage
uv run pytest tests/ --cov=src

# Run integration tests
uv run pytest tests/ -m integration
```

## Debugging

### **Backend Debugging**

```bash
# Enable debug logging
export LOG_LEVEL=DEBUG

# Run with Python debugger
uv run python -m pdb src/ia_mb_api_chatbot/run.py

# Check AWS connectivity
aws dynamodb describe-table --table-name chatbot-interactions-dev
aws bedrock list-foundation-models --region eu-west-1
```

### **Common Issues**

### **AWS Connection Issues**

```bash
# Verify AWS credentials
aws sts get-caller-identity

# Check IAM permissions
aws iam get-user

# Test DynamoDB access
aws dynamodb list-tables
```

### **UV Package Issues**

```bash
# Clear UV cache
uv cache clean

# Reinstall dependencies
uv sync --force

# Check for conflicts
uv tree --show-conflicts
```

## Advanced Development

### **Hot Reloading**

The backend supports hot reloading during development:

- **Backend**: FastAPI automatically reloads on code changes
- **Frontend**: Integrated NiceGUI interface updates automatically

### **API Development**

```bash
# Interactive API documentation  
http://localhost:8082/docs          # Swagger UI
http://localhost:8082/redoc         # ReDoc

# Test API endpoints
curl -X POST http://localhost:8082/sessions \
  -H "Content-Type: application/json" \
  -H "x-santander-client-id: test-client" \
  -d '{"analytics": {"browser": "Chrome", "device": "Desktop", "pageUrl": "http://localhost:8082/gui/", "channel": "web"}}'
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

## Troubleshooting

### **Common Problems**

1. **Port Already in Use**

   ```bash
   # Backend (port 8082)
   lsof -ti:8082 | xargs kill
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

This development guide provides everything needed to set up and work with the IA MB API Chatbot locally using UV package management and the integrated frontend interface.

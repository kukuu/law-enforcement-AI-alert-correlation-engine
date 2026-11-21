# RUNBOOK: COMPLETE RUNNING INSTRUCTIONS

Operations manual for running, debugging, and recovering the platform.

## Prerequisites

```
# Install Required Software
# 1. Node.js 18+ and npm
# 2. Java 17+ and Maven
# 3. Docker and Docker Compose
# 4. Git

# Verify installations
node --version      # Should be 18+
npm --version       # Should be 9+
java --version      # Should be 17+
mvn --version       # Should be 3.6+
docker --version    # Should be 20+
docker-compose --version
git --version
```

```
 java --
version
openjdk 21.0.9 2025-10-21 LTS
OpenJDK Runtime Environment Zulu21.46+19-CA (build 21.0.9+10-LTS)
OpenJDK 64-Bit Server VM Zulu21.46+19-CA (build 21.0.9+10-LTS, mixed mode, sharing)
```

path to JAVA

```
 /usr/li
bexec/java_home
/Library/Java/JavaVirtualMachines/zulu-21.jdk/Contents/Home

....
1. 
export PATH=/path/to/maven/bin:$PATH

2. 📌 To avoid repeating export every time, add it permanently to your shell config - where POM file is:

echo 'export PATH=/path/to/maven/bin:$PATH' >> ~/.zshrc source ~/.zshrc

3. Run at where POM file is

mvn clean compile

4. Frontend

npm run start

```


Maven version

```
-3.9.6-bin.tar.gz
-rw-r--r--  1 alexanderadu-sarkodie  wheel   9.0M Jan 10 22:22 apache-maven-3.9.6-bin.tar.gz

----

Apache Maven 3.9.6 (bc0240f3c744dd6b6ec2920b3cd08dcc295161ae)
Maven home: /usr/local/maven
Java version: 21.0.9, vendor: Azul Systems, Inc., runtime: /Library/Java/JavaVirtualMachines/zulu-21.jdk/Contents/Home
Default locale: en_GB, platform encoding: UTF-8
OS name: "mac os x", version: "12.7.4", arch: "x86_64", family: "mac"



```

Step 1: Clone and Setup Project

```
# Clone the repository
git clone https://github.com/your-org/le-alert-correlation-system.git
cd le-alert-correlation-system

# Create environment files
cp frontend/.env.example frontend/.env
cp backend/.env.example backend/.env

# Generate security keys
openssl genrsa -out backend/security-keys/jwt-private.key 2048
openssl rsa -in backend/security-keys/jwt-private.key -pubout -out backend/security-keys/jwt-public.key

# Generate SSL certificates for development
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout backend/security-keys/ssl.key \
  -out backend/security-keys/ssl.crt \
  -subj "/C=US/ST=State/L=City/O=LE/CN=localhost"
```

Step 2: Configure Environment Variables

```
# Edit frontend/.env
nano frontend/.env

# Add these values:
VITE_API_URL=http://localhost:8080
VITE_WS_URL=ws://localhost:8080/ws
VITE_APP_VERSION=1.0.0
VITE_APP_TITLE="LE Alert Correlation System"
VITE_ENCRYPTION_KEY=your-secure-encryption-key-32-chars
VITE_SENTRY_DSN=  # Optional for error tracking

# Edit backend/.env
nano backend/.env

# Add these values:
JWT_SECRET=your-jwt-secret-key-256-bit
DB_PASSWORD=secure-postgres-password
MONGO_PASSWORD=secure-mongo-password
REDIS_PASSWORD=secure-redis-password
KAFKA_PASSWORD=secure-kafka-password
VAULT_TOKEN=development-token
ENCRYPTION_KEY=your-backend-encryption-key
```

Step 3: Start Infrastructure with Docker

```

# Navigate to project root
cd le-alert-correlation-system

# Start all infrastructure services
docker-compose up -d

# Verify services are running
docker-compose ps

# Expected output:
# Name                State   Ports
# ---------------------------------------
# mongodb             Up      27017/tcp
# postgres            Up      5432/tcp
# redis               Up      6379/tcp
# zookeeper           Up      2181/tcp
# kafka               Up      9092/tcp, 9093/tcp
# schema-registry     Up      8081/tcp
# prometheus          Up      9090/tcp
# grafana             Up      3001/tcp
# jaeger              Up      16686/tcp, 14250/tcp, 14268/tcp

# Initialize databases
docker-compose exec postgres psql -U le_admin -d le_system -f /docker-entrypoint-initdb.d/init.sql
docker-compose exec mongodb mongosh /docker-entrypoint-initdb.d/init.js

# Create Kafka topics
./kafka-setup/topics-create.sh

# Access monitoring dashboards:
# Prometheus: http://localhost:9090
# Grafana: http://localhost:3001 (admin/admin)
# Jaeger: http://localhost:16686

```

Step 4: Build and Run Backend Services

```
# Navigate to backend directory
cd backend

# Build all Java services
mvn clean install -DskipTests

# Or build individual services
mvn clean install -pl common-lib -am
mvn clean install -pl api-gateway -am
mvn clean install -pl alert-service -am
mvn clean install -pl correlation-engine -am
mvn clean install -pl auth-service -am

# Option 1: Run with Spring Boot Maven plugin
# Run API Gateway
cd api-gateway
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Open new terminal, run Alert Service
cd backend/alert-service
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Open new terminal, run Auth Service
cd backend/auth-service
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Open new terminal, run Correlation Engine
cd backend/correlation-engine
mvn quarkus:dev

# Option 2: Run with Docker
# Build Docker images
docker build -t le-api-gateway:latest -f api-gateway/Dockerfile .
docker build -t le-alert-service:latest -f alert-service/Dockerfile .
docker build -t le-correlation-engine:latest -f correlation-engine/Dockerfile .
docker build -t le-auth-service:latest -f auth-service/Dockerfile .

# Run with Docker Compose
cd ..
docker-compose -f docker-compose-backend.yml up -d

# Verify backend services
curl http://localhost:8080/actuator/health
# Expected: {"status":"UP"}

# Test authentication
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'

# Expected: JWT token response

```

Step 5: Build and Run React Frontend

```
# Navigate to frontend directory
cd frontend

# Install dependencies
npm install

# Install specific versions if needed
npm install react@18.2.0 react-dom@18.2.0
npm install @tanstack/react-query@4.32.6
npm install leaflet@1.9.4 react-leaflet@4.2.1

# Option 1: Development mode with hot reload
npm run dev

# Access at: http://localhost:3000

# Option 2: Production build and serve
npm run build
npm run preview

# Access at: http://localhost:4173

# Option 3: Docker for frontend
docker build -t le-alerts-ui:latest -f Dockerfile.prod .
docker run -p 3000:80 le-alerts-ui:latest

# Access at: http://localhost:3000
```

Step 6: Test the Complete System

```
# Test API endpoints
# 1. Health check
curl http://localhost:8080/actuator/health

# 2. Get alerts (requires authentication)
curl -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  http://localhost:8080/api/v1/alerts

# 3. WebSocket connection test
# Open browser console at http://localhost:3000
# Check WebSocket connection in Network tab

# 4. Simulate data ingestion
# Send test alert to Kafka
./scripts/send-test-alert.sh

# Or use HTTP endpoint
curl -X POST http://localhost:8080/api/v1/alerts/test \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "TEST",
    "category": "TEST_ALERT",
    "priority": "HIGH",
    "location": {"lat": 40.7128, "lng": -74.0060}
  }'

# 5. Check correlation engine output
# View logs: docker-compose logs correlation-engine
# Or check MongoDB: 
docker-compose exec mongodb mongosh le_alerts --eval "db.incidents.find().pretty()"

```

Step 7: Development Workflow

```
# 1. Start all services in development mode
# Terminal 1: Infrastructure
docker-compose up -d

# Terminal 2: Backend services
cd backend
./scripts/start-dev.sh

# Terminal 3: Frontend
cd frontend
npm run dev

# 2. Running tests
# Frontend tests
cd frontend
npm run test            # Unit tests
npm run test:coverage   # Coverage report
npm run test:e2e       # End-to-end tests

# Backend tests
cd backend
mvn test               # All tests
mvn test -pl alert-service  # Specific service

# 3. Code quality checks
cd frontend
npm run lint
npm run type-check
npm run format

cd backend
mvn spotless:check    # Code formatting
mvn pmd:check         # Static analysis
mvn spotbugs:check    # Bug patterns

# 4. Security scanning
npm run security:scan
mvn dependency-check:check

# 5. Build for production
cd frontend
npm run build:prod

cd backend
mvn clean package -Pprod
```

Step 8: Deployment Commands

```
# Deploy to Kubernetes (if configured)
./scripts/deploy-k8s.sh

# Deploy to AWS
./scripts/deploy-aws.sh --env production

# Deploy with Terraform
cd infrastructure/terraform
terraform init
terraform plan
terraform apply -auto-approve

# Backup and restore
./scripts/backup.sh
./scripts/restore.sh backup-2024-01-15.tar.gz

```
## Troubleshooting Guide

```
# 1. Port conflicts
lsof -i :3000  # Check what's using port 3000
kill -9 <PID>  # Kill process

# 2. Database connection issues
docker-compose logs postgres
docker-compose exec postgres pg_isready

# 3. Kafka issues
docker-compose logs kafka
docker-compose exec kafka kafka-topics --list --bootstrap-server localhost:9092

# 4. React build issues
rm -rf frontend/node_modules frontend/package-lock.json
npm cache clean --force
npm install

# 5. Java build issues
mvn clean
rm -rf ~/.m2/repository/com/le
mvn install -U

# 6. Memory issues (increase heap)
export MAVEN_OPTS="-Xmx2G -XX:MaxPermSize=512M"
export NODE_OPTIONS="--max-old-space-size=4096"

# 7. Reset everything
docker-compose down -v
rm -rf backend/target frontend/dist

```

## Useful Development URLs

```
Local Development:
- React App: http://localhost:3000
- API Gateway: http://localhost:8080
- API Documentation: http://localhost:8080/swagger-ui.html
- GraphQL Playground: http://localhost:8080/graphiql
- Eureka Dashboard: http://localhost:8761
- Config Server: http://localhost:8888
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3001 (admin/admin)
- Jaeger Tracing: http://localhost:16686
- Kafka UI: http://localhost:8082 (if Kafdrop running)
- MongoDB Express: http://localhost:8083
- PGAdmin: http://localhost:5050 (admin@admin.com/admin)

Production URLs (example):
- Application: https://alerts.le.example.com
- API: https://api.le.example.com
- Monitoring: https://monitor.le.example.com
- Documentation: https://docs.le.example.com

```

## Security Setup Commands


```
# Generate SSL certificates for production
openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt \
  -subj "/C=US/ST=State/L=City/O=LE/CN=alerts.le.example.com"

# Set up Vault for secrets management
docker run -d --name vault -p 8200:8200 vault:latest
export VAULT_ADDR='http://localhost:8200'
vault operator init
vault login
vault secrets enable -path=secret kv-v2

# Store secrets
vault kv put secret/le-alerts/production \
  jwt_secret="your-jwt-secret" \
  db_password="secure-password" \
  encryption_key="your-encryption-key"

# Configure HashiCorp Vault Agent
# See infrastructure/vault/ for configuration
```

## Performance Testing

```
# Load test with k6
k6 run scripts/load-test.js

# Stress test API
artillery run scripts/stress-test.yml

# Monitor performance
docker-compose logs -f --tail=100 alert-service
```

## React Frontend Startup Methods

Method 1: Standard React Development Server

```
# In the frontend directory, you have multiple options:

# Option A: Using Vite (Modern, Fast) - RECOMMENDED
cd frontend
npm run dev
# Access at: http://localhost:3000

# Option B: If using Create React App (Traditional)
cd frontend
npm start
# Access at: http://localhost:3000

# Option C: Production preview
cd frontend
npm run build
npm run preview
# Access at: http://localhost:4173
```

### Complete Step-by-Step Running Instructions

Step 1: Navigate to Correct Directory

```
# Assuming you're in the project root
pwd
# Should show: /path/to/le-alert-correlation-system

# Navigate to frontend
cd frontend

# List files to verify
ls -la
# Should show: package.json, src/, public/, vite.config.ts, etc.
```

Step 2: Install Dependencies (First Time Only)

```
# Install all dependencies
npm install

# Or if you want specific versions
npm ci  # Clean install from package-lock.json
```

Step 3: Configure Environment

```
# Copy example environment file
cp .env.example .env

# Edit the .env file with your settings
nano .env

# Add these minimum configurations:
VITE_API_URL=http://localhost:8080
VITE_WS_URL=ws://localhost:8080
VITE_APP_TITLE="LE Alert System"
```

Step 4: Start React Frontend

```
# Start the React development server
npm run dev

# OR (since we defined "start" script)
npm start

# Expected output:
# ➜  Local:   http://localhost:3000/
# ➜  Network: http://192.168.1.100:3000/
# ➜  press h to show help

# Open browser to: http://localhost:3000
```

- Option B: With Backend Services

```
# Terminal 1: Start infrastructure (Kafka, MongoDB, etc.)
cd ..  # Go to project root
docker-compose up -d

# Terminal 2: Start Java backend services
cd backend
./scripts/start-backend.sh

# Terminal 3: Start React frontend
cd ../frontend
npm start  # or npm run dev

# Open browser to: http://localhost:3000

```

- Option C: Using the start-all.sh Script

```

# Make the script executable
chmod +x start-all.sh

# Run the complete system
./start-all.sh

# This script will:
# 1. Start Docker infrastructure
# 2. Build and start Java backend
# 3. Install frontend dependencies
# 4. Start React frontend with npm run dev

```

Step 5: Verify Frontend is Running

```

# Check if React is running
curl http://localhost:3000

# Check running processes
ps aux | grep -i "node.*react"

# Check logs in the terminal where you ran npm start
# You should see:
# ✓ Built in 1.23s
# ➜  Local:   http://localhost:3000/

```

## Common Issues and Solutions

- Issue 1: Port 3000 already in use

```

# Find what's using port 3000
lsof -i :3000

# Kill the process
kill -9 <PID>

# Or use a different port
# Edit package.json scripts:
# "start": "vite --port 3001"

```

- Issue 2: Node modules missing/corrupt

```
# Clean install
rm -rf node_modules package-lock.json
npm cache clean --force
npm install
```

- Issue 3: TypeScript errors

```
# Check TypeScript compilation
npx tsc --noEmit

# Or run the type check script
npm run type-check
```

## Development Workflow Example

### Running the system:
```
# Open 3 terminal windows

# TERMINAL 1: Infrastructure
cd le-alert-correlation-system
docker-compose up -d

# TERMINAL 2: Backend services
cd le-alert-correlation-system/backend
./scripts/start-dev.sh
# Wait for: "Started AlertApplication in X seconds"

# TERMINAL 3: React frontend
cd le-alert-correlation-system/frontend
npm start
# Wait for: "Local: http://localhost:3000"

# Now open browser to http://localhost:3000
# Login with: admin / admin123

```

### Production Deployment

```
# Build the React app
cd frontend
npm run build:prod

# The build output will be in: frontend/dist/

# To serve the built files locally:
npm install -g serve
serve -s dist -p 3000

# Or use the preview command:
npm run preview
# Access at: http://localhost:4173

```

### React Development Tips

```
# Hot Module Replacement is enabled by default
# Changes to your code will auto-refresh in browser

# Open React DevTools in browser (F12 -> Components/Profiler)

# Environment variables in React code:
console.log(import.meta.env.VITE_API_URL)

# To add new dependencies:
npm install <package-name> --save

# To update dependencies:
npm update

```

### Testing the Frontend

```
# Run unit tests
npm test

# Run tests in watch mode
npm run test:watch

# Run end-to-end tests (requires backend running)
npm run test:e2e

# Generate test coverage report
npm run test:coverage
# Open: frontend/coverage/index.html
```

### Debugging React

```
# Add debugger statements in code
debugger;

# Use React DevTools browser extension

# Check console for errors

# Use network tab to see API requests

# To debug build issues:
npm run build -- --debug

# To analyze bundle size:
npm run build -- --report
```

### Quick Reference Card

```

# FRONTEND COMMANDS (run in frontend/ directory)
npm install          # Install dependencies
npm start           # Start development server
npm run dev         # Same as npm start
npm run build       # Build for production
npm run preview     # Preview production build
npm test           # Run tests
npm run lint       # Check code quality

# BACKEND COMMANDS (run in backend/ directory)
mvn clean install  # Build all services
mvn spring-boot:run # Run Spring Boot app
./scripts/start-dev.sh # Start all backend services

# INFRASTRUCTURE (run in project root)
docker-compose up -d  # Start databases, Kafka, etc.
docker-compose down   # Stop infrastructure

# COMPLETE SYSTEM
./start-all.sh        # Start everything (frontend + backend + infra)

```

### Vite Configuration for React (vite.config.ts)

```

import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    host: true,
    open: true,  // Auto-open browser
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
      '/ws': {
        target: 'ws://localhost:8080',
        ws: true,
      },
    },
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
  },
});

```

## JAVA Compilation

- compilation command 

```

mvn clean compile

```

-Terminal response

```
JAVA/real-time-alert-correlation-engine/backend/correlation-engine/src/main/resources
[INFO] 
[INFO] --- quarkus:2.16.12.Final:generate-code (default) @ correlation-engine ---
[INFO] 
[INFO] --- compiler:3.11.0:compile (default-compile) @ correlation-engine ---
[INFO] Changes detected - recompiling the module! :dependency
[INFO] Compiling 10 source files with javac [debug target 17] to target/classes
[WARNING] system modules path not set in conjunction with -source 17
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary for Real-Time Alert Correlation Backend 1.0.0-SNAPSHOT:
[INFO] 
[INFO] Real-Time Alert Correlation Backend ................ SUCCESS [  0.129 s]
[INFO] Common Utilities ................................... SUCCESS [  1.095 s]
[INFO] Correlation Engine ................................. SUCCESS [  4.178 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  6.902 s
[INFO] Finished at: 2026-01-12T16:46:34Z
[INFO] ------------------------------------------------------------------------
(base) Alexanders-MacBook-Pro:backend alexan
de
radu-sarkodie$ 
```


## Complete React Frontend Build & Execution Commands
Project Structure:
text
real-time-alert-correlation-engine/
├── frontend/                    # React/Vite frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── tailwind.config.js
│   ├── postcss.config.js
│   ├── public/
│   └── src/
└── backend/                    # Backend modules

### PART 1: FRONTEND BUILD & EXECUTION COMMANDS

1. Install Dependencies (First Time)

#### Navigate to frontend directory

```
cd /path/to/real-time-alert-correlation-engine/frontend
```

#### Install all dependencies from package.json
```
npm install


# OR with yarn (if you prefer)
# yarn install

```

#### To install specific global tools (optional)
```
npm install -g concurrently   # For running multiple commands
npm install -g serve          # For serving production build
```

2. Development Mode (Hot Reload)

```
# Start development server (default port: 5173)
cd /path/to/real-time-alert-correlation-engine/frontend
npm run dev

# OR with custom port
npm run dev -- --port 3000

# Output will show:
# VITE v4.x.x  ready in xxx ms
# ➜  Local:   http://localhost:5173/
# ➜  Network: http://192.168.x.x:5173/

```
3. Production Build

####  Create optimized production build
```
cd /path/to/real-time-alert-correlation-engine/frontend
npm run build

# Output will be in:
# /path/to/real-time-alert-correlation-engine/frontend/dist/
```

4. Preview Production Build


#### Preview the production build locally
```
cd /path/to/real-time-alert-correlation-engine/frontend
npm run preview

# OR serve with a static server
npx serve dist/
# OR
npm install -g serve
serve -s dist -p 4173

```

5. Testing

#### Run tests (if configured)
```
cd /path/to/real-time-alert-correlation-engine/frontend
npm run test

# Run tests in watch mode
npm run test:watch

# Run tests with coverage
npm run test:coverage

```

6. Linting & Code Quality



#### Run ESLint (if configured)

```
npm run lint

# Fix linting issues automatically
npm run lint:fix

# Type checking (if using TypeScript)
npm run type-check
# OR
npx tsc --noEmit

```

PART 2: COMBINED BACKEND + FRONTEND EXECUTION

7. Run Both Services (Separate Terminals)
Terminal 1 - Backend:

bash
cd /path/to/real-time-alert-correlation-engine/backend/correlation-engine
mvn quarkus:dev
# Backend runs on: http://localhost:8080
Terminal 2 - Frontend:

bash
cd /path/to/real-time-alert-correlation-engine/frontend
npm run dev
# Frontend runs on: http://localhost:5173
8. Run Both Services with Single Command (Using Concurrently)
Create a script at root: /path/to/real-time-alert-correlation-engine/run-dev.sh

bash
#!/bin/bash
echo "Starting Real-Time Alert Correlation Engine..."

echo "=================================================="
echo "1. Checking prerequisites..."
echo "=================================================="

# Check if Node.js is installed
if ! command -v node &> /dev/null; then
    echo "ERROR: Node.js is not installed. Please install Node.js 16+"
    exit 1
fi

# Check if npm is installed
if ! command -v npm &> /dev/null; then
    echo "ERROR: npm is not installed. Please install npm"
    exit 1
fi

# Check if Maven is installed
if ! command -v mvn &> /dev/null; then
    echo "ERROR: Maven is not installed. Please install Maven 3.6+"
    exit 1
fi

echo "✓ Node.js: $(node --version)"
echo "✓ npm: $(npm --version)"
echo "✓ Maven: $(mvn --version | grep 'Apache Maven' | head -1)"

echo ""
echo "=================================================="
echo "2. Installing frontend dependencies..."
echo "=================================================="
cd frontend
if [ ! -d "node_modules" ]; then
    echo "Installing npm packages..."
    npm install
    if [ $? -ne 0 ]; then
        echo "ERROR: npm install failed"
        exit 1
    fi
else
    echo "✓ node_modules already exists"
fi

echo ""
echo "=================================================="
echo "3. Starting services..."
echo "=================================================="

# Start backend in background
echo "Starting Backend (Quarkus) on http://localhost:8080..."
cd ../backend/correlation-engine
mvn quarkus:dev > backend.log 2>&1 &
BACKEND_PID=$!
echo "Backend PID: $BACKEND_PID"

# Wait for backend to start
sleep 10

# Start frontend in background
echo "Starting Frontend (Vite) on http://localhost:5173..."
cd ../../frontend
npm run dev > frontend.log 2>&1 &
FRONTEND_PID=$!
echo "Frontend PID: $FRONTEND_PID"

echo ""
echo "=================================================="
echo "Services are starting..."
echo "=================================================="
echo "Backend:  http://localhost:8080"
echo "Frontend: http://localhost:5173"
echo "Logs:"
echo "  - Backend:  backend/correlation-engine/backend.log"
echo "  - Frontend: frontend/frontend.log"
echo ""
echo "Press Ctrl+C to stop all services"
echo "=================================================="

# Function to cleanup on exit
cleanup() {
    echo ""
    echo "=================================================="
    echo "Shutting down services..."
    echo "=================================================="
    
    echo "Stopping frontend (PID: $FRONTEND_PID)..."
    kill $FRONTEND_PID 2>/dev/null || true
    
    echo "Stopping backend (PID: $BACKEND_PID)..."
    kill $BACKEND_PID 2>/dev/null || true
    
    echo "Cleaning up..."
    wait $FRONTEND_PID $BACKEND_PID 2>/dev/null
    
    echo "All services stopped."
    exit 0
}

# Trap Ctrl+C
trap cleanup INT

# Keep script running
wait
Make it executable and run:

bash
cd /path/to/real-time-alert-correlation-engine
chmod +x run-dev.sh
./run-dev.sh
9. Using npm-run-all (Alternative)
Update frontend/package.json:

json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "dev:full": "npm-run-all --parallel dev backend",
    "backend": "cd ../backend/correlation-engine && mvn quarkus:dev"
  },
  "devDependencies": {
    "npm-run-all": "^4.1.5"
  }
}

Then run:

bash
cd /path/to/real-time-alert-correlation-engine/frontend
npm install npm-run-all
npm run dev:full
PART 3: PRODUCTION DEPLOYMENT SCRIPTS
10. Full Production Build Script
Create: /path/to/real-time-alert-correlation-engine/build-production.sh

bash
#!/bin/bash
echo "Building Real-Time Alert Correlation Engine for production..."
echo "=================================================="

# Set error handling
set -e

echo "1. Building backend..."
cd backend
echo "   Cleaning..."
mvn clean
echo "   Packaging..."
mvn package -DskipTests
echo "   ✓ Backend built: correlation-engine/target/*.jar"

echo ""
echo "2. Building frontend..."
cd ../frontend
echo "   Installing dependencies..."
npm ci --silent
echo "   Building..."
npm run build
echo "   ✓ Frontend built: frontend/dist/"

echo ""
echo "3. Creating deployment package..."
cd ..
DEPLOY_DIR="deployment-$(date +%Y%m%d-%H%M%S)"
mkdir -p $DEPLOY_DIR

# Copy backend JAR
cp backend/correlation-engine/target/*.jar $DEPLOY_DIR/correlation-engine.jar

# Copy frontend build
cp -r frontend/dist $DEPLOY_DIR/frontend

# Copy configuration files
cp docker-compose.yml $DEPLOY_DIR/ 2>/dev/null || true
cp .env $DEPLOY_DIR/ 2>/dev/null || true
cp RUNBOOK.md $DEPLOY_DIR/ 2>/dev/null || true

# Create startup script
cat > $DEPLOY_DIR/start.sh << 'EOF'
#!/bin/bash
echo "Starting Real-Time Alert Correlation Engine..."

# Start backend
java -jar correlation-engine.jar &
BACKEND_PID=$!

# Start frontend (if you want to serve from same machine)
# You might want to use nginx in production
cd frontend
npx serve -s . -p 4173 &
FRONTEND_PID=$!

echo "Backend PID: $BACKEND_PID"
echo "Frontend PID: $FRONTEND_PID"
echo "Backend: http://localhost:8080"
echo "Frontend: http://localhost:4173"

trap "kill $BACKEND_PID $FRONTEND_PID; exit" INT
wait
EOF

chmod +x $DEPLOY_DIR/start.sh

echo ""
echo "=================================================="
echo "Production build complete!"
echo "Deployment package: $DEPLOY_DIR/"
echo "Contents:"
echo "  - correlation-engine.jar (Backend)"
echo "  - frontend/ (Frontend build)"
echo "  - start.sh (Startup script)"
echo "=================================================="
11. Docker-Based Execution
Create: /path/to/real-time-alert-correlation-engine/docker-compose.yml

yaml
version: '3.8'

services:
  # Frontend service
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:80"
    environment:
      - REACT_APP_API_URL=http://backend:8080
    depends_on:
      - backend
    networks:
      - alert-network

  # Backend service
  backend:
    build:
      context: ./backend/correlation-engine
      dockerfile: Dockerfile.jvm
    ports:
      - "8080:8080"
    environment:
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
      - QUARKUS_PROFILE=prod
    depends_on:
      - kafka
    networks:
      - alert-network

  # Kafka for message streaming
  kafka:
    image: confluentinc/cp-kafka:latest
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    depends_on:
      - zookeeper
    networks:
      - alert-network

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - alert-network

networks:
  alert-network:
    driver: bridge
Create frontend Dockerfile: /path/to/real-time-alert-correlation-engine/frontend/Dockerfile

dockerfile

# Build stage
FROM node:18-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm ci --silent
COPY . .
RUN npm run build

# Production stage

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
Run with Docker:

bash
# Build and start all services
cd /path/to/real-time-alert-correlation-engine
docker-compose up --build

# Stop services
docker-compose down

# View logs
docker-compose logs -f

# Rebuild specific service
docker-compose up --build frontend
PART 4: TROUBLESHOOTING COMMANDS
12. Diagnostic Commands
bash
# Check frontend dependencies
cd /path/to/real-time-alert-correlation-engine/frontend
npm list --depth=0

# Check for outdated packages
npm outdated

# Clear npm cache
npm cache clean --force

# Remove node_modules and reinstall
rm -rf node_modules package-lock.json
npm install

# Check port usage
sudo lsof -i :5173  # Frontend default port
sudo lsof -i :8080  # Backend default port

# Kill process on specific port
sudo kill -9 $(sudo lsof -t -i:5173)

# Check environment variables
echo $NODE_ENV
echo $REACT_APP_API_URL
13. Environment Configuration
Create: /path/to/real-time-alert-correlation-engine/frontend/.env

env
# Frontend environment variables
VITE_API_URL=http://localhost:8080
VITE_WS_URL=ws://localhost:8080/ws
VITE_KAFKA_BROKERS=localhost:9092
VITE_APP_TITLE="Real-Time Alert Correlation"
Update vite.config.ts to use these:

typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      }
    }
  }
})

SUMMARY OF KEY COMMANDS:

bash
# Development workflow:
cd /path/to/real-time-alert-correlation-engine
./run-dev.sh

# OR manually:
# Terminal 1:
cd backend/correlation-engine && mvn quarkus:dev

# Terminal 2:
cd frontend && npm run dev

# Production build:
cd /path/to/real-time-alert-correlation-engine
./build-production.sh

# Docker deployment:
docker-compose up --build

# Testing:
cd frontend && npm run test



## Complete React Frontend Build & Execution Commands
Project Structure:
text
real-time-alert-correlation-engine/
├── frontend/                    # React/Vite frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── tailwind.config.js
│   ├── postcss.config.js
│   ├── public/
│   └── src/
└── backend/                    # Backend modules

PART 1: FRONTEND BUILD & EXECUTION COMMANDS

1. Install Dependencies (First Time)
bash
# Navigate to frontend directory
cd /path/to/real-time-alert-correlation-engine/frontend

# Install all dependencies from package.json
npm install
# OR with yarn (if you prefer)
# yarn install

# To install specific global tools (optional)
npm install -g concurrently   # For running multiple commands
npm install -g serve          # For serving production build

2. Development Mode (Hot Reload)
bash
# Start development server (default port: 5173)
cd /path/to/real-time-alert-correlation-engine/frontend
npm run dev

# OR with custom port
npm run dev -- --port 3000

# Output will show:
# VITE v4.x.x  ready in xxx ms
# ➜  Local:   http://localhost:5173/
# ➜  Network: http://192.168.x.x:5173/

3. Production Build
bash
# Create optimized production build
cd /path/to/real-time-alert-correlation-engine/frontend
npm run build

# Output will be in:
# /path/to/real-time-alert-correlation-engine/frontend/dist/

4. Preview Production Build
bash
# Preview the production build locally
cd /path/to/real-time-alert-correlation-engine/frontend
npm run preview

# OR serve with a static server
npx serve dist/
# OR
npm install -g serve
serve -s dist -p 4173
5. Testing
bash
# Run tests (if configured)
cd /path/to/real-time-alert-correlation-engine/frontend
npm run test

# Run tests in watch mode
npm run test:watch

# Run tests with coverage
npm run test:coverage

6. Linting & Code Quality
bash
# Run ESLint (if configured)
npm run lint

# Fix linting issues automatically
npm run lint:fix

# Type checking (if using TypeScript)
npm run type-check
# OR
npx tsc --noEmit

PART 2: COMBINED BACKEND + FRONTEND EXECUTION

7. Run Both Services (Separate Terminals)

Terminal 1 - Backend:

```
cd /path/to/real-time-alert-correlation-engine/backend/correlation-engine

mvn quarkus:dev

# Backend runs on: http://localhost:8080

```

Terminal 2 - Frontend:

```
cd /path/to/real-time-alert-correlation-engine/frontend

npm run dev

# Frontend runs on: http://localhost:5173

```
8. Start-up Script: Run Both Services with Single Command (Using Concurrently)

Create a script at root: /path/to/real-time-alert-correlation-engine/run-dev.sh

```
#!/bin/bash
echo "Starting Real-Time Alert Correlation Engine..."

echo "=================================================="
echo "1. Checking prerequisites..."
echo "=================================================="

# Check if Node.js is installed
if ! command -v node &> /dev/null; then
    echo "ERROR: Node.js is not installed. Please install Node.js 16+"
    exit 1
fi

# Check if npm is installed
if ! command -v npm &> /dev/null; then
    echo "ERROR: npm is not installed. Please install npm"
    exit 1
fi

# Check if Maven is installed
if ! command -v mvn &> /dev/null; then
    echo "ERROR: Maven is not installed. Please install Maven 3.6+"
    exit 1
fi

echo "✓ Node.js: $(node --version)"
echo "✓ npm: $(npm --version)"
echo "✓ Maven: $(mvn --version | grep 'Apache Maven' | head -1)"

echo ""
echo "=================================================="
echo "2. Installing frontend dependencies..."
echo "=================================================="
cd frontend
if [ ! -d "node_modules" ]; then
    echo "Installing npm packages..."
    npm install
    if [ $? -ne 0 ]; then
        echo "ERROR: npm install failed"
        exit 1
    fi
else
    echo "✓ node_modules already exists"
fi

echo ""
echo "=================================================="
echo "3. Starting services..."
echo "=================================================="

# Start backend in background

echo "Starting Backend (Quarkus) on http://localhost:8080..."
cd ../backend/correlation-engine
mvn quarkus:dev > backend.log 2>&1 &
BACKEND_PID=$!
echo "Backend PID: $BACKEND_PID"

# Wait for backend to start
sleep 10

# Start frontend in background
echo "Starting Frontend (Vite) on http://localhost:5173..."
cd ../../frontend
npm run dev > frontend.log 2>&1 &
FRONTEND_PID=$!
echo "Frontend PID: $FRONTEND_PID"

echo ""
echo "=================================================="
echo "Services are starting..."
echo "=================================================="
echo "Backend:  http://localhost:8080"
echo "Frontend: http://localhost:5173"
echo "Logs:"
echo "  - Backend:  backend/correlation-engine/backend.log"
echo "  - Frontend: frontend/frontend.log"
echo ""
echo "Press Ctrl+C to stop all services"
echo "=================================================="

# Function to cleanup on exit
cleanup() {
    echo ""
    echo "=================================================="
    echo "Shutting down services..."
    echo "=================================================="
    
    echo "Stopping frontend (PID: $FRONTEND_PID)..."
    kill $FRONTEND_PID 2>/dev/null || true
    
    echo "Stopping backend (PID: $BACKEND_PID)..."
    kill $BACKEND_PID 2>/dev/null || true
    
    echo "Cleaning up..."
    wait $FRONTEND_PID $BACKEND_PID 2>/dev/null
    
    echo "All services stopped."
    exit 0
}

# Trap Ctrl+C
trap cleanup INT

# Keep script running
wait

```

Make it executable and run:

1. 
```
cd /path/to/real-time-alert-correlation-engine
chmod +x run-dev.sh
```

2. 
```
./run-dev.sh

```
9. Using npm-run-all (Alternative)

Update frontend/package.json:

json
```
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "dev:full": "npm-run-all --parallel dev backend",
    "backend": "cd ../backend/correlation-engine && mvn quarkus:dev"
  },
  "devDependencies": {
    "npm-run-all": "^4.1.5"
  }
}

```

Then run:

```
cd /path/to/real-time-alert-correlation-engine/frontend

npm install npm-run-all

npm run dev:full
```

PART 3: PRODUCTION DEPLOYMENT SCRIPTS

10. Full Production Build Script
Create: /path/to/real-time-alert-correlation-engine/build-production.sh

bash
#!/bin/bash
echo "Building Real-Time Alert Correlation Engine for production..."
echo "=================================================="

# Set error handling
set -e

echo "1. Building backend..."
cd backend
echo "   Cleaning..."
mvn clean
echo "   Packaging..."
mvn package -DskipTests
echo "   ✓ Backend built: correlation-engine/target/*.jar"

echo ""
echo "2. Building frontend..."
cd ../frontend
echo "   Installing dependencies..."
npm ci --silent
echo "   Building..."
npm run build
echo "   ✓ Frontend built: frontend/dist/"

echo ""
echo "3. Creating deployment package..."
cd ..
DEPLOY_DIR="deployment-$(date +%Y%m%d-%H%M%S)"
mkdir -p $DEPLOY_DIR

# Copy backend JAR
cp backend/correlation-engine/target/*.jar $DEPLOY_DIR/correlation-engine.jar

# Copy frontend build
cp -r frontend/dist $DEPLOY_DIR/frontend

# Copy configuration files
cp docker-compose.yml $DEPLOY_DIR/ 2>/dev/null || true
cp .env $DEPLOY_DIR/ 2>/dev/null || true
cp RUNBOOK.md $DEPLOY_DIR/ 2>/dev/null || true

# Create startup script
cat > $DEPLOY_DIR/start.sh << 'EOF'
#!/bin/bash
echo "Starting Real-Time Alert Correlation Engine..."

# Start backend
java -jar correlation-engine.jar &
BACKEND_PID=$!

# Start frontend (if you want to serve from same machine)
# You might want to use nginx in production
cd frontend
npx serve -s . -p 4173 &
FRONTEND_PID=$!

echo "Backend PID: $BACKEND_PID"
echo "Frontend PID: $FRONTEND_PID"
echo "Backend: http://localhost:8080"
echo "Frontend: http://localhost:4173"

trap "kill $BACKEND_PID $FRONTEND_PID; exit" INT
wait
EOF

chmod +x $DEPLOY_DIR/start.sh

echo ""
echo "=================================================="
echo "Production build complete!"
echo "Deployment package: $DEPLOY_DIR/"
echo "Contents:"
echo "  - correlation-engine.jar (Backend)"
echo "  - frontend/ (Frontend build)"
echo "  - start.sh (Startup script)"
echo "=================================================="
11. Docker-Based Execution
Create: /path/to/real-time-alert-correlation-engine/docker-compose.yml

yaml
version: '3.8'

services:
  # Frontend service
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:80"
    environment:
      - REACT_APP_API_URL=http://backend:8080
    depends_on:
      - backend
    networks:
      - alert-network

  # Backend service
  backend:
    build:
      context: ./backend/correlation-engine
      dockerfile: Dockerfile.jvm
    ports:
      - "8080:8080"
    environment:
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
      - QUARKUS_PROFILE=prod
    depends_on:
      - kafka
    networks:
      - alert-network

  # Kafka for message streaming
  kafka:
    image: confluentinc/cp-kafka:latest
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    depends_on:
      - zookeeper
    networks:
      - alert-network

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - alert-network

networks:
  alert-network:
    driver: bridge
Create frontend Dockerfile: /path/to/real-time-alert-correlation-engine/frontend/Dockerfile

dockerfile
# Build stage
FROM node:18-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm ci --silent
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
Run with Docker:

bash
# Build and start all services
cd /path/to/real-time-alert-correlation-engine
docker-compose up --build

# Stop services
docker-compose down

# View logs
docker-compose logs -f

# Rebuild specific service
docker-compose up --build frontend
PART 4: TROUBLESHOOTING COMMANDS
12. Diagnostic Commands
bash
# Check frontend dependencies
cd /path/to/real-time-alert-correlation-engine/frontend
npm list --depth=0

# Check for outdated packages
npm outdated

# Clear npm cache
npm cache clean --force

# Remove node_modules and reinstall
rm -rf node_modules package-lock.json
npm install

# Check port usage
sudo lsof -i :5173  # Frontend default port
sudo lsof -i :8080  # Backend default port

# Kill process on specific port
sudo kill -9 $(sudo lsof -t -i:5173)

# Check environment variables
echo $NODE_ENV
echo $REACT_APP_API_URL
13. Environment Configuration
Create: /path/to/real-time-alert-correlation-engine/frontend/.env

env
# Frontend environment variables
VITE_API_URL=http://localhost:8080
VITE_WS_URL=ws://localhost:8080/ws
VITE_KAFKA_BROKERS=localhost:9092
VITE_APP_TITLE="Real-Time Alert Correlation"
Update vite.config.ts to use these:

typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      }
    }
  }
})
SUMMARY OF KEY COMMANDS:
bash
# Development workflow:
cd /path/to/real-time-alert-correlation-engine
./run-dev.sh

# OR manually:
# Terminal 1:
cd backend/correlation-engine && mvn quarkus:dev

# Terminal 2:
cd frontend && npm run dev

# Production build:
cd /path/to/real-time-alert-correlation-engine
./build-production.sh

# Docker deployment:
docker-compose up --build

# Testing:
cd frontend && npm run test

## Adding Maven to path

Run this at where the root is  i.e where the pom.xml is in our case 
backend/correlation-engine

step 1: Add Maven to PATH

```
echo 'export PATH="/usr/local/apache-maven-3.9.6/bin:$PATH"' >> ~/.zshrc

```
Step 2: Reload configuration


```
source ~/.zshrc

```

Step 3: Verify
```
mvn --version
```
Step 4: Test your project
```
cd real-time-alert-correlation-engine/backend
mvn clean compile

```

## Maven version

Default locale: en_GB, platform encoding: UTF-8
OS name: "mac os x", version: "12.7.4", arch: "x86_6


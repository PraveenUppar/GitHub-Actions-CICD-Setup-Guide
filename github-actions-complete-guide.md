# GitHub Actions Complete Setup Guide

## 1. Complete Setup - Setting up GitHub Actions for Projects

### Prerequisites
- GitHub repository (public or private)
- Code project to build/test/deploy
- Basic understanding of YAML syntax

### Step 1: Create Workflow Directory Structure
```bash
mkdir -p .github/workflows
```

### Step 2: Create Your First Workflow File
Create `.github/workflows/ci.yml`:

```yaml
name: CI Pipeline

on:
  push:
    branches: [ "main", "develop" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run tests
      run: npm test
      
    - name: Build application
      run: npm run build
```

## 2. Setup Runners using a VM

### Self-Hosted Runner Configuration

#### Step 1: Prepare Your VM
- Deploy a VM (AWS EC2, Azure VM, GCP Compute Engine, or on-premises)
- Minimum requirements: 2 vCPU, 7GB RAM, 14GB disk space
- Operating System: Linux (Ubuntu/CentOS), Windows, or macOS
- Enable outbound HTTPS traffic (port 443)

#### Step 2: Download and Configure Runner
Navigate to your GitHub repository → Settings → Actions → Runners → New self-hosted runner

**For Linux:**
```bash
# Download
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Configure
./config.sh --url https://github.com/OWNER/REPOSITORY --token YOUR_TOKEN

# Install as service
sudo ./svc.sh install
sudo ./svc.sh start
```

**For Windows (PowerShell):**
```powershell
# Download
Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-win-x64-2.311.0.zip -OutFile actions-runner-win-x64-2.311.0.zip
Expand-Archive -Path actions-runner-win-x64-2.311.0.zip -DestinationPath .

# Configure
./config.cmd --url https://github.com/OWNER/REPOSITORY --token YOUR_TOKEN

# Install as service
./svc.sh install
./svc.sh start
```

#### Step 3: Verify Runner
- Check GitHub repository → Settings → Actions → Runners
- Runner should show as "Idle" with green dot

#### Step 4: Use Self-Hosted Runner in Workflows
```yaml
jobs:
  build:
    runs-on: self-hosted  # or use specific labels
    steps:
    - uses: actions/checkout@v4
    # ... rest of your steps
```

## 3. Configuring Workflows & Write the YAML Pipeline

### Advanced Workflow Configuration

#### Environment Variables and Secrets
```yaml
env:
  NODE_VERSION: '18'
  DATABASE_URL: ${{ secrets.DATABASE_URL }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
    - name: Deploy to production
      run: |
        echo "Deploying to ${{ vars.ENVIRONMENT_NAME }}"
        echo "Using database: ${{ secrets.DATABASE_URL }}"
```

#### Matrix Builds
```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [16, 18, 20]
        
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
```

#### Conditional Execution
```yaml
steps:
- name: Deploy to staging
  if: github.ref == 'refs/heads/develop'
  run: echo "Deploying to staging"
  
- name: Deploy to production
  if: github.ref == 'refs/heads/main'
  run: echo "Deploying to production"
```

#### Artifacts and Caching
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: ~/.npm
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
        
    - name: Build
      run: npm run build
      
    - name: Upload build artifacts
      uses: actions/upload-artifact@v3
      with:
        name: build-files
        path: dist/
```

### Complex Workflow Example
```yaml
name: Full Stack CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Lint code
      run: npm run lint
      
    - name: Run tests
      run: npm run test:coverage
      
    - name: Upload coverage reports
      uses: codecov/codecov-action@v3

  build-and-push:
    needs: lint-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
      
    steps:
    - uses: actions/checkout@v4
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
        
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha
          
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Deploy to Kubernetes
      run: |
        echo "Deploying to production cluster"
        # Add your deployment commands here
```

## 4. Full Stack CI/CD Pipelines

### Complete Full Stack Example

#### Frontend (React) Pipeline
```yaml
name: Frontend CI/CD

on:
  push:
    branches: [main]
    paths: ['frontend/**']

jobs:
  frontend-build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./frontend
        
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: frontend/package-lock.json
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run tests
      run: npm run test -- --coverage --watchAll=false
      
    - name: Build application
      run: npm run build
      
    - name: Deploy to S3
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      run: |
        aws s3 sync build/ s3://${{ secrets.S3_BUCKET_NAME }} --delete
        aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_ID }} --paths "/*"
```

#### Backend (Node.js/Express) Pipeline
```yaml
name: Backend CI/CD

on:
  push:
    branches: [main]
    paths: ['backend/**']

jobs:
  backend-build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./backend
        
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
          
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: backend/package-lock.json
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run database migrations
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
      run: npm run migrate
      
    - name: Run tests
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
      run: npm test
      
    - name: Build Docker image
      run: docker build -t backend:${{ github.sha }} .
      
    - name: Deploy to production
      if: github.ref == 'refs/heads/main'
      run: |
        echo "Deploy backend to production"
        # Add deployment commands
```

#### Database Migration Pipeline
```yaml
name: Database Migration

on:
  push:
    branches: [main]
    paths: ['database/migrations/**']

jobs:
  migrate:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        
    - name: Run migrations
      env:
        DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
      run: |
        npm install
        npm run migrate:up
        
    - name: Verify migration
      env:
        DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
      run: npm run migrate:status
```

### Monorepo Full Stack Pipeline
```yaml
name: Monorepo CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.changes.outputs.frontend }}
      backend: ${{ steps.changes.outputs.backend }}
      database: ${{ steps.changes.outputs.database }}
    steps:
    - uses: actions/checkout@v4
    - uses: dorny/paths-filter@v2
      id: changes
      with:
        filters: |
          frontend:
            - 'apps/frontend/**'
          backend:
            - 'apps/backend/**'
          database:
            - 'database/**'

  frontend:
    needs: changes
    if: ${{ needs.changes.outputs.frontend == 'true' }}
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build Frontend
      run: |
        cd apps/frontend
        npm ci
        npm run build

  backend:
    needs: changes
    if: ${{ needs.changes.outputs.backend == 'true' }}
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build Backend
      run: |
        cd apps/backend
        npm ci
        npm run build

  deploy:
    needs: [frontend, backend]
    if: always() && (needs.frontend.result == 'success' || needs.backend.result == 'success')
    runs-on: ubuntu-latest
    environment: production
    steps:
    - name: Deploy to production
      run: echo "Deploying updated services"
```

## Best Practices

### Security
- Use secrets for sensitive data
- Limit runner permissions with `permissions` key
- Use environment protection rules
- Pin action versions to specific commits

### Performance
- Use caching for dependencies
- Implement path-based filtering
- Use matrix builds for parallel execution
- Optimize Docker builds with multi-stage builds

### Maintenance
- Use reusable workflows
- Implement proper error handling
- Add status badges to README
- Monitor workflow execution times
- Regular runner maintenance and updates
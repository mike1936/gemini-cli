# 🚀 Development Environment Setup

This guide walks you through setting up a complete development environment for contributing to Gemini CLI, from basic setup to advanced debugging configurations.

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Repository Setup](#repository-setup)
- [Development Workflow](#development-workflow)
- [IDE Configuration](#ide-configuration)
- [Debugging Setup](#debugging-setup)
- [Testing Environment](#testing-environment)
- [Common Issues & Solutions](#common-issues--solutions)
- [Next Steps](#next-steps)

## Prerequisites

### Required Software

Ensure you have the following installed:

```bash
# Node.js (version 20 or higher)
node --version
# Should output: v20.x.x or higher

# npm (comes with Node.js)
npm --version
# Should output: 9.x.x or higher

# Git
git --version
# Should output: git version 2.x.x
```

### Optional but Recommended

```bash
# GitHub CLI (for easier repository management)
gh --version

# Docker (for integration testing with sandbox)
docker --version

# VS Code (recommended IDE)
code --version
```

### System Requirements

- **Operating System**: Windows 10+, macOS 10.15+, or Linux
- **Memory**: 4GB RAM minimum, 8GB recommended
- **Storage**: 2GB free space for repository and dependencies
- **Network**: Stable internet connection for package downloads

## Repository Setup

### 1. Fork and Clone

```bash
# Fork the repository on GitHub first, then:
git clone https://github.com/YOUR_USERNAME/gemini-cli.git
cd gemini-cli

# Add upstream remote for staying in sync
git remote add upstream https://github.com/google-gemini/gemini-cli.git
```

### 2. Install Dependencies

```bash
# Install all dependencies (this may take a few minutes)
npm ci

# Verify installation was successful
npm run build
```

### 3. Verify Setup

```bash
# Run the complete preflight check
npm run preflight

# This should complete without errors and includes:
# - Formatting check
# - Linting
# - Type checking
# - Building all packages
# - Running all tests
```

### 4. Test the CLI

```bash
# Start the development CLI
npm start

# You should see the Gemini CLI interface
# Press Ctrl+C to exit
```

## Development Workflow

### Daily Development Workflow

```bash
# 1. Start your development session
cd gemini-cli

# 2. Sync with upstream (daily)
git fetch upstream
git checkout main
git merge upstream/main

# 3. Create a feature branch
git checkout -b feature/my-new-feature

# 4. Make your changes
# ... edit files ...

# 5. Test frequently during development
npm run build        # Build all packages
npm run test         # Run unit tests
npm run lint         # Check code style

# 6. Test your changes manually
npm start            # Start the CLI to test interactively

# 7. Commit your changes
git add .
git commit -m "feat: add new feature"

# 8. Push and create pull request
git push origin feature/my-new-feature
```

### Incremental Development

For faster iteration during development:

```bash
# Build only the specific package you're working on
npm run build --workspace @google/gemini-cli-core

# Run tests for a specific package
npm run test --workspace @google/gemini-cli-core

# Build and start immediately
npm run build-and-start

# Watch mode for continuous building (in a separate terminal)
npm run typecheck -- --watch
```

### Package-Specific Development

When working on specific packages:

```bash
# Work on the CLI package
cd packages/cli
npm run build
npm run test

# Work on the core package
cd packages/core
npm run build
npm run test

# Return to root for full builds
cd ../..
npm run build
```

## IDE Configuration

### VS Code Setup (Recommended)

#### Required Extensions

Install these VS Code extensions for the best experience:

```json
// .vscode/extensions.json (already included in the repo)
{
  "recommendations": [
    "ms-vscode.vscode-typescript-next",
    "esbenp.prettier-vscode",
    "ms-vscode.vscode-eslint",
    "bradlc.vscode-tailwindcss",
    "vitest.explorer"
  ]
}
```

#### Workspace Settings

The repository includes pre-configured VS Code settings:

```json
// .vscode/settings.json (already configured)
{
  "typescript.preferences.useAliasesForRenames": false,
  "typescript.suggest.autoImports": true,
  "typescript.updateImportsOnFileMove.enabled": "always",
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "eslint.validate": ["typescript", "typescriptreact"],
  "vitest.enable": true
}
```

#### Launch Configuration

Debug configuration for VS Code:

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Gemini CLI",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/scripts/start.js",
      "env": {
        "DEBUG": "1"
      },
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "name": "Debug Tests",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/node_modules/vitest/dist/cli.js",
      "args": ["run", "--reporter=verbose"],
      "env": {
        "NODE_ENV": "test"
      },
      "console": "integratedTerminal"
    }
  ]
}
```

### Alternative IDEs

#### IntelliJ IDEA / WebStorm

```javascript
// Configure TypeScript service
// File → Settings → Languages & Frameworks → TypeScript
// Enable TypeScript Language Service
// Set service directory to: node_modules/typescript/lib
```

#### Vim/Neovim

```vim
" .vimrc / init.vim configuration for TypeScript development
Plug 'neoclide/coc.nvim', {'branch': 'release'}
Plug 'prettier/vim-prettier', { 'do': 'npm install' }

" CoC extensions
:CocInstall coc-tsserver coc-eslint coc-prettier
```

## Debugging Setup

### Debug Console Logging

Enable debug logging for development:

```bash
# Enable verbose logging
export DEBUG=1
npm start

# Or set environment variable in your shell profile
echo 'export DEBUG=1' >> ~/.bashrc  # or ~/.zshrc
```

### Debugging Specific Components

```typescript
// Add debug logging to your code
import { debugLog } from '../utils/debug.js';

export class MyTool {
  async execute(): Promise<MyResult> {
    debugLog('MyTool', 'Starting execution with params:', this.params);

    try {
      const result = await this.performOperation();
      debugLog('MyTool', 'Operation completed successfully:', result);
      return result;
    } catch (error) {
      debugLog('MyTool', 'Operation failed:', error);
      throw error;
    }
  }
}
```

### Node.js Inspector

For deep debugging, use Node.js inspector:

```bash
# Start with debugger
npm run debug

# This starts the CLI with --inspect-brk flag
# Open Chrome and go to: chrome://inspect
# Click "Open dedicated DevTools for Node"
```

### Testing with Debug Mode

```bash
# Run tests with debugging
NODE_OPTIONS="--inspect-brk" npm test

# Run specific test with debugging
NODE_OPTIONS="--inspect-brk" npm test -- packages/core/src/tools/shell.test.ts
```

## Testing Environment

### Unit Testing Setup

```bash
# Run all tests
npm test

# Run tests with coverage
npm run test:ci

# Run tests in watch mode (during development)
npm test -- --watch

# Run specific test file
npm test -- packages/core/src/tools/fileSystem.test.ts

# Run tests matching a pattern
npm test -- --grep "should validate parameters"
```

### Integration Testing

```bash
# Run integration tests (requires more setup)
npm run test:e2e

# Run with specific sandbox configuration
npm run test:integration:sandbox:docker
```

### Mock Configuration for Testing

When writing tests, use the existing mock patterns:

```typescript
// Example test setup
import { vi } from 'vitest';

// Mock external dependencies at the top of your test file
vi.mock('@google/genai', async (importOriginal) => {
  const actual = await importOriginal();
  return {
    ...actual,
    GenerativeModel: vi.fn(() => ({
      generateContent: vi.fn().mockResolvedValue({
        response: { text: () => 'Mock response' }
      }))
    }))
  };
});

vi.mock('fs/promises', () => ({
  readFile: vi.fn(),
  writeFile: vi.fn(),
  access: vi.fn()
}));
```

### Testing New Tools

```typescript
// packages/core/src/tools/myTool.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { MyTool } from './myTool.js';

describe('MyTool', () => {
  beforeEach(() => {
    vi.resetAllMocks();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('should execute successfully with valid parameters', async () => {
    const tool = new MyTool({ param1: 'value1' });
    const result = await tool.execute(new AbortController().signal);

    expect(result.success).toBe(true);
    expect(result.data).toBeDefined();
  });

  it('should handle errors gracefully', async () => {
    // Mock a failure scenario
    vi.mocked(someExternalFunction).mockRejectedValue(new Error('Test error'));

    const tool = new MyTool({ param1: 'invalid' });
    const result = await tool.execute(new AbortController().signal);

    expect(result.success).toBe(false);
    expect(result.error).toContain('Test error');
  });
});
```

## Common Issues & Solutions

### Installation Issues

#### **Issue: `npm ci` fails with permission errors**

**Solution:**

```bash
# Fix npm permissions
sudo chown -R $(whoami) ~/.npm
npm cache clean --force
npm ci
```

#### **Issue: Node.js version too old**

**Solution:**

```bash
# Install Node Version Manager (nvm)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc

# Install and use the correct Node.js version
nvm install 20
nvm use 20
nvm alias default 20
```

### Build Issues

#### **Issue: TypeScript compilation errors**

**Solution:**

```bash
# Clear TypeScript cache
rm -rf packages/*/dist
rm -rf packages/*/.tsbuildinfo

# Rebuild from scratch
npm run clean
npm ci
npm run build
```

#### **Issue: ESLint errors**

**Solution:**

```bash
# Auto-fix most ESLint issues
npm run lint:fix

# For specific files
npx eslint --fix packages/core/src/myFile.ts
```

### Runtime Issues

#### **Issue: "Module not found" errors**

**Solution:**

```bash
# Ensure all packages are built
npm run build

# Check for missing dependencies
npm run typecheck
```

#### **Issue: Tests failing with mock issues**

**Solution:**

```typescript
// Ensure mocks are properly hoisted
const mockFunction = vi.hoisted(() => vi.fn());

vi.mock('external-module', () => ({
  someFunction: mockFunction,
}));
```

### Environment Issues

#### **Issue: Git hooks not running**

**Solution:**

```bash
# Reinstall husky hooks
npx husky install
```

#### **Issue: VS Code TypeScript errors**

**Solution:**

```bash
# Restart TypeScript service in VS Code
# Command Palette (Ctrl+Shift+P) → "TypeScript: Restart TS Server"

# Or reload VS Code window
# Command Palette → "Developer: Reload Window"
```

## Advanced Configuration

### Performance Optimization

```bash
# Enable TypeScript project references for faster builds
npm run build -- --build

# Use parallel processing for tests
npm test -- --reporter=verbose --run --passWithNoTests

# Optimize VS Code for large codebases
# Add to VS Code settings.json:
{
  "typescript.disableAutomaticTypeAcquisition": true,
  "typescript.preferences.includePackageJsonAutoImports": "off"
}
```

### Custom Scripts

Add custom development scripts to your `package.json`:

```json
{
  "scripts": {
    "dev": "npm run build && npm start",
    "test:watch": "npm test -- --watch",
    "test:debug": "NODE_OPTIONS='--inspect-brk' npm test",
    "type:watch": "npm run typecheck -- --watch",
    "quick-check": "npm run lint && npm run typecheck"
  }
}
```

## Next Steps

Now that your development environment is set up, continue your learning journey:

### **🛠️ Next: [Creating Your First Tool](./08-first-tool.md)**

Build your first tool from scratch with hands-on guidance.

### **🔍 Related Topics**

- [Testing & Debugging Guide](./09-testing-debugging.md) - Advanced testing and debugging techniques
- [Tool System Deep Dive](./06-tool-system.md) - Understanding the tool architecture
- [Contributing Guidelines](../../CONTRIBUTING.md) - Process and community guidelines

### **🛠️ Validation Exercises**

1. **Verify Your Setup**: Run `npm run preflight` and ensure it passes
2. **Make a Small Change**: Edit a comment in a file, build, and test
3. **Run the CLI**: Start the CLI and have a conversation with Gemini
4. **Debug a Tool**: Set a breakpoint in an existing tool and trace its execution

### **💡 Key Takeaways**

- Use `npm run preflight` before submitting any changes
- VS Code is recommended for the best development experience
- Debug mode provides valuable insights during development
- Tests should be run frequently during development
- The development workflow emphasizes incremental building and testing

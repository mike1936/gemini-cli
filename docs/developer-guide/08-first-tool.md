# 🛠️ Creating Your First Tool

This hands-on guide walks you through creating a complete tool from scratch, teaching you the patterns and practices used throughout Gemini CLI's tool system.

## 📋 Table of Contents

- [Tool Planning](#tool-planning)
- [Step-by-Step Implementation](#step-by-step-implementation)
- [Testing Your Tool](#testing-your-tool)
- [Registering and Integration](#registering-and-integration)
- [Advanced Features](#advanced-features)
- [Common Patterns](#common-patterns)
- [Next Steps](#next-steps)

## Tool Planning

### Choose Your Tool

For this tutorial, we'll create a **File Stats Tool** that provides detailed information about files and directories. This tool will demonstrate:

- Parameter validation
- File system operations
- Error handling
- User confirmation
- Result formatting

### Tool Specification

**Name**: `file_stats`  
**Purpose**: Get detailed statistics about files or directories  
**Parameters**:

- `path` (required): File or directory path
- `includeHidden` (optional): Include hidden files in directory stats
- `format` (optional): Output format ('json' | 'table')

**Behavior**:

- For files: size, permissions, creation/modification dates
- For directories: file count, total size, subdirectory count
- Requires confirmation for large directories (>1000 files)

### Define Types First

```typescript
// packages/core/src/tools/fileStats.ts

export interface FileStatsParams {
  path: string;
  includeHidden?: boolean;
  format?: 'json' | 'table';
}

export interface FileStatsResult extends ToolResult {
  path: string;
  type: 'file' | 'directory';
  size?: number;
  fileCount?: number;
  permissions?: string;
  created?: Date;
  modified?: Date;
  subdirectories?: number;
  children?: Array<{
    name: string;
    type: 'file' | 'directory';
    size: number;
  }>;
}
```

## Step-by-Step Implementation

### Step 1: Create the Tool File

Create the new tool file:

```bash
touch packages/core/src/tools/fileStats.ts
```

### Step 2: Implement Basic Structure

```typescript
// packages/core/src/tools/fileStats.ts
import { promises as fs } from 'fs';
import * as path from 'path';
import { BaseToolInvocation, BaseToolBuilder } from './tools.js';
import type {
  ToolResult,
  ToolCallConfirmationDetails,
  ToolLocation,
} from './tools.js';

export interface FileStatsParams {
  path: string;
  includeHidden?: boolean;
  format?: 'json' | 'table';
}

export interface FileStatsResult extends ToolResult {
  path: string;
  type: 'file' | 'directory';
  size?: number;
  fileCount?: number;
  permissions?: string;
  created?: Date;
  modified?: Date;
  subdirectories?: number;
  children?: Array<{
    name: string;
    type: 'file' | 'directory';
    size: number;
  }>;
}

export class FileStatsTool extends BaseToolInvocation<
  FileStatsParams,
  FileStatsResult
> {
  constructor(readonly params: FileStatsParams) {
    super(params);
  }

  getDescription(): string {
    return `Getting statistics for: ${this.params.path}`;
  }

  toolLocations(): ToolLocation[] {
    return [{ path: this.params.path }];
  }

  async shouldConfirmExecute(): Promise<ToolCallConfirmationDetails | false> {
    // We'll implement this in Step 4
    return false;
  }

  async execute(): Promise<FileStatsResult> {
    // We'll implement this in Step 3
    throw new Error('Not implemented yet');
  }
}
```

### Step 3: Implement Core Logic

```typescript
async execute(signal: AbortSignal): Promise<FileStatsResult> {
  try {
    // Resolve and validate path
    const resolvedPath = path.resolve(this.params.path);

    // Get file/directory stats
    const stats = await fs.stat(resolvedPath);

    if (stats.isFile()) {
      return await this.getFileStats(resolvedPath, stats);
    } else if (stats.isDirectory()) {
      return await this.getDirectoryStats(resolvedPath, stats, signal);
    } else {
      return {
        success: false,
        error: `Path is neither a file nor directory: ${resolvedPath}`,
        path: resolvedPath,
        type: 'file'
      };
    }
  } catch (error) {
    return {
      success: false,
      error: `Failed to get stats: ${error.message}`,
      path: this.params.path,
      type: 'file'
    };
  }
}

private async getFileStats(filePath: string, stats: fs.Stats): Promise<FileStatsResult> {
  return {
    success: true,
    path: filePath,
    type: 'file',
    size: stats.size,
    permissions: this.formatPermissions(stats.mode),
    created: stats.birthtime,
    modified: stats.mtime
  };
}

private async getDirectoryStats(
  dirPath: string,
  stats: fs.Stats,
  signal: AbortSignal
): Promise<FileStatsResult> {
  const children = await this.getDirectoryChildren(dirPath, signal);

  const files = children.filter(child => child.type === 'file');
  const subdirs = children.filter(child => child.type === 'directory');
  const totalSize = files.reduce((sum, file) => sum + file.size, 0);

  return {
    success: true,
    path: dirPath,
    type: 'directory',
    fileCount: files.length,
    subdirectories: subdirs.length,
    size: totalSize,
    permissions: this.formatPermissions(stats.mode),
    created: stats.birthtime,
    modified: stats.mtime,
    children: this.params.format === 'json' ? children : undefined
  };
}

private async getDirectoryChildren(
  dirPath: string,
  signal: AbortSignal
): Promise<Array<{ name: string; type: 'file' | 'directory'; size: number }>> {
  const entries = await fs.readdir(dirPath);
  const children = [];

  for (const entry of entries) {
    // Check for cancellation
    if (signal.aborted) {
      throw new Error('Operation cancelled');
    }

    // Skip hidden files unless requested
    if (!this.params.includeHidden && entry.startsWith('.')) {
      continue;
    }

    try {
      const entryPath = path.join(dirPath, entry);
      const entryStats = await fs.stat(entryPath);

      children.push({
        name: entry,
        type: entryStats.isDirectory() ? 'directory' : 'file',
        size: entryStats.size
      });
    } catch (error) {
      // Skip entries we can't read (permission issues, broken symlinks, etc.)
      continue;
    }
  }

  return children;
}

private formatPermissions(mode: number): string {
  const perms = (mode & parseInt('777', 8)).toString(8);
  return `0${perms}`;
}
```

### Step 4: Implement Confirmation Logic

```typescript
async shouldConfirmExecute(): Promise<ToolCallConfirmationDetails | false> {
  try {
    const stats = await fs.stat(this.params.path);

    // Files don't need confirmation
    if (stats.isFile()) {
      return false;
    }

    // For directories, check if they're large
    if (stats.isDirectory()) {
      const preview = await this.getDirectoryPreview();

      // Require confirmation for large directories
      if (preview.estimatedFileCount > 1000) {
        return {
          type: 'info',
          title: 'Large Directory Analysis',
          prompt: `This directory appears to contain ${preview.estimatedFileCount}+ files. Analyzing may take some time and system resources.`,
          onConfirm: async (outcome) => {
            if (outcome !== 'approved') {
              throw new Error('Directory analysis cancelled by user');
            }
          }
        };
      }
    }

    return false;
  } catch (error) {
    // If we can't read the path, let execute() handle the error
    return false;
  }
}

private async getDirectoryPreview(): Promise<{ estimatedFileCount: number }> {
  try {
    const entries = await fs.readdir(this.params.path);
    // Quick estimate - count visible entries
    const visibleEntries = this.params.includeHidden
      ? entries
      : entries.filter(entry => !entry.startsWith('.'));

    return { estimatedFileCount: visibleEntries.length };
  } catch {
    return { estimatedFileCount: 0 };
  }
}
```

### Step 5: Create the Tool Builder

```typescript
export class FileStatsToolBuilder extends BaseToolBuilder<
  FileStatsParams,
  FileStatsResult
> {
  name = 'file_stats';
  description =
    'Get detailed statistics about files or directories including size, permissions, and contents';

  schema = {
    type: 'object',
    properties: {
      path: {
        type: 'string',
        description: 'Path to the file or directory to analyze',
      },
      includeHidden: {
        type: 'boolean',
        description: 'Include hidden files in directory analysis',
        default: false,
      },
      format: {
        type: 'string',
        enum: ['json', 'table'],
        description: 'Output format for the results',
        default: 'table',
      },
    },
    required: ['path'],
  } as const;

  protected async validateParams(params: unknown): Promise<FileStatsParams> {
    if (!this.isValidParams(params)) {
      throw new Error('Invalid parameters for file_stats tool');
    }

    // Validate path
    if (typeof params.path !== 'string' || params.path.trim() === '') {
      throw new Error(
        'Path parameter is required and must be a non-empty string',
      );
    }

    // Resolve relative paths
    const resolvedPath = path.resolve(params.path);

    return {
      path: resolvedPath,
      includeHidden: params.includeHidden ?? false,
      format: params.format ?? 'table',
    };
  }

  protected createTool(params: FileStatsParams): FileStatsTool {
    return new FileStatsTool(params);
  }

  private isValidParams(params: unknown): params is Partial<FileStatsParams> {
    return typeof params === 'object' && params !== null && 'path' in params;
  }
}
```

## Testing Your Tool

### Step 1: Create Test File

```typescript
// packages/core/src/tools/fileStats.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { promises as fs } from 'fs';
import { FileStatsTool, FileStatsToolBuilder } from './fileStats.js';

// Mock fs operations
vi.mock('fs', async (importOriginal) => {
  const actual = await importOriginal();
  return {
    ...actual,
    promises: {
      stat: vi.fn(),
      readdir: vi.fn(),
    },
  };
});

const mockFs = vi.mocked(fs);

describe('FileStatsTool', () => {
  beforeEach(() => {
    vi.resetAllMocks();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  describe('FileStatsToolBuilder', () => {
    const builder = new FileStatsToolBuilder();

    it('should validate correct parameters', async () => {
      const params = { path: '/test/file.txt' };
      const validated = await builder.validateParams(params);

      expect(validated.path).toBe('/test/file.txt');
      expect(validated.includeHidden).toBe(false);
      expect(validated.format).toBe('table');
    });

    it('should reject invalid parameters', async () => {
      await expect(builder.validateParams({})).rejects.toThrow();
      await expect(builder.validateParams({ path: '' })).rejects.toThrow();
      await expect(builder.validateParams({ path: 123 })).rejects.toThrow();
    });
  });

  describe('FileStatsTool execution', () => {
    it('should analyze a file successfully', async () => {
      // Mock file stats
      mockFs.stat.mockResolvedValue({
        isFile: () => true,
        isDirectory: () => false,
        size: 1024,
        mode: 0o644,
        birthtime: new Date('2023-01-01'),
        mtime: new Date('2023-01-02'),
      } as any);

      const tool = new FileStatsTool({ path: '/test/file.txt' });
      const result = await tool.execute(new AbortController().signal);

      expect(result.success).toBe(true);
      expect(result.type).toBe('file');
      expect(result.size).toBe(1024);
      expect(result.permissions).toBe('0644');
    });

    it('should analyze a directory successfully', async () => {
      // Mock directory stats
      mockFs.stat.mockResolvedValue({
        isFile: () => false,
        isDirectory: () => true,
        mode: 0o755,
        birthtime: new Date('2023-01-01'),
        mtime: new Date('2023-01-02'),
      } as any);

      // Mock directory contents
      mockFs.readdir.mockResolvedValue(['file1.txt', 'file2.txt'] as any);

      // Mock stats for directory contents
      mockFs.stat
        .mockResolvedValueOnce({
          /* directory stats */
        } as any)
        .mockResolvedValueOnce({ isDirectory: () => false, size: 100 } as any)
        .mockResolvedValueOnce({ isDirectory: () => false, size: 200 } as any);

      const tool = new FileStatsTool({ path: '/test/dir' });
      const result = await tool.execute(new AbortController().signal);

      expect(result.success).toBe(true);
      expect(result.type).toBe('directory');
      expect(result.fileCount).toBe(2);
      expect(result.size).toBe(300);
    });

    it('should handle file not found errors', async () => {
      mockFs.stat.mockRejectedValue(
        new Error('ENOENT: no such file or directory'),
      );

      const tool = new FileStatsTool({ path: '/nonexistent/file.txt' });
      const result = await tool.execute(new AbortController().signal);

      expect(result.success).toBe(false);
      expect(result.error).toContain('Failed to get stats');
    });

    it('should respect cancellation signals', async () => {
      const abortController = new AbortController();

      // Mock slow operation
      mockFs.stat.mockResolvedValue({
        isFile: () => false,
        isDirectory: () => true,
      } as any);

      mockFs.readdir.mockImplementation(() => {
        return new Promise((resolve) => {
          // Simulate slow operation
          setTimeout(() => resolve(['file1.txt'] as any), 100);
        });
      });

      const tool = new FileStatsTool({ path: '/test/dir' });

      // Cancel after starting
      setTimeout(() => abortController.abort(), 10);

      const result = await tool.execute(abortController.signal);
      expect(result.success).toBe(false);
      expect(result.error).toContain('cancelled');
    });
  });

  describe('confirmation logic', () => {
    it('should not require confirmation for files', async () => {
      mockFs.stat.mockResolvedValue({
        isFile: () => true,
        isDirectory: () => false,
      } as any);

      const tool = new FileStatsTool({ path: '/test/file.txt' });
      const confirmation = await tool.shouldConfirmExecute(
        new AbortController().signal,
      );

      expect(confirmation).toBe(false);
    });

    it('should require confirmation for large directories', async () => {
      mockFs.stat.mockResolvedValue({
        isFile: () => false,
        isDirectory: () => true,
      } as any);

      // Mock large directory
      const largeFileList = Array.from(
        { length: 1500 },
        (_, i) => `file${i}.txt`,
      );
      mockFs.readdir.mockResolvedValue(largeFileList as any);

      const tool = new FileStatsTool({ path: '/large/dir' });
      const confirmation = await tool.shouldConfirmExecute(
        new AbortController().signal,
      );

      expect(confirmation).not.toBe(false);
      expect(confirmation?.type).toBe('info');
      expect(confirmation?.title).toContain('Large Directory');
    });
  });
});
```

### Step 2: Run Tests

```bash
# Run your specific test
npm test -- packages/core/src/tools/fileStats.test.ts

# Run with coverage
npm test -- --coverage packages/core/src/tools/fileStats.test.ts

# Run in watch mode while developing
npm test -- --watch packages/core/src/tools/fileStats.test.ts
```

## Registering and Integration

### Step 1: Add to Tool Registry

```typescript
// packages/core/src/tools/index.ts
export { FileStatsTool, FileStatsToolBuilder } from './fileStats.js';
```

### Step 2: Register in Default Configuration

```typescript
// packages/core/src/config/defaultToolRegistry.ts
import { FileStatsToolBuilder } from '../tools/fileStats.js';

export function createDefaultToolRegistry(): ToolRegistry {
  const registry = new ToolRegistry();

  // ... existing tools ...

  // Add our new tool
  registry.registerTool('file_stats', new FileStatsToolBuilder());

  return registry;
}
```

### Step 3: Manual Testing

```bash
# Build the project
npm run build

# Start the CLI
npm start

# Test your tool
# In the CLI, type something like:
# "Can you get statistics for the package.json file?"
# or
# "What files are in the src directory?"
```

## Advanced Features

### Add Output Formatting

```typescript
private formatOutput(result: FileStatsResult): string {
  if (this.params.format === 'json') {
    return JSON.stringify(result, null, 2);
  }

  // Table format
  const lines = [];
  lines.push(`📊 File Statistics: ${result.path}`);
  lines.push(`Type: ${result.type}`);

  if (result.size !== undefined) {
    lines.push(`Size: ${this.formatSize(result.size)}`);
  }

  if (result.permissions) {
    lines.push(`Permissions: ${result.permissions}`);
  }

  if (result.fileCount !== undefined) {
    lines.push(`Files: ${result.fileCount}`);
  }

  if (result.subdirectories !== undefined) {
    lines.push(`Subdirectories: ${result.subdirectories}`);
  }

  return lines.join('\n');
}

private formatSize(bytes: number): string {
  const units = ['B', 'KB', 'MB', 'GB'];
  let size = bytes;
  let unitIndex = 0;

  while (size >= 1024 && unitIndex < units.length - 1) {
    size /= 1024;
    unitIndex++;
  }

  return `${size.toFixed(1)} ${units[unitIndex]}`;
}
```

### Add Progress Reporting

```typescript
async execute(
  signal: AbortSignal,
  updateOutput?: (output: string) => void
): Promise<FileStatsResult> {
  updateOutput?.('🔍 Analyzing path...');

  const stats = await fs.stat(this.params.path);

  if (stats.isDirectory()) {
    updateOutput?.('📁 Reading directory contents...');
    const children = await this.getDirectoryChildren(this.params.path, signal);
    updateOutput?.(`📊 Found ${children.length} items`);
  }

  updateOutput?.('✅ Analysis complete');

  // ... rest of implementation
}
```

### Add Caching

```typescript
class FileStatsTool extends BaseToolInvocation<
  FileStatsParams,
  FileStatsResult
> {
  private static cache = new Map<
    string,
    { result: FileStatsResult; timestamp: number }
  >();

  async execute(signal: AbortSignal): Promise<FileStatsResult> {
    const cacheKey = `${this.params.path}:${this.params.includeHidden}`;
    const cached = FileStatsTool.cache.get(cacheKey);

    // Use cache if recent (5 minutes)
    if (cached && Date.now() - cached.timestamp < 5 * 60 * 1000) {
      return { ...cached.result, fromCache: true };
    }

    const result = await this.performAnalysis(signal);

    // Cache successful results
    if (result.success) {
      FileStatsTool.cache.set(cacheKey, {
        result,
        timestamp: Date.now(),
      });
    }

    return result;
  }
}
```

## Common Patterns

### Pattern 1: Parameter Validation

```typescript
protected async validateParams(params: unknown): Promise<MyParams> {
  // 1. Type guard
  if (!this.isValidShape(params)) {
    throw new Error('Invalid parameter shape');
  }

  // 2. Required field validation
  if (!params.requiredField) {
    throw new Error('Required field is missing');
  }

  // 3. Sanitization
  const sanitized = {
    ...params,
    path: path.resolve(params.path),
    timeout: Math.max(1000, params.timeout || 5000)
  };

  // 4. Additional validation
  if (sanitized.value < 0) {
    throw new Error('Value must be positive');
  }

  return sanitized;
}
```

### Pattern 2: Error Handling

```typescript
async execute(signal: AbortSignal): Promise<MyResult> {
  try {
    return await this.performOperation(signal);
  } catch (error) {
    // Log error with context
    console.error(`${this.constructor.name} failed:`, {
      error: error.message,
      params: this.params
    });

    // Return structured error
    return {
      success: false,
      error: this.formatErrorMessage(error),
      errorCode: this.getErrorCode(error),
      recoveryHint: this.getRecoveryHint(error)
    };
  }
}
```

### Pattern 3: Cancellation Support

```typescript
async performLongOperation(signal: AbortSignal): Promise<void> {
  for (const item of this.items) {
    // Check for cancellation frequently
    if (signal.aborted) {
      throw new Error('Operation cancelled');
    }

    await this.processItem(item);
  }
}
```

## Next Steps

Congratulations! You've created your first tool. Continue your learning journey:

### **🐛 Next: [Testing & Debugging Guide](./09-testing-debugging.md)**

Learn advanced testing and debugging techniques for your tools.

### **🔍 Related Topics**

- [Tool System Deep Dive](./06-tool-system.md) - Advanced tool patterns
- [Core Classes & Interfaces](./05-core-classes.md) - Type system details
- [Performance Considerations](./11-performance.md) - Optimization techniques

### **🛠️ Practice Exercises**

1. **Extend Your Tool**: Add new features like file type detection or permission analysis
2. **Create Another Tool**: Build a tool for a different domain (network, database, etc.)
3. **Optimize Performance**: Add caching, progress reporting, or concurrent processing
4. **Error Scenarios**: Test your tool with various error conditions

### **💡 Key Takeaways**

- Always validate parameters thoroughly
- Handle errors gracefully with structured results
- Support cancellation for long-running operations
- Write comprehensive tests for all scenarios
- Use confirmation for potentially dangerous operations
- Follow consistent patterns for maintainability

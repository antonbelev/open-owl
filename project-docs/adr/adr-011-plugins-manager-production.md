# ADR-011: Plugins Manager - Production Readiness Plan

**Status:** Proposed
**Date:** 2025-12-13
**Decision Makers:** Product Team, Engineering Team
**Stakeholders:** Open Owl Users, Claude Code Users
**Related:** ADR-004 (MCP Manager), ADR-010 (Remote MCP Servers Discovery)

---

## Executive Summary

This ADR outlines the plan to bring Open Owl's Plugins Manager from "Under Development" status to production readiness. The feature will provide:

1. **Out-of-the-box access to official Anthropic plugins** from `github.com/anthropics/claude-code/plugins`
2. **Secure marketplace management** with trust indicators and security warnings
3. **CLI delegation pattern** (consistent with MCP Manager) for reliable plugin operations
4. **Plugin verification and pre-installation checks** before committing to installation

The key insight from our MCP Manager (ADR-004) is that **delegating to Claude Code CLI commands** provides better reliability, automatic feature parity, and simpler maintenance. This ADR extends that pattern to plugins.

---

## Context

### Current State Analysis

Open Owl has a substantial plugins implementation that is marked as "Under Development":

**What's Implemented:**
- ✅ `PluginsService` with marketplace and plugin management (660 lines)
- ✅ `PluginsManager` UI with tabs, search, filters (989 lines)
- ✅ `usePlugins` hook with full CRUD operations (340 lines)
- ✅ IPC handlers for 10 plugin operations
- ✅ Plugin card display with component counts
- ✅ Marketplace management (add/remove)
- ✅ GitHub repo intelligence (stars, forks, health score)
- ✅ Grid/List view modes

**What's Missing for Production:**
- ❌ No integration with Claude Code's plugin CLI commands
- ❌ Custom file management (writes to `~/.claude/plugins/` directly)
- ❌ No official Anthropic marketplace pre-configured
- ❌ No security warnings or trust indicators
- ❌ No plugin verification before installation
- ❌ Enable/disable doesn't sync with Claude Code
- ❌ No project-level plugin support

### Claude Code Plugin System

Based on the official documentation, Claude Code plugins:

1. **Structure:** `.claude-plugin/plugin.json` manifest with `commands/`, `agents/`, `skills/`, `hooks/`, `.mcp.json`
2. **Marketplaces:** `.claude-plugin/marketplace.json` catalogs with plugin listings
3. **Installation:** Via `/plugin install plugin@marketplace` or `/plugin marketplace add <source>`
4. **Scopes:** User-level and project-level (via `.claude/settings.json`)
5. **Trust Model:** Based on repository folder trust

### Official Anthropic Plugins (13 available)

From `github.com/anthropics/claude-code/plugins`:

| Plugin | Purpose | Components |
|--------|---------|------------|
| **agent-sdk-dev** | Development kit for Claude Agent SDK | Agents, Skills |
| **claude-opus-4-5-migration** | Automated migration to Opus 4.5 | Commands |
| **code-review** | PR code review with specialized agents | Agents |
| **commit-commands** | Git workflow automation | Commands |
| **explanatory-output-style** | Educational insights about implementation | Skills |
| **feature-dev** | 7-phase feature development workflow | Agents, Commands |
| **frontend-design** | Production-grade UI creation | Agents, Skills |
| **hookify** | Custom hook creation for unwanted behaviors | Hooks |
| **learning-output-style** | Interactive learning mode | Skills |
| **plugin-dev** | Plugin development toolkit (7 skills) | Skills, Commands |
| **pr-review-toolkit** | Specialized PR analysis agents | Agents |
| **ralph-wiggum** | Self-referential AI loops | Agents |
| **security-guidance** | Security monitoring for 9 dangerous patterns | Hooks |

---

## Decision

We will refactor the Plugins Manager to:

1. **Delegate to Claude Code CLI** for all plugin operations (install, uninstall, enable, disable)
2. **Pre-configure the official Anthropic marketplace** as a first-class data source
3. **Add security and trust indicators** to help users make informed decisions
4. **Implement plugin verification** before installation
5. **Remove direct file manipulation** in favor of CLI commands

### Design Principles

1. **CLI Delegation:** Same pattern as MCP Manager - let Claude Code handle file operations
2. **Security First:** Clear trust indicators, security warnings, permission explanations
3. **Official First:** Anthropic plugins prominently featured and pre-verified
4. **Verify Before Install:** Show what a plugin will do before committing

---

## Detailed Design

### 1. Architecture Changes

#### Current Architecture (Direct File Manipulation)

```
┌─────────────────────────────────────────────────────────────────┐
│  Current PluginsService                                          │
├─────────────────────────────────────────────────────────────────┤
│  ❌ Writes directly to ~/.claude/plugins/                        │
│  ❌ Manages installed_plugins.json ourselves                     │
│  ❌ Git clone via execSync for installation                      │
│  ❌ No sync with Claude Code's plugin state                      │
└─────────────────────────────────────────────────────────────────┘
```

#### Proposed Architecture (CLI Delegation)

```
┌─────────────────────────────────────────────────────────────────┐
│                     Plugins Manager (Renderer)                   │
├─────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐ │
│  │ Official       │  │ Marketplace    │  │ Installed Plugins  │ │
│  │ Anthropic Hub  │  │ Browser        │  │ Manager            │ │
│  └───────┬────────┘  └───────┬────────┘  └─────────┬──────────┘ │
│          │                   │                     │             │
│          └───────────────────┼─────────────────────┘             │
│                              ▼                                   │
│                    ┌─────────────────┐                          │
│                    │ usePlugins Hook │                          │
│                    └────────┬────────┘                          │
└─────────────────────────────┼───────────────────────────────────┘
                              ▼ IPC
┌─────────────────────────────────────────────────────────────────┐
│                        Main Process                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  PluginsService (Refactored)                                 ││
│  │  ├─ Delegates to `/plugin` CLI commands                      ││
│  │  ├─ Parses CLI output for UI display                         ││
│  │  ├─ Fetches marketplace manifests for browsing               ││
│  │  └─ Calculates security/trust metrics                        ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Claude Code CLI                                             ││
│  │  ├─ /plugin install plugin@marketplace                       ││
│  │  ├─ /plugin uninstall plugin@marketplace                     ││
│  │  ├─ /plugin enable plugin@marketplace                        ││
│  │  ├─ /plugin disable plugin@marketplace                       ││
│  │  ├─ /plugin marketplace add <source>                         ││
│  │  └─ /plugin marketplace remove <name>                        ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 2. Official Anthropic Plugins Hub

A dedicated section featuring the 13 official Anthropic plugins:

```
┌──────────────────────────────────────────────────────────────────┐
│  Plugins  >  Official Anthropic Plugins                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Anthropic                                                   │ │
│  │ Official plugins from the Claude Code team                  │ │
│  │ ✓ Verified  •  ✓ Maintained  •  ✓ Secure                   │ │
│  │ github.com/anthropics/claude-code/plugins                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  Featured                                                         │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │  [Star Icon]                                                  ││
│  │  code-review                           ✓ Official  Agents    ││
│  │  Automated PR code review using multiple specialized agents  ││
│  │  with confidence-based scoring                               ││
│  │                                                               ││
│  │  Components: 4 agents, 2 skills                              ││
│  │  [View Details]  [Install →]                                 ││
│  └──────────────────────────────────────────────────────────────┘│
│  ┌──────────────────────────────────────────────────────────────┐│
│  │  feature-dev                           ✓ Official  Workflow  ││
│  │  Comprehensive feature development workflow with a           ││
│  │  structured 7-phase approach                                 ││
│  │                                                               ││
│  │  Components: 3 agents, 5 commands                            ││
│  │  [View Details]  [Install →]                                 ││
│  └──────────────────────────────────────────────────────────────┘│
│  ┌──────────────────────────────────────────────────────────────┐│
│  │  security-guidance                     ✓ Official  Hooks     ││
│  │  Security reminder hook monitoring for 9 dangerous patterns  ││
│  │                                                               ││
│  │  Components: 1 hook                                          ││
│  │  [View Details]  [Install →]                                 ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                   │
│  [View All 13 Official Plugins →]                                │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 3. Plugin Pre-Installation Verification

Before installing any plugin, show users exactly what they're getting:

```
┌──────────────────────────────────────────────────────────────────┐
│  Install Plugin: code-review                                [×]   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ ✓ Trust Verification                                        │ │
│  │                                                             │ │
│  │ Source: github.com/anthropics/claude-code/plugins          │ │
│  │ Publisher: Anthropic (Verified ✓)                          │ │
│  │ License: MIT                                                │ │
│  │ Last Updated: 3 days ago                                    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ 📦 Components to be Installed                               │ │
│  │                                                             │ │
│  │ Agents (4):                                                 │ │
│  │   • architecture-reviewer - Reviews code architecture       │ │
│  │   • security-scanner - Scans for security issues           │ │
│  │   • performance-analyst - Analyzes performance impact      │ │
│  │   • test-coverage-checker - Verifies test coverage         │ │
│  │                                                             │ │
│  │ Skills (2):                                                 │ │
│  │   • code-style-enforcer - Enforces code style guidelines   │ │
│  │   • documentation-checker - Validates documentation        │ │
│  │                                                             │ │
│  │ Commands (0)  •  Hooks (0)  •  MCP Servers (0)              │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ ⚙️ Installation Scope                                       │ │
│  │                                                             │ │
│  │ ◉ User (available in all projects)                         │ │
│  │ ○ Project (select project below)                           │ │
│  │                                                             │ │
│  │ This plugin will be available whenever you use Claude Code. │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ 📋 CLI Command Preview                                      │ │
│  │                                                             │ │
│  │ /plugin install code-review@anthropic-official             │ │
│  │                                                [Copy]       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  [Cancel]                              [Install Plugin →]        │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 4. Security & Trust Model

#### Trust Levels

```typescript
// src/shared/types/plugin-security.types.ts

export type PluginTrustLevel =
  | 'official'      // Anthropic official plugins
  | 'verified'      // Third-party verified publishers
  | 'community'     // Community-submitted
  | 'unknown';      // Unverified source

export interface PluginSecurityContext {
  trustLevel: PluginTrustLevel;
  publisher: {
    name: string;
    verified: boolean;
    url?: string;
  };
  repository: {
    url: string;
    stars?: number;
    lastUpdate?: string;
    license?: string;
  };
  riskFactors: PluginRiskFactor[];
  permissions: PluginPermission[];
}

export interface PluginRiskFactor {
  severity: 'info' | 'warning' | 'critical';
  title: string;
  description: string;
}

export interface PluginPermission {
  type: 'filesystem' | 'network' | 'shell' | 'mcp';
  description: string;
  scope?: string;
}
```

#### Security Warning UI

```
┌──────────────────────────────────────────────────────────────────┐
│  ⚠️ Security Review: community-plugin                       [×]   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Trust Level: COMMUNITY                                          │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ ⚠️ Unverified Publisher                                     │ │
│  │ This plugin is not from a verified publisher. The code has │ │
│  │ not been reviewed by Anthropic.                            │ │
│  │                                                             │ │
│  │ Recommendation: Review the source code before installing.  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ ⚠️ Shell Command Execution                                  │ │
│  │ This plugin includes hooks that can execute shell commands │ │
│  │ on your system.                                            │ │
│  │                                                             │ │
│  │ Hooks found:                                                │ │
│  │ • PreToolUse: Runs validation script                       │ │
│  │ • PostToolUse: Logs tool usage to file                     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ ℹ️ MCP Server Integration                                   │ │
│  │ This plugin configures an MCP server that will have access │ │
│  │ to Claude Code's context.                                  │ │
│  │                                                             │ │
│  │ Server: custom-analytics-server                            │ │
│  │ Command: node /path/to/server.js                           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ☐ I understand the risks and want to proceed                    │
│                                                                   │
│  [Cancel]  [View Source Code →]  [Install Anyway]                │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 5. CLI Integration

#### Command Mapping

| UI Action | CLI Command |
|-----------|-------------|
| Install plugin | `/plugin install plugin@marketplace` |
| Uninstall plugin | `/plugin uninstall plugin@marketplace` |
| Enable plugin | `/plugin enable plugin@marketplace` |
| Disable plugin | `/plugin disable plugin@marketplace` |
| Add marketplace | `/plugin marketplace add <source>` |
| Remove marketplace | `/plugin marketplace remove <name>` |
| List marketplaces | `/plugin marketplace list` |
| Update marketplace | `/plugin marketplace update <name>` |

#### Service Implementation

```typescript
// src/main/services/PluginsService.ts (refactored)

export class PluginsService {
  private claudeService: ClaudeService;

  constructor(claudeService: ClaudeService) {
    this.claudeService = claudeService;
  }

  /**
   * Install a plugin using Claude Code CLI
   */
  async installPlugin(
    pluginName: string,
    marketplaceName: string
  ): Promise<PluginInstallResult> {
    console.log('[PluginsService] Installing plugin:', { pluginName, marketplaceName });

    try {
      // Build and execute CLI command
      const command = `claude /plugin install ${this.escapeArg(pluginName)}@${this.escapeArg(marketplaceName)}`;
      const { stdout, stderr } = await this.claudeService.executeCommand(command);

      // Parse result
      if (stderr && !stdout.includes('Successfully installed')) {
        return {
          success: false,
          error: stderr
        };
      }

      console.log('[PluginsService] Plugin installed successfully:', pluginName);
      return { success: true };
    } catch (error) {
      console.error('[PluginsService] Installation failed:', error);
      return {
        success: false,
        error: error instanceof Error ? error.message : 'Installation failed'
      };
    }
  }

  /**
   * Add a marketplace using Claude Code CLI
   */
  async addMarketplace(source: string): Promise<{ success: boolean; error?: string }> {
    console.log('[PluginsService] Adding marketplace:', source);

    try {
      const command = `claude /plugin marketplace add ${this.escapeArg(source)}`;
      await this.claudeService.executeCommand(command);

      console.log('[PluginsService] Marketplace added successfully');
      return { success: true };
    } catch (error) {
      console.error('[PluginsService] Failed to add marketplace:', error);
      return {
        success: false,
        error: error instanceof Error ? error.message : 'Failed to add marketplace'
      };
    }
  }

  /**
   * Fetch marketplace manifest for browsing (read-only operation)
   * This is the only direct file/network access we do
   */
  async fetchMarketplaceManifest(source: string): Promise<MarketplaceManifest | null> {
    // Keep existing implementation for browsing
    // This doesn't modify any files, just fetches for display
  }

  /**
   * Analyze plugin security before installation
   */
  async analyzePluginSecurity(
    pluginPath: string
  ): Promise<PluginSecurityContext> {
    const context: PluginSecurityContext = {
      trustLevel: 'unknown',
      publisher: { name: 'Unknown', verified: false },
      repository: { url: '' },
      riskFactors: [],
      permissions: []
    };

    // Check for official Anthropic plugin
    if (pluginPath.includes('anthropics/claude-code/plugins')) {
      context.trustLevel = 'official';
      context.publisher = { name: 'Anthropic', verified: true };
    }

    // Analyze plugin components for risks
    const manifest = await this.readPluginMetadata(pluginPath);

    // Check for hooks (shell execution capability)
    if (manifest.hooks) {
      context.permissions.push({
        type: 'shell',
        description: 'Can execute shell commands via hooks'
      });
      context.riskFactors.push({
        severity: 'warning',
        title: 'Shell Command Execution',
        description: 'This plugin includes hooks that can execute shell commands'
      });
    }

    // Check for MCP servers
    if (manifest.mcpServers) {
      context.permissions.push({
        type: 'mcp',
        description: 'Configures MCP server integration'
      });
    }

    return context;
  }

  private escapeArg(arg: string): string {
    // Escape shell special characters
    if (arg.includes(' ') || arg.includes('"') || arg.includes("'")) {
      return `"${arg.replace(/"/g, '\\"')}"`;
    }
    return arg;
  }
}
```

### 6. UI Changes

#### Tab Structure Update

```
┌──────────────────────────────────────────────────────────────────┐
│  Plugins                                                          │
├──────────────────────────────────────────────────────────────────┤
│  [Official] [Browse Marketplace] [Installed] [Marketplaces]      │
│                                                                   │
│  ─────────────────────────────────────────────────────────────── │
│                                                                   │
│  Tab Content Here                                                │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**Tab Changes:**
1. **NEW: Official** - Dedicated tab for Anthropic official plugins (featured prominently)
2. **Browse Marketplace** - Same as before, but with trust badges
3. **Installed** - Same as before, shows actual Claude Code plugin state
4. **Marketplaces** - Same as before, but Anthropic marketplace pre-configured

#### Pre-configured Anthropic Marketplace

On first launch, automatically configure the official Anthropic marketplace:

```typescript
// Auto-configure on startup
const ANTHROPIC_MARKETPLACE = {
  name: 'anthropic-official',
  source: 'github:anthropics/claude-code/plugins',
  description: 'Official plugins from the Claude Code team',
  verified: true
};
```

### 7. Data Types Updates

```typescript
// src/shared/types/plugin.types.ts (additions)

/**
 * Plugin with security context
 */
export interface PluginWithSecurity extends MarketplacePlugin {
  security: PluginSecurityContext;
}

/**
 * Official Anthropic plugin metadata
 */
export interface OfficialPlugin extends MarketplacePlugin {
  isOfficial: true;
  category: OfficialPluginCategory;
  featured?: boolean;
}

export type OfficialPluginCategory =
  | 'code-review'
  | 'development'
  | 'security'
  | 'productivity'
  | 'learning';

/**
 * Plugin installation options
 */
export interface PluginInstallOptions {
  pluginName: string;
  marketplaceName: string;
  scope: 'user' | 'project';
  projectPath?: string;
  skipSecurityCheck?: boolean; // For official plugins
}
```

### 8. Implementation Plan

#### Phase 1: CLI Integration (Week 1)

**Backend:**
- [ ] Refactor `PluginsService` to delegate to CLI commands
- [ ] Update `ClaudeService` with plugin command execution
- [ ] Remove direct file manipulation for install/uninstall/toggle
- [ ] Keep marketplace manifest fetching (read-only)

**Testing:**
- [ ] Test CLI command construction
- [ ] Test error handling for CLI failures
- [ ] Verify plugin state syncs with Claude Code

#### Phase 2: Official Anthropic Hub (Week 1-2)

**Backend:**
- [ ] Pre-configure Anthropic marketplace on startup
- [ ] Fetch and parse all 13 official plugins
- [ ] Calculate security context for each

**Frontend:**
- [ ] Add "Official" tab to PluginsManager
- [ ] Create `OfficialPluginsHub` component
- [ ] Feature top 3-5 plugins with rich cards
- [ ] Add "Verified" badges to official plugins

#### Phase 3: Security & Verification (Week 2)

**Backend:**
- [ ] Implement `analyzePluginSecurity()` method
- [ ] Detect hooks, MCP servers, filesystem access
- [ ] Calculate trust level based on source

**Frontend:**
- [ ] Create `PluginSecurityDialog` component
- [ ] Add security warnings for community plugins
- [ ] Show component breakdown before install
- [ ] Add "I understand the risks" checkbox

#### Phase 4: Pre-Installation Verification (Week 2-3)

**Frontend:**
- [ ] Create `InstallPluginDialog` component
- [ ] Show all components to be installed
- [ ] Preview CLI command
- [ ] Add scope selection (user/project)

**UX:**
- [ ] Loading states during installation
- [ ] Success/error notifications
- [ ] Refresh plugin list after install

#### Phase 5: Polish & Documentation (Week 3)

- [ ] Remove "Under Development" badge
- [ ] Update FEATURES.md
- [ ] Add plugin usage documentation
- [ ] Write user guide for security considerations
- [ ] E2E testing for full plugin workflow

### 9. IPC Channels Update

```typescript
// src/shared/types/ipc.plugins.types.ts

export const PLUGINS_CHANNELS = {
  // Existing (keep)
  GET_MARKETPLACES: 'plugins:get-marketplaces',
  GET_AVAILABLE: 'plugins:get-available',
  GET_INSTALLED: 'plugins:get-installed',
  GET_GITHUB_INFO: 'plugins:get-github-info',
  GET_HEALTH: 'plugins:get-health',

  // Refactored (use CLI)
  ADD_MARKETPLACE: 'plugins:add-marketplace',      // -> /plugin marketplace add
  REMOVE_MARKETPLACE: 'plugins:remove-marketplace', // -> /plugin marketplace remove
  INSTALL_PLUGIN: 'plugins:install',                // -> /plugin install
  UNINSTALL_PLUGIN: 'plugins:uninstall',            // -> /plugin uninstall
  TOGGLE_PLUGIN: 'plugins:toggle',                  // -> /plugin enable|disable

  // New
  GET_OFFICIAL_PLUGINS: 'plugins:get-official',
  ANALYZE_SECURITY: 'plugins:analyze-security',
  GET_PLUGIN_COMPONENTS: 'plugins:get-components',
} as const;
```

### 10. Migration Strategy

#### Breaking Changes

1. **Remove custom file management** - No longer write to `~/.claude/plugins/`
2. **Sync with Claude Code** - Plugin state now comes from Claude Code, not our files

#### Migration Steps

1. **Detect existing Open Owl plugin data** in `~/.claude/plugins/`
2. **Check if plugins exist in Claude Code** via CLI
3. **Offer to re-install** plugins that are in our database but not in Claude Code
4. **Clean up** our custom `installed_plugins.json` after migration

```typescript
async migratePlugins(): Promise<void> {
  console.log('[PluginsService] Checking for plugin migration...');

  // Read our old plugin database
  const oldPlugins = await this.readOldInstalledPlugins();
  if (oldPlugins.length === 0) return;

  // Get current Claude Code plugin state
  const currentPlugins = await this.getClaudeCodePlugins();

  // Find plugins to migrate
  const toMigrate = oldPlugins.filter(
    old => !currentPlugins.some(cur => cur.name === old.name)
  );

  if (toMigrate.length > 0) {
    console.log('[PluginsService] Found plugins to migrate:', toMigrate.map(p => p.name));
    // Show migration dialog to user
  }
}
```

---

## Security Considerations

### Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| Malicious plugins | High | Trust levels, security warnings, source verification |
| Shell injection in hooks | High | Show hook commands before install, require acknowledgment |
| Unauthorized file access | Medium | Show filesystem permissions, scope warnings |
| Supply chain attacks | Medium | Verify publisher, check repository health |
| MCP server risks | Medium | Show MCP config, explain access scope |

### Security Best Practices

1. **Official plugins first:** Always recommend official Anthropic plugins
2. **Trust verification:** Show publisher verification status prominently
3. **Permission transparency:** List all plugin capabilities before install
4. **Acknowledgment required:** Community plugins require explicit acceptance
5. **Source visibility:** Always show where plugin code comes from

---

## Success Criteria

### Must Have (Launch Blockers)

- [ ] All plugin operations delegate to Claude Code CLI
- [ ] Official Anthropic marketplace pre-configured
- [ ] Security warnings for community plugins
- [ ] Pre-installation component preview
- [ ] Trust level badges on all plugins

### Should Have (Post-Launch)

- [ ] Plugin update notifications
- [ ] Bulk enable/disable
- [ ] Plugin search across all marketplaces
- [ ] Installation history

### Nice to Have (Future)

- [ ] Plugin recommendations based on usage
- [ ] Community ratings/reviews
- [ ] Plugin dependency management
- [ ] Project-level plugin configuration UI

---

## Risks & Mitigations

### Risk 1: CLI Command Format Changes

**Impact:** High - Plugin operations would break
**Mitigation:**
- Version check for Claude Code
- Graceful fallback with error messages
- Documentation of required Claude Code version

### Risk 2: Slow CLI Operations

**Impact:** Medium - Poor UX during install
**Mitigation:**
- Loading states with progress
- Allow cancellation
- Background installation with notification

### Risk 3: Security Perception

**Impact:** Medium - Users may distrust non-official plugins
**Mitigation:**
- Clear distinction between official and community
- Detailed security analysis
- Easy-to-understand risk explanations

---

## Appendix A: Official Anthropic Plugins Reference

```json
{
  "marketplace": {
    "name": "anthropic-official",
    "source": "github:anthropics/claude-code/plugins",
    "description": "Official plugins from the Claude Code team"
  },
  "plugins": [
    {
      "name": "agent-sdk-dev",
      "description": "Development kit for Claude Agent SDK projects with interactive setup and validation agents",
      "category": "development",
      "components": { "agents": 2, "skills": 3 }
    },
    {
      "name": "claude-opus-4-5-migration",
      "description": "Automated migration tool for updating code to Opus 4.5",
      "category": "productivity",
      "components": { "commands": 1 }
    },
    {
      "name": "code-review",
      "description": "Automated PR code review using multiple specialized agents with confidence-based scoring",
      "category": "code-review",
      "featured": true,
      "components": { "agents": 4, "skills": 2 }
    },
    {
      "name": "commit-commands",
      "description": "Git workflow automation for commits, pushes, and pull requests",
      "category": "productivity",
      "components": { "commands": 3 }
    },
    {
      "name": "explanatory-output-style",
      "description": "Educational insights about implementation choices and codebase patterns",
      "category": "learning",
      "components": { "skills": 1 }
    },
    {
      "name": "feature-dev",
      "description": "Comprehensive feature development workflow with a structured 7-phase approach",
      "category": "development",
      "featured": true,
      "components": { "agents": 3, "commands": 5 }
    },
    {
      "name": "frontend-design",
      "description": "Production-grade interface creation avoiding generic AI aesthetics",
      "category": "development",
      "components": { "agents": 2, "skills": 4 }
    },
    {
      "name": "hookify",
      "description": "Custom hook creation to prevent unwanted behaviors through pattern analysis",
      "category": "security",
      "components": { "hooks": 1, "commands": 1 }
    },
    {
      "name": "learning-output-style",
      "description": "Interactive mode requesting meaningful code contributions at decision points",
      "category": "learning",
      "components": { "skills": 1 }
    },
    {
      "name": "plugin-dev",
      "description": "Comprehensive toolkit for developing Claude Code plugins with 7 expert skills",
      "category": "development",
      "components": { "skills": 7, "commands": 2 }
    },
    {
      "name": "pr-review-toolkit",
      "description": "Specialized agents for PR analysis across multiple dimensions",
      "category": "code-review",
      "components": { "agents": 5 }
    },
    {
      "name": "ralph-wiggum",
      "description": "Self-referential AI loops for iterative autonomous development",
      "category": "development",
      "components": { "agents": 1 }
    },
    {
      "name": "security-guidance",
      "description": "Security reminder hook monitoring for 9 dangerous patterns",
      "category": "security",
      "featured": true,
      "components": { "hooks": 1 }
    }
  ]
}
```

---

## Appendix B: CLI Command Reference

```bash
# Marketplace operations
/plugin marketplace add owner/repo           # GitHub shorthand
/plugin marketplace add https://url.com      # Direct URL
/plugin marketplace add ./local/path         # Local path
/plugin marketplace list                     # List all marketplaces
/plugin marketplace update <name>            # Refresh marketplace
/plugin marketplace remove <name>            # Remove marketplace

# Plugin operations
/plugin install plugin-name@marketplace      # Install plugin
/plugin uninstall plugin-name@marketplace    # Remove plugin
/plugin enable plugin-name@marketplace       # Enable plugin
/plugin disable plugin-name@marketplace      # Disable plugin

# Interactive mode
/plugin                                      # Open plugin browser
```

---

## References

1. **Claude Code Plugins Documentation:** https://code.claude.com/docs/en/plugins
2. **Plugin Marketplaces Guide:** https://code.claude.com/docs/en/plugin-marketplaces
3. **Plugins Reference:** https://code.claude.com/docs/en/plugins-reference
4. **Official Anthropic Plugins:** https://github.com/anthropics/claude-code/tree/main/plugins
5. **ADR-004 MCP Manager:** See `project-docs/adr/adr-004-mcp-manager.md`

---

## Conclusion

This ADR transforms the Plugins Manager from an experimental feature to a production-ready tool by:

1. **Adopting CLI delegation** - Consistent with MCP Manager, reliable, future-proof
2. **Prioritizing official plugins** - Anthropic plugins featured prominently
3. **Emphasizing security** - Clear trust levels, permission explanations, risk warnings
4. **Verifying before installing** - Users know exactly what they're getting

**Expected Impact:**
- 90% of users start with official Anthropic plugins
- 50% reduction in plugin-related support issues
- Zero security incidents from plugin installation

The implementation timeline is approximately 3 weeks, with the feature ready for production in the next minor release.

---

**Questions or Feedback?**
Please comment on this ADR document or open a GitHub discussion.



# Open Owl 🦉

**Stop editing JSON and MD files. Manage your Claude Code setup visually.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Node Version](https://img.shields.io/badge/node-%3E%3D18.0.0-brightgreen.svg)](https://nodejs.org/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![Beta](https://img.shields.io/badge/status-Beta-blue.svg)](#)

---

## The Problem

Claude Code is powerful but configuration is painful:
- Hand-editing JSON for permissions rules? Error-prone.
- Managing MCP servers across projects? Copy-paste hell.
- Finding that one subagent you created? Good luck grep-ing through `.claude/`.
- You want to know any metrics about your Claude Code usage? You need to download yet another tool.

## The Solution

Open Owl gives you a visual interface for everything Claude Code configuration:

![Open Owl Dashboard](screenshots/open-owl-dashboard.png)

**No more terminal commands. No more JSON typos. Just point, click, done.**

## What You Can Do

### Core Features
- **Settings Editor** - User and project-level settings with search and validation
- **Permission Rules** - Visual builder with 6 security templates
- **MCP Servers** - Add, configure, and test Model Context Protocol integrations
- **Subagents** - Create custom agents with tool restrictions and model selection
- **Skills & Commands** - Manage skills and import slash commands from GitHub repos
- **Plugins** - Browse marketplaces, install plugins, enable/disable without editing files

### Power Features
- **Usage Metrics** - Interactive charts showing spend, tokens, and model breakdown
- **Status Line Templates** - 10+ pre-built templates with live preview
- **Dashboard** - Real-time Claude Code detection and API status monitoring
- **Debug Logs** - View and search Claude Code logs without terminal or importing the whole `.claude` directory into an IDE.

[See all features with screenshots →](https://antonbelev.github.io/open-owl/screenshots.html)

## Installation

### Download (Recommended)

**[📥 Download Latest Release](https://github.com/antonbelev/open-owl/releases/latest)**

- macOS (Intel + Apple Silicon)
- Windows (x64 + ARM64)

[Full installation guide →](https://antonbelev.github.io/open-owl/installation.html)

### Build from Source

```bash
git clone https://github.com/antonbelev/open-owl.git
cd open-owl
npm install
npm run dev:electron
```

**Requirements:** Node.js 18+, npm 9+, Claude Code CLI installed

## FAQ

**Does this replace Claude Code?**
No. Open Owl is a UI layer on top of Claude Code CLI. You still need Claude Code installed.

**Is my data safe?**
100% local. No telemetry, no external servers, no data collection. Everything stays on your machine.

**What platforms work?**
macOS (Intel + Apple Silicon) and Windows (x64 + ARM64). Linux support coming soon.

## Contributing

PRs welcome! Check [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

Quick start:
```bash
git clone https://github.com/antonbelev/open-owl.git
cd open-owl
npm install
npm run dev:electron
```

## Tech Stack

Electron + React 18 + TypeScript + Vite + Zustand + Tailwind CSS

See [CLAUDE.md](CLAUDE.md) for development notes and [docs/architecture.md](docs/architecture.md) for system design.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=antonbelev/open-owl&type=Date)](https://star-history.com/#antonbelev/open-owl&Date)

## License

Open Owl is open-source software licensed under the [MIT License](LICENSE).

---

**Disclaimer:** Open Owl is not affiliated with Anthropic. Claude is a trademark of Anthropic PCB.

**Support:** [Issues](https://github.com/antonbelev/open-owl/issues) • [Discussions](https://github.com/antonbelev/open-owl/discussions) • [Documentation](https://antonbelev.github.io/open-owl/)

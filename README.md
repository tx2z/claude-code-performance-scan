# Claude Code Performance Scan

A comprehensive performance analysis command for [Claude Code](https://claude.ai/claude-code) that detects performance bottlenecks and optimization opportunities using specialized AI agents.

## Features

- **Algorithm Complexity Analysis** - Detects O(n^2) or worse patterns, missing memoization
- **Memory Leak Detection** - Finds memory leaks, unbounded caches, closure retention
- **Database Optimization** - Identifies N+1 queries, missing indexes, unbounded queries
- **Frontend Performance** - Analyzes render blocking, bundle sizes, Web Vitals issues
- **Network Efficiency** - Checks compression, caching, request optimization
- **Async Pattern Analysis** - Finds sequential async that could be parallel
- **Build/Bundle Analysis** - Detects large dependencies, tree shaking issues

## Requirements

- [Claude Code CLI](https://claude.ai/claude-code) installed and configured
- A project to analyze

## Installation

1. Clone or download this repository
2. Copy the folders to your project's `.claude/` directory:

```bash
# From your project root
cp -r path/to/claude-code-performance-scan/commands .claude/
cp -r path/to/claude-code-performance-scan/performance .claude/
```

Your project structure should look like:
```
your-project/
├── .claude/
│   ├── commands/
│   │   └── performance-scan.md
│   └── performance/
│       ├── agents/
│       │   ├── algorithm-complexity.md
│       │   ├── memory.md
│       │   ├── database.md
│       │   ├── frontend.md
│       │   ├── network.md
│       │   ├── async-operations.md
│       │   └── build-bundle.md
│       └── templates/
│           └── performance-report.md
├── src/
└── ...
```

3. (Optional) Add `performance-reports/` to your `.gitignore`:

```bash
echo "performance-reports/" >> .gitignore
```

## Optional: Optimize for Your Tech Stack

After installation, you can optimize the performance scanner for your specific codebase. This improves detection accuracy by focusing on performance patterns relevant to your frameworks.

Run this prompt in Claude Code:

```
I just installed the performance-scan command in .claude/. Please:

1. Analyze my codebase to detect my tech stack (frontend framework, backend framework, database, build tools)
2. Read the command files in .claude/commands/performance-scan.md and .claude/performance/agents/
3. Optimize each performance agent by:
   - Removing checks for technologies I don't use
   - Adding performance patterns specific to my frameworks (React re-renders, Django ORM, etc.)
   - Configuring database optimization checks for my ORM
   - Adjusting bundle analysis based on my build tool (Webpack/Vite/esbuild)
4. Keep the agent structure, performance categories, and output format unchanged

Show me what you'll change before applying.
```

## Usage

In Claude Code, run the performance scan command:

```
/performance-scan
```

### Scan Modes

| Command | Description |
|---------|-------------|
| `/performance-scan` | Full scan (all performance checks) |
| `/performance-scan quick` | Critical issues only (faster) |
| `/performance-scan frontend` | Frontend performance only |
| `/performance-scan backend` | Backend/API performance only |
| `/performance-scan database` | Database optimization only |
| `/performance-scan memory` | Memory analysis only |
| `/performance-scan bundle` | Build/bundle analysis only |
| `/performance-scan runtime` | Runtime performance (algorithm + async) |
| `/performance-scan network` | Network efficiency only |

## Performance Categories

### Core Analysis Agents

| ID | Name | Focus Areas |
|----|------|-------------|
| PERF01 | Algorithm Complexity | O(n^2) loops, missing memoization, inefficient data structures |
| PERF02 | Memory | Leaks, unbounded caches, closure retention, cleanup issues |
| PERF03 | Database | N+1 queries, missing indexes, query optimization |
| PERF04 | Frontend | Web Vitals, bundle size, render blocking, lazy loading |
| PERF05 | Network | Compression, caching, request batching, CDN usage |
| PERF06 | Async Operations | Promise.all, debounce/throttle, request batching |
| PERF07 | Build/Bundle | Dependencies, tree shaking, minification, source maps |

## How It Works

1. **Tech Stack Detection** - Automatically detects your frameworks, build tools, and database
2. **Agent Execution** - Spawns specialized agents for each performance domain
3. **Finding Collection** - Aggregates findings with impact scores
4. **Report Generation** - Creates a markdown report in `performance-reports/`
5. **Optimization Mode** - Optionally applies recommended optimizations

## Output

Reports are saved to `performance-reports/YYYY-MM-DD-HHmm-scan.md` with:

- Performance Score (0-100)
- Critical Bottlenecks (immediate action needed)
- Quick Wins (low effort, high impact)
- Major Optimizations (significant improvements)
- Estimated Impact (performance gain predictions)
- Benchmark Suggestions
- Monitoring Recommendations

## Supported Tech Stacks

The scanner auto-detects and adapts to:

**Frontend:**
- React, Vue, Angular, Svelte, Next.js, Nuxt.js
- Webpack, Vite, Rollup, esbuild, Parcel

**Backend:**
- Node.js, Express, NestJS, Fastify
- Python, Django, FastAPI, Flask
- PHP, Laravel, Symfony
- .NET Core, ASP.NET
- Go, Gin, Echo
- Java, Spring Boot

**Databases:**
- PostgreSQL, MySQL, MongoDB
- TypeORM, Prisma, Sequelize, Mongoose
- SQLAlchemy, Django ORM
- Entity Framework, GORM

## Customization

To adapt for your specific needs:

1. **Tech stack detection** - Modify Step 1 in `performance-scan.md`
2. **Agent behavior** - Edit individual agents in `performance/agents/`
3. **Report format** - Modify `performance/templates/performance-report.md`
4. **Severity thresholds** - Adjust impact levels in agent files

## Impact Classification

| Level | Description | Action Timeline |
|-------|-------------|-----------------|
| Critical | Severe performance degradation | Immediate |
| High | Significant slowdown | This sprint |
| Medium | Noticeable impact | Within 30 days |
| Low | Minor optimization | When convenient |

## Contributing

Contributions welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

## License

MIT License - see [LICENSE](LICENSE) file.

## Disclaimer

This tool performs static analysis and pattern matching. It may produce false positives and cannot guarantee detection of all performance issues. Always validate findings with actual profiling and benchmarking for production systems.

## References

- [Google Web Vitals](https://web.dev/vitals/)
- [Node.js Performance Best Practices](https://nodejs.org/en/docs/guides/dont-block-the-event-loop/)
- [React Performance Optimization](https://react.dev/learn/render-and-commit)
- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code)

# Azure Durable Documentation

Comprehensive documentation for Azure Durable technologies:

- **Azure Durable Functions** - Serverless durable orchestrations
- **Durable Task Scheduler** - Managed orchestration backend
- **Durable Task SDKs** - Portable SDKs for any compute platform

## Local Development

### Prerequisites

- Ruby 3.1 or later
- Bundler gem

### Setup

```bash
# Install dependencies
bundle install

# Serve locally
bundle exec jekyll serve
```

Visit `http://localhost:4000` to view the site.

### Build

```bash
bundle exec jekyll build
```

Output is generated in the `_site` directory.

## Deployment

This site is automatically deployed to GitHub Pages when changes are pushed to the `main` branch. See `.github/workflows/pages.yml` for the deployment configuration.

## Documentation Structure

```
docs/
├── concepts/           # Core concepts (orchestrators, activities, entities)
├── durable-functions/  # Azure Durable Functions guide
├── durable-task-scheduler/  # DTS setup, dashboard, identity
├── sdks/              # .NET, Python, Java SDK references
├── patterns/          # Common orchestration patterns
├── architecture/      # Deployment architecture guides
├── comparison/        # Decision guides and advantages
└── samples/           # Code samples and examples
```

## License

This documentation is provided under the MIT License.

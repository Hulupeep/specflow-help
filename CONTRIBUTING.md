# Contributing to Specflow Documentation

Thank you for your interest in improving the Specflow documentation!

## How to Contribute

### Content Updates

1. **Find or create an issue** describing what needs updating
2. **Fork the repository** and create a branch
3. **Make your changes** following the style guide below
4. **Test locally** (see "Running Locally" below)
5. **Submit a pull request** with a clear description

### Style Guide

#### Page Frontmatter
Every page must have:
```yaml
---
layout: default
title: Page Title
nav_order: 1
permalink: /path/
---
```

#### Headings
- Use `# Main Title` for page title
- Use `## Section` for sections
- Use `### Subsection` for subsections

#### Code Blocks
Always specify the language:
````markdown
```yaml
contract_type: feature
```
````

#### Links
- Internal links: `[Text](/path/)`
- External links: `[Text](https://example.com)`

#### Compiler Analogy
Include the compiler analogy on major landing pages:
> TypeScript rejects type errors. Specflow rejects architecture errors.

### Running Locally

```bash
# Install dependencies
bundle install

# Serve the site
bundle exec jekyll serve

# View at http://localhost:4000
```

### Content Guidelines

1. **Lead with value** — Answer "why should I care?" first
2. **Use examples** — Show, don't just tell
3. **Test code snippets** — All code must be copy-paste-runnable
4. **Keep it concise** — Developers skim documentation

### Areas Needing Help

Check [open issues](https://github.com/Hulupeep/specflow-help/issues) for:
- Missing documentation
- Outdated examples
- Unclear explanations
- Broken links

## Questions?

Open a [Discussion](https://github.com/Hulupeep/specflow-help/discussions) or comment on an existing issue.

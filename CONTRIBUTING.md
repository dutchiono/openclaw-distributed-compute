# Contributing to OpenClaw Distributed Compute

Thank you for your interest in contributing to OpenClaw Distributed Compute! This project aims to democratize access to supercomputer-scale resources by harnessing idle OpenClaw devices worldwide.

---

## How to Contribute

### Areas of Contribution

We welcome contributions in several areas:

#### 1. Protocol Development
- Distributed systems architecture
- Work distribution and scheduling algorithms
- Fault tolerance and retry mechanisms
- Network optimization and bandwidth efficiency

#### 2. Device Agents
- macOS client software (primary platform)
- Linux and Windows compatibility
- Resource monitoring and management
- Security and sandboxing

#### 3. Scheduling & Optimization
- Heterogeneous hardware scheduling (M1-M4 Max)
- Load balancing across the network
- Priority queuing and fair-share policies
- Performance profiling and optimization

#### 4. Scientific Applications
- Domain-specific integrations (ML, genomics, rendering, etc.)
- Framework adapters (PyTorch, TensorFlow, JAX)
- Benchmarking and validation
- Use case documentation

#### 5. Documentation
- User guides and tutorials
- API documentation
- Architecture diagrams
- Best practices and patterns

---

## Getting Started

### Prerequisites

- **For Protocol/Backend Development:**
  - Python 3.10+ or Go 1.21+
  - Docker and Kubernetes (for testing)
  - Familiarity with distributed systems concepts

- **For Device Agent Development:**
  - macOS 13+ with Apple Silicon (M1/M2/M3/M4)
  - Swift 5.9+ or Python 3.10+
  - Xcode 15+ (for native macOS development)

- **For Scientific Applications:**
  - Experience with ML frameworks or scientific computing
  - Understanding of parallel and distributed algorithms

### Development Setup

```bash
# Clone the repository
git clone https://github.com/dutchiono/openclaw-distributed-compute.git
cd openclaw-distributed-compute

# Install dependencies (coming soon)
# make install

# Run tests (coming soon)
# make test
```

---

## Contribution Workflow

### 1. Fork the Repository

Click the "Fork" button on GitHub to create your own copy of the repository.

### 2. Create a Feature Branch

```bash
git checkout -b feature/your-feature-name
```

Use descriptive branch names:
- `feature/add-heterogeneous-scheduling`
- `bugfix/fix-connection-timeout`
- `docs/update-api-guide`

### 3. Make Your Changes

- Write clean, well-documented code
- Follow existing code style and conventions
- Add tests for new functionality
- Update documentation as needed

### 4. Commit Your Changes

```bash
git add .
git commit -m "feat: add heterogeneous hardware scheduling"
```

Use [Conventional Commits](https://www.conventionalcommits.org/) format:
- `feat:` - New features
- `fix:` - Bug fixes
- `docs:` - Documentation changes
- `test:` - Test additions or changes
- `refactor:` - Code refactoring
- `perf:` - Performance improvements

### 5. Push and Create a Pull Request

```bash
git push origin feature/your-feature-name
```

Then create a Pull Request on GitHub with:
- Clear description of changes
- Reference to related issues (if any)
- Screenshots or demos (if applicable)

---

## Code Style Guidelines

### Python
- Follow [PEP 8](https://pep8.org/) style guide
- Use type hints for function signatures
- Maximum line length: 100 characters
- Use `black` for formatting
- Use `ruff` for linting

### Swift
- Follow [Swift API Design Guidelines](https://swift.org/documentation/api-design-guidelines/)
- Use SwiftLint for consistency
- Prefer value types and immutability

### Go
- Follow standard Go conventions
- Use `gofmt` for formatting
- Run `golangci-lint` before committing

---

## Testing

### Unit Tests
- Write unit tests for all new functionality
- Aim for 80%+ code coverage
- Use mocks and fixtures for external dependencies

### Integration Tests
- Test distributed scenarios with multiple nodes
- Validate fault tolerance and recovery
- Test network partition scenarios

### Performance Tests
- Benchmark scheduling algorithms
- Measure network bandwidth efficiency
- Profile resource usage

---

## Documentation

- Update README.md for user-facing changes
- Add inline code comments for complex logic
- Create architectural decision records (ADRs) for major design choices
- Update API documentation for interface changes

---

## Community Guidelines

### Code of Conduct

We are committed to providing a welcoming and inclusive environment. Please:
- Be respectful and professional
- Provide constructive feedback
- Focus on what is best for the community
- Show empathy towards other contributors

### Communication

- **GitHub Issues:** Bug reports and feature requests
- **GitHub Discussions:** General questions and ideas
- **Pull Requests:** Code review and technical discussion
- **Discord (coming soon):** Real-time chat and collaboration

---

## Issue Reporting

### Bug Reports

When reporting bugs, please include:
- Clear description of the issue
- Steps to reproduce
- Expected vs. actual behavior
- Environment details (OS, hardware, versions)
- Relevant logs or error messages

### Feature Requests

When requesting features, please include:
- Clear description of the proposed feature
- Use cases and benefits
- Potential implementation approaches
- Any relevant examples or references

---

## Recognition

Contributors will be recognized in:
- GitHub contributors page
- Project README (for significant contributions)
- Release notes
- Annual contributor highlights

---

## License

By contributing to OpenClaw Distributed Compute, you agree that your contributions will be licensed under the MIT License.

---

## Questions?

Feel free to:
- Open a [GitHub Discussion](https://github.com/dutchiono/openclaw-distributed-compute/discussions)
- Ask in our Discord (coming soon)
- Reach out to maintainers

Thank you for helping build the future of distributed computing!
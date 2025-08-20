# Pre-commit Hooks Security Guide

## Overview

Pre-commit hooks are automated security and code quality checks that run before each git commit. This comprehensive guide covers the implementation, configuration, and best practices for using pre-commit hooks in the AIOutlet project to prevent security vulnerabilities and maintain code quality.

## Table of Contents

1. [What are Pre-commit Hooks?](#what-are-pre-commit-hooks)
2. [Why Use Pre-commit Hooks?](#why-use-pre-commit-hooks)
3. [Security Benefits](#security-benefits)
4. [Installation and Setup](#installation-and-setup)
5. [Configuration Details](#configuration-details)
6. [Security Tools Integration](#security-tools-integration)
7. [Code Quality Tools](#code-quality-tools)
8. [Common Issues and Solutions](#common-issues-and-solutions)
9. [Best Practices](#best-practices)
10. [Testing Pre-commit Hooks](#testing-pre-commit-hooks)
11. [Troubleshooting](#troubleshooting)

## What are Pre-commit Hooks?

Pre-commit hooks are scripts that run automatically before a git commit is finalized. They act as a safety net, preventing insecure or low-quality code from being committed to the repository. If any hook fails, the commit is blocked until the issues are resolved.

### Key Characteristics

- **Automatic execution**: Run on every `git commit` attempt
- **Fail-safe mechanism**: Block commits when security/quality issues are detected
- **Multi-tool integration**: Support for various security and linting tools
- **Cross-platform compatibility**: Work on Windows, Linux, and macOS

## Why Use Pre-commit Hooks?

### Security Protection

- **Prevent secret leaks**: Detect API keys, passwords, tokens before they reach the repository
- **Vulnerability scanning**: Identify security issues in code before deployment
- **Dependency security**: Check for known vulnerabilities in project dependencies

### Code Quality Assurance

- **Consistent formatting**: Automatic code formatting with tools like Black
- **Import organization**: Sort and organize imports with isort
- **Linting**: Enforce coding standards with flake8
- **Syntax validation**: Check YAML, JSON, XML files for syntax errors

### Development Efficiency

- **Early detection**: Catch issues before they reach CI/CD pipelines
- **Reduced review time**: Automated checks reduce manual code review burden
- **Team consistency**: Enforce consistent standards across all developers

## Security Benefits

### Multi-layered Security Approach

Our pre-commit configuration implements multiple security layers:

1. **Secret Detection (`detect-secrets`)**
   - Scans for API keys, passwords, tokens
   - Uses entropy analysis and pattern matching
   - Maintains a baseline of known false positives

2. **Security Analysis (`bandit`)**
   - Python-specific security issue detection
   - Identifies hardcoded passwords, SQL injection risks
   - Common vulnerability patterns (CWE database)

3. **Dependency Vulnerability Scanning (`safety`)**
   - Checks Python packages against known vulnerability databases
   - Reports outdated packages with security issues

4. **Private Key Detection (`detect-private-key`)**
   - Identifies SSH private keys, SSL certificates
   - Prevents accidental credential commits

5. **Additional Security Patterns**
   - Custom regex patterns for organization-specific secrets
   - Environment file security validation

## Installation and Setup

### Prerequisites

- Python 3.7 or higher
- Git repository
- pip package manager

### Step 1: Install Pre-commit

```bash
# Install pre-commit globally
pip install pre-commit

# Or install in development environment
pip install -r requirements-dev.txt
```

### Step 2: Install Git Hooks

```bash
# Navigate to your project root
cd /path/to/your/project

# Install pre-commit hooks
pre-commit install
```

### Step 3: Verify Installation

```bash
# Check if hooks are installed
ls -la .git/hooks/pre-commit

# Test hooks on all files
pre-commit run --all-files
```

## Configuration Details

The pre-commit configuration is defined in `.pre-commit-config.yaml`:

```yaml
repos:
  # Basic file checks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-toml
      - id: check-xml
      - id: check-merge-conflict
      - id: check-added-large-files
      - id: detect-private-key
      - id: check-case-conflict

  # Secret detection
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
        exclude: package.lock.json

  # Python security analysis
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.5
    hooks:
      - id: bandit
        args: ['-r', '.']
        exclude: tests/

  # Dependency vulnerability checking
  - repo: https://github.com/Lucas-C/pre-commit-hooks-safety
    rev: v1.3.2
    hooks:
      - id: python-safety-dependencies-check

  # Code formatting and quality
  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black
        language_version: python3

  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort
        args: ["--profile", "black"]

  - repo: https://github.com/pycqa/flake8
    rev: 6.0.0
    hooks:
      - id: flake8
        args: ['--max-line-length=88', '--ignore=E203,W503']

  # Infrastructure as Code security
  - repo: https://github.com/hadolint/hadolint
    rev: v2.12.0
    hooks:
      - id: hadolint
        name: Lint Dockerfiles
        types: [dockerfile]

  - repo: https://github.com/adrienverge/yamllint.git
    rev: v1.32.0
    hooks:
      - id: yamllint
```

## Security Tools Integration

### 1. Detect-Secrets

**Purpose**: Comprehensive secret detection using multiple methods

**Detection Methods**:

- **Entropy Analysis**: Identifies high-entropy strings (likely passwords/tokens)
- **Keyword Analysis**: Searches for common secret keywords
- **Pattern Matching**: Recognizes specific formats (AWS keys, JWTs, etc.)

**Configuration**:

```bash
# Initialize baseline (run once)
detect-secrets scan --baseline .secrets.baseline

# Update baseline when adding legitimate secrets
detect-secrets scan --update .secrets.baseline

# Audit baseline
detect-secrets audit .secrets.baseline
```

**Common Secret Types Detected**:

- API Keys (OpenAI, Stripe, SendGrid, etc.)
- AWS Access Keys and Secret Keys
- JWT Tokens
- Database Connection Strings
- SSH Private Keys
- GitHub Tokens
- Base64 High Entropy Strings

### 2. Bandit Security Analysis

**Purpose**: Python-specific security vulnerability detection

**Security Categories**:

- **B105**: Hardcoded password strings
- **B106**: Hardcoded password function defaults
- **B107**: Test for shell injection
- **B108**: Hardcoded temporary file
- **B110**: Try/except pass (security implications)
- **B112**: Try/except continue (security implications)
- **B113**: Request without timeout
- **B201**: Flask app with debug=True
- **B601**: Shell injection via subprocess

**Example Vulnerabilities**:

```python
# BAD: Hardcoded password (B105)
PASSWORD = "super_secret_123"

# BAD: SQL injection risk (B608)
query = "SELECT * FROM users WHERE id = " + user_id

# BAD: Shell injection (B602)
os.system("rm " + filename)

# GOOD: Environment variables
PASSWORD = os.getenv("DATABASE_PASSWORD")

# GOOD: Parameterized queries
query = "SELECT * FROM users WHERE id = %s"
cursor.execute(query, (user_id,))
```

### 3. Safety Dependency Scanner

**Purpose**: Check Python dependencies for known security vulnerabilities

**Features**:

- Scans `requirements.txt` and `Pipfile.lock`
- Cross-references with safety vulnerability database
- Reports CVE numbers and severity levels
- Suggests version upgrades

**Example Output**:

```text
╒══════════════════════════════════════════════════════════════════════════════╕
│                                                                              │
│                               /$$$$$$            /$$                         │
│                              /$$__  $$          | $$                         │
│           /$$$$$$$  /$$$$$$ | $$  \__//$$$$$$  /$$$$$$   /$$   /$$           │
│          /$$_____/ |____  $$| $$$$   /$$__  $$|_  $$_/  | $$  | $$           │
│         |  $$$$$$   /$$$$$$$| $$_/  | $$$$$$$$  | $$    | $$  | $$           │
│          \____  $$ /$$__  $$| $$    | $$_____/  | $$ /$$| $$  | $$           │
│          /$$$$$$$/|  $$$$$$$| $$    |  $$$$$$$  |  $$$$/|  $$$$$$$           │
│         |_______/  \_______/|__/     \_______/   \___/   \____  $$           │
│                                                          /$$  | $$           │
│                                                         |  $$$$$$/           │
│  by pyup.io                                             \______/            │
│                                                                              │
╞══════════════════════════════════════════════════════════════════════════════╡
│ REPORT                                                                       │
│ checked 25 packages, using default DB                                       │
╞══════════════════════════════════════════════════════════════════════════════╡
│ package                 │ installed │ affected           │ ID               │
╞══════════════════════════════════════════════════════════════════════════════╡
│ django                  │ 3.1.0     │ <3.1.13            │ 40291            │
│ pillow                  │ 8.0.0     │ <8.1.2             │ 39525            │
╘══════════════════════════════════════════════════════════════════════════════╛
```

## Code Quality Tools

### 1. Black Code Formatter

**Purpose**: Automatic Python code formatting

**Benefits**:

- Consistent code style across team
- Reduces formatting debates
- Improves readability

**Configuration** (in `pyproject.toml`):

```toml
[tool.black]
line-length = 88
target-version = ['py38']
include = '\.pyi?$'
extend-exclude = '''
/(
  # directories
  \.eggs
  | \.git
  | \.tox
  | \.venv
  | build
  | dist
)/
'''
```

### 2. isort Import Sorting

**Purpose**: Organize Python imports consistently

**Configuration**:

```toml
[tool.isort]
profile = "black"
multi_line_output = 3
line_length = 88
known_first_party = ["your_project"]
```

### 3. Flake8 Linting

**Purpose**: Python code style and error checking

**Common Error Codes**:

- **E501**: Line too long
- **E401**: Multiple imports on one line  
- **F401**: Imported but unused
- **W503**: Line break before binary operator

**Configuration** (in `.flake8`):

```ini
[flake8]
max-line-length = 88
ignore = E203, W503, E902
exclude = 
    .git,
    __pycache__,
    venv,
    env,
    build,
    dist
```

## Common Issues and Solutions

### Issue 1: "pre-commit: command not found"

**Problem**: Pre-commit is not in PATH or not installed

**Solutions**:

```bash
# Check if pre-commit is installed
pip list | grep pre-commit

# Install if missing
pip install pre-commit

# Use Python module if PATH issues
python -m pre_commit install
```

### Issue 2: Secrets Detected in Legitimate Files

**Problem**: False positive secret detection

**Solutions**:

1. **Add pragma comment**:

```python
API_EXAMPLE = "sk-example123456789"  # pragma: allowlist secret
```

2. **Update baseline**:

```bash
detect-secrets scan --update .secrets.baseline
detect-secrets audit .secrets.baseline
```

3. **Exclude files in config**:

```yaml
- id: detect-secrets
  exclude: 'tests/fixtures/.*\.json$'
```

### Issue 3: Bandit False Positives

**Problem**: Security warnings on test files or safe code

**Solutions**:

1. **Skip specific tests**:

```python
import subprocess
subprocess.call(['ls', '-l'])  # nosec B602
```

2. **Exclude directories**:

```yaml
- id: bandit
  exclude: 'tests/|docs/'
```

### Issue 4: Hook Execution Failures

**Problem**: Hooks fail due to missing dependencies

**Solutions**:

```bash
# Clear pre-commit cache
pre-commit clean

# Reinstall all hooks
pre-commit install --install-hooks

# Run specific hook to debug
pre-commit run bandit --all-files
```

## Best Practices

### 1. Team Adoption

**Gradual Implementation**:

1. Start with basic hooks (formatting, syntax checks)
2. Add security hooks incrementally
3. Train team on resolving common issues
4. Document organization-specific patterns

**Communication**:

- Share this documentation with all team members
- Conduct training sessions on security best practices
- Create internal FAQ for common issues

### 2. Configuration Management

**Version Control**:

- Keep `.pre-commit-config.yaml` in version control
- Document any custom configurations
- Use consistent hook versions across projects

**Regular Updates**:

```bash
# Update hook versions
pre-commit autoupdate

# Test updates
pre-commit run --all-files
```

### 3. Secret Management

**Prevention Strategies**:

- Use environment variables for secrets
- Implement proper secret management tools (HashiCorp Vault, AWS Secrets Manager)
- Educate developers on secure coding practices

**Example Secure Patterns**:

```python
# BAD: Hardcoded secrets
API_KEY = "sk-1234567890abcdef"
DATABASE_URL = "postgresql://admin:password@localhost/db"

# GOOD: Environment variables
import os
API_KEY = os.getenv("API_KEY")
DATABASE_URL = os.getenv("DATABASE_URL")

# BETTER: Configuration management
from config import settings
API_KEY = settings.API_KEY
DATABASE_URL = settings.DATABASE_URL
```

### 4. CI/CD Integration

**GitHub Actions Example**:

```yaml
name: Pre-commit Checks
on: [push, pull_request]
jobs:
  pre-commit:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-python@v4
      with:
        python-version: 3.9
    - uses: pre-commit/action@v3.0.0
```

## Testing Pre-commit Hooks

### Manual Testing

**Test Individual Hooks**:

```bash
# Test specific hook
pre-commit run bandit --all-files
pre-commit run detect-secrets --all-files

# Test on specific files
pre-commit run --files src/models.py

# Skip hooks for emergency commits
git commit --no-verify -m "Emergency fix"
```

### Automated Testing

**Create Test Files**:

```bash
# Create file with intentional issues
echo 'PASSWORD = "secret123"' > test_security.py
git add test_security.py
git commit -m "Test security"  # Should fail
```

**Validation Scripts**:

```bash
#!/bin/bash
# test-precommit.sh
set -e

echo "Testing pre-commit hooks..."

# Test secret detection
echo 'API_KEY = "sk-test123456789"' > temp_test.py
git add temp_test.py
if git commit -m "Test commit" 2>/dev/null; then
    echo "ERROR: Secret detection failed!"
    exit 1
else
    echo "✓ Secret detection working"
fi

# Cleanup
git reset --hard HEAD
rm -f temp_test.py

echo "All tests passed!"
```

## Troubleshooting

### Common Error Messages

**1. "Potential secrets about to be committed"**

```text
Secret Type: Base64 High Entropy String
Location: config.py:10
```

**Solution**: Remove the secret or add to allowlist

**2. "bandit security issues found"**

```text
Issue: [B105:hardcoded_password_string] Possible hardcoded password
```

**Solution**: Move password to environment variable

**3. "flake8 style violations"**

```text
E501 line too long (120 > 88 characters)
```

**Solution**: Break long lines or adjust configuration

### Debug Commands

```bash
# Verbose hook execution
pre-commit run --verbose bandit

# Show hook output
pre-commit run --show-diff-on-failure

# Run hooks manually
.git/hooks/pre-commit

# Check hook installation
pre-commit --version
pre-commit sample-config
```

### Performance Optimization

**Speed Up Hooks**:

1. **Use file filters**:

```yaml
- id: bandit
  files: '\.py$'
  exclude: 'tests/|migrations/'
```

2. **Parallel execution** (enabled by default)

3. **Cache management**:

```bash
# Clear cache if issues persist
pre-commit clean
rm -rf ~/.cache/pre-commit
```

## Integration with Development Workflow

### IDE Integration

**VS Code Extensions**:

- Python extension for linting
- GitLens for commit history
- Pre-commit hook extension

**Configuration** (`.vscode/settings.json`):

```json
{
  "python.linting.enabled": true,
  "python.linting.flake8Enabled": true,
  "python.formatting.provider": "black",
  "python.sortImports.args": ["--profile", "black"],
  "editor.formatOnSave": true
}
```

### Git Workflow

**Recommended Process**:

1. Make code changes
2. Run `pre-commit run --all-files` before staging
3. Fix any issues found
4. Stage changes with `git add`
5. Commit with descriptive message
6. Hooks run automatically and block if issues found

## Conclusion

Pre-commit hooks provide essential security and quality assurance for the AIOutlet project. By implementing comprehensive checks before code reaches the repository, we:

- **Prevent security vulnerabilities** from being committed
- **Maintain consistent code quality** across the team  
- **Reduce code review overhead** through automated checks
- **Improve overall security posture** of the project

Regular maintenance, team training, and continuous improvement of hook configurations ensure maximum effectiveness of this security layer.

For questions or additional support, please refer to the project's security documentation or contact the development team.

---

**Last Updated**: August 20, 2025  
**Version**: 1.0  
**Author**: AIOutlet Development Team

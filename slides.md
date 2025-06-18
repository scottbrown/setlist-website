---
marp: true
theme: default
paginate: true
backgroundColor: white
backgroundImage: url('https://marp.app/assets/hero-background.svg')
---

# SetList
## Automated AWS Configuration Management

*A Go CLI tool for Enterprise AWS Organizations*

---

# The Problem

## AWS SSO Configuration Pain Points

- **Multiple AWS Accounts**: Organizations often have 10+ AWS accounts
- **Multiple Permission Sets**: Each account may have 3-5 different roles
- **Manual Process**: `aws sso configure` requires individual profile setup
- **Team Consistency**: Hard to ensure everyone has the same configuration
- **Configuration Drift**: Changes in permission sets require manual updates

**Result**: Hours of manual work, inconsistent configurations, frustrated developers

---

# The Solution: SetList

## Automated AWS Config Generation

```bash
# One command generates complete AWS configuration
setlist --sso-session myorg --sso-region us-east-1 --output ~/.aws/config
```

**What it does:**
- 🔍 Discovers all accessible AWS accounts in your organization
- 🔑 Finds all permission sets provisioned to each account
- 📝 Generates complete `.aws/config` file automatically
- 🏷️ Supports account nicknames for better usability
- ⚡ Reduces hours of work to seconds

---

# Architecture Overview

## Modern Go CLI Application

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   CLI Layer     │    │  Business Logic  │    │  AWS Services   │
│   (Cobra)       │───▶│   (Core Types)   │───▶│ Organizations   │
│                 │    │                  │    │ SSO Admin       │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│ File Generation │    │  Type Safety &   │    │   AWS SDK v2    │
│   (INI Format)  │    │   Validation     │    │  with Context   │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

**Key Technologies**: Go 1.24, AWS SDK v2, Cobra CLI, go-ini

---

# Core Components

## CLI Layer (`/cmd/`)
- **main.go**: Application entry point
- **rootCmd.go**: Command handling with Cobra framework
- **flags.go**: Command-line flag definitions

## Business Logic
- **organizations.go**: AWS Organizations API client
- **ssoadmin.go**: AWS SSO Admin API interactions  
- **config_file.go**: Configuration file modeling
- **file_builder.go**: INI file generation logic
- **profile.go**: Profile data structures with validation

---

# AWS Integration

## Required Permissions (Read-Only)

```go
// Minimal permissions required
permissions := []string{
    "organizations:ListAccounts",                    // Discover accounts
    "sso:ListInstances",                            // Get SSO instance
    "sso:ListPermissionSetsProvisionedToAccount",   // Find permission sets
    "sso:DescribePermissionSet",                    // Get permission details
}
```

## API Integration Pattern

```go
func ListAccounts(ctx context.Context, client OrganizationsClient) ([]types.Account, error) {
    var accounts []types.Account
    paginator := organizations.NewListAccountsPaginator(client, &organizations.ListAccountsInput{})
    
    for paginator.HasMorePages() {
        page, err := paginator.NextPage(ctx)
        if err != nil {
            return nil, fmt.Errorf("failed to list accounts: %w", err)
        }
        accounts = append(accounts, page.Accounts...)
    }
    return accounts, nil
}
```

---

# Type Safety & Domain Modeling

## Custom Types with Validation

```go
type AWSAccountId string
type IdentityStoreId string
type Region string
type SessionDuration string

// Constructor with validation
func NewAWSAccountId(id string) (AWSAccountId, error) {
    if len(id) != 12 {
        return AWSAccountId(""), errors.New("invalid length for AWS account id")
    }
    if !isNumeric(id) {
        return AWSAccountId(""), errors.New("AWS account id must be numeric")
    }
    return AWSAccountId(id), nil
}
```

**Benefits**: Compile-time safety, clear domain modeling, validation at boundaries

---

# Configuration File Generation

## Generated Output Structure

```ini
[default]
# Generated on: 2025-06-18T10:15:30 UTC
sso_session = myorg

[sso-session myorg]
sso_start_url = https://d-12345abcde.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access

# Administrator access. Session Duration: PT12H
[profile 123456789012-AdministratorAccess]
sso_session = myorg
sso_account_id = 123456789012
sso_role_name = AdministratorAccess

# Developer access. Session Duration: PT8H
[profile 123456789012-DeveloperAccess]
sso_session = myorg
sso_account_id = 123456789012
sso_role_name = DeveloperAccess
```

---

# Advanced Features

## Account Nickname Mapping

```bash
# Map account IDs to friendly names
setlist --sso-session myorg --sso-region us-east-1 \
  --mapping "123456789012=prod,210987654321=staging" \
  --output ~/.aws/config
```

**Result**: Dual profiles for compatibility
- `123456789012-AdministratorAccess` (numeric ID)
- `prod-AdministratorAccess` (friendly name)

## Flexible Output Options
- **File output**: `--output ~/.aws/config`
- **Stdout**: `--stdout` (for piping/scripting)
- **Friendly SSO URLs**: `--sso-friendly-name company-name`

---

# Build System & Development

## Task-Based Build System

```yaml
# taskfile.yml
tasks:
  build:   go build -o .build/setlist ./cmd
  test:    go test ./...
  check:   [sast, vet, vuln]  # Security scanning
  fmt:     go fmt ./...
  bench:   go test -bench=. -benchmem ./...
  release: Creates multi-platform artifacts
```

## Development Commands

```bash
task build    # Local development build
task test     # Run comprehensive test suite
task check    # Security scans (gosec, govulncheck)
task release VERSION=v1.2.3  # Multi-platform release
```

---

# Quality Assurance

## Comprehensive Testing Strategy

- **Unit Tests**: All core functions with table-driven tests
- **Benchmark Tests**: Performance monitoring for critical paths
- **Mock Interfaces**: AWS services mocked for reliable testing
- **Integration Tests**: End-to-end workflow validation

## Security & Code Quality

```bash
# Automated security scanning
gosec ./...           # Static application security testing
go vet ./...          # Go-specific code analysis  
govulncheck ./...     # Vulnerability scanning for dependencies
```

**Result**: Production-ready code with security best practices

---

# CI/CD Pipeline

## GitHub Actions Workflow

```yaml
# Multi-platform builds
platforms: [linux, windows, darwin]
architectures: [amd64, arm64]

# Automated processes
- Unit testing with 5-minute timeout
- Security scanning (SAST, vulnerability checks)
- Multi-platform binary compilation
- GitHub release creation with artifacts
- SBOM (Software Bill of Materials) generation
```

## Release Process

1. **Tag Creation** → Triggers automated release
2. **Cross-Compilation** → 6 platform/architecture combinations
3. **Security Scanning** → Multiple security tools
4. **Artifact Generation** → Binaries + SBOM for compliance
5. **GitHub Release** → Automated release notes and assets

---

# Performance & Scalability

## Efficient AWS API Usage

```go
// Context-aware operations with timeouts
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
defer cancel()

// Efficient pagination for large organizations
paginator := organizations.NewListAccountsPaginator(client, input)
for paginator.HasMorePages() {
    // Process pages without loading all data into memory
}
```

## Benchmarking Results

- **Memory Usage**: Minimal footprint, streaming approach
- **API Efficiency**: Batched operations, proper pagination
- **Execution Time**: Sub-minute execution for 50+ account organizations

---

# Real-World Usage

## Command Examples

```bash
# Basic usage
setlist --sso-session myorg --sso-region us-east-1 --output ~/.aws/config

# With AWS profile authentication  
setlist --sso-session myorg --sso-region us-east-1 --profile admin

# Account discovery
setlist --list-accounts --sso-region us-east-1

# Permission check
setlist --permissions

# Team onboarding (with nicknames)
setlist --sso-session acme --sso-region us-east-1 \
  --mapping "123456789012=prod,987654321098=staging,456789012345=dev" \
  --output ~/.aws/config
```

---

# Impact & Benefits

## Time Savings
- **Before**: 30+ minutes manual configuration per developer
- **After**: 30 seconds automated generation
- **Team of 10**: 5+ hours saved per configuration update

## Consistency & Reliability
- **Eliminated**: Manual configuration errors
- **Ensured**: Team-wide configuration consistency  
- **Automated**: Updates when permission sets change

## Developer Experience
- **Onboarding**: New team members get complete config instantly
- **Maintenance**: Zero-effort configuration updates
- **Discoverability**: See all available accounts and roles

---

# Technical Excellence

## Modern Go Practices

- **Type Safety**: Custom domain types with validation
- **Error Handling**: Comprehensive error wrapping and context
- **Concurrency**: Context-aware operations with cancellation
- **Testing**: Table-driven tests with high coverage
- **Documentation**: Clear code structure and comprehensive README

## Enterprise Ready

- **Security**: Minimal permissions, no credential storage
- **Compliance**: SBOM generation, vulnerability scanning
- **Reliability**: Robust error handling, timeout management
- **Observability**: Clear logging and error messages

---

# Future Roadmap

## Planned Enhancements

- **Homebrew Distribution**: Package manager integration
- **Configuration Validation**: Verify generated profiles work
- **Template Support**: Custom profile templates
- **Backup/Restore**: Configuration history management
- **Interactive Mode**: Guided configuration setup

## Community & Contributions

- **Open Source**: MIT License, contributions welcome
- **Issue Tracking**: GitHub Issues for bug reports and features
- **Documentation**: Comprehensive README and examples
- **Release Notes**: Semantic versioning with clear changelogs

---

# Conclusion

## SetList: Transforming AWS Configuration Management

**Problem Solved**: Manual, error-prone AWS SSO configuration
**Solution Delivered**: Automated, consistent, enterprise-ready CLI tool
**Technology**: Modern Go application with AWS SDK v2 integration
**Impact**: Hours of manual work reduced to seconds

**Key Takeaways**:
- ✅ Eliminates manual AWS configuration overhead  
- ✅ Ensures team-wide consistency and reduces errors
- ✅ Demonstrates modern Go development best practices
- ✅ Production-ready with comprehensive testing and security
- ✅ Enterprise-scale with multi-platform distribution

**GitHub**: `github.com/scottbrown/setlist`

---

<style>
section {
  font-size: 20px;
}
h1 {
  color: #2563eb;
}
h2 {
  color: #1d4ed8;
}
code {
  background-color: #f1f5f9;
  padding: 2px 4px;
  border-radius: 3px;
}
pre {
  background-color: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  padding: 16px;
}
</style>
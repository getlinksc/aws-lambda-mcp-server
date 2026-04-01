# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## Unreleased

### Added

- GitHub Actions workflows to publish to PyPI as `aws-lambda-mcp-server` and `lambda-mcp-server`
- Expanded README with examples for tools, resources, session management, and custom session backends

## [0.1.14] - 2026-03-06

### Added

- Streamlined boto3 user agent tracking (`md/awslabs#mcp#mcp-lambda-handler#<version>`)

### Fixed

- `Optional[T]` type hints now correctly resolve to the inner type JSON Schema instead of defaulting to `string`

## [0.1.13] - 2026-01-09

### Fixed

- Upgraded urllib3 to resolve security advisory GHSA-38jv-5279-wg99

## [0.1.12] - 2025-10-24

### Fixed

- Tool names are no longer converted from `snake_case` to camelCase — the original function name is preserved

## [0.1.11] - 2025-07-31

### Added

- Resource support: `resources/list` and `resources/read` MCP methods
- `Resource`, `FileResource`, and `StaticResource` types
- `@mcp.resource()` decorator and `mcp.add_resource()` for registering resource providers

## [0.1.10] - 2025-07-01

### Added

- `ImageContent` support — tools that return `bytes` are automatically encoded as base64 image content blocks with MIME type auto-detection (JPEG, PNG, GIF, WebP)

### Fixed

- Notification requests (no `id` field) now correctly return HTTP 202 instead of an error

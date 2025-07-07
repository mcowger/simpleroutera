# LLM Proxy System - Detailed Requirements Specification

## Table of Contents
1. [System Overview](#system-overview)
2. [Functional Requirements](#functional-requirements)
3. [Provider Management](#provider-management)
4. [Virtual Provider System](#virtual-provider-system)
5. [Usage Tracking & Limits](#usage-tracking--limits)
6. [Configuration Management](#configuration-management)
7. [Web User Interface](#web-user-interface)
8. [API Requirements](#api-requirements)
9. [Data Persistence](#data-persistence)
10. [Error Handling & Recovery](#error-handling--recovery)
11. [Performance Requirements](#performance-requirements)
12. [Deployment Requirements](#deployment-requirements)
13. [Implementation Guidelines](#implementation-guidelines)

## System Overview

### Purpose
Build a proxy service that accepts OpenAI-compatible API requests and intelligently routes them to various LLM providers based on configurable rules, with focus on cost containment through usage tracking and limit enforcement.

### Core Capabilities
- Accept OpenAI Chat Completion API requests (streaming and non-streaming)
- Route requests to HTTP-based LLM providers or local executable models
- Track usage (tokens, requests, costs, errors) across multiple time windows
- Enforce configurable limits with fallback behavior
- Provide web-based configuration and monitoring interface
- Persist configuration and usage data to local files

### General Expectations
- Avoid writing net new code for things that are likely to exist already 
in public libraries and modules.
- When using public libraries, use only ones with typescript types defined.
- Some examples we should consider using.  Try to use these if possible.  Use the Context7 MCP to gather API information where possible.
  - openai-api-mock - good for testing / mocking.
  - ai-sdk/openai-compatible - allows exposing a OpenAI compatible API.
  - token.js - Integrate 200+ LLMs with one TypeScript SDK using OpenAI's forma

## Functional Requirements

### FR-1: API Compatibility
**Requirement**: The system must expose an OpenAI-compatible Chat Completions API endpoint.

**Details**:
- Accept requests at `/v1/chat/completions`
- Support all standard OpenAI chat completion parameters (model, messages, temperature, max_tokens, etc.)
- Support both streaming (`stream: true`) and non-streaming responses
- Return responses in exact OpenAI format
- Handle streaming responses with Server-Sent Events format
- No authentication required for incoming requests. The authorization token that the client sends should not be checked or validated, but should be kept with any metric data for future use.

### FR-2: Base Provider Support
**Requirement**: The system must support multiple types of providers that can fulfill chat completion requests.

**Details**:
- **HTTP Providers**: Standard LLM APIs that accept OpenAI-format requests
- **Local Providers**: Executable programs that communicate via stdin/stdout
- Each provider must be individually configurable and manageable
- Providers can be enabled/disabled without system restart

### FR-3: Request Routing
**Requirement**: The system must route incoming requests to appropriate providers based on configurable rules.

**Details**:
- Support direct provider selection (passthrough mode)
- Support virtual provider routing with priority-based selection
- Route to highest priority available provider that hasn't exceeded limits
- Automatically failover to next priority provider on errors or limits
- Respect provider cooldown periods after errors

### FR-4: Usage Tracking
**Requirement**: The system must track detailed usage statistics for all providers.

**Details**:
- Track per provider across three time windows: per-minute, daily, monthly
- Track request count, token usage (input/output/total), error count, estimated cost
- Reset counters automatically at time window boundaries
- Maintain running totals that persist across application restarts
- Support manual reset of usage statistics

### FR-5: Limit Enforcement
**Requirement**: The system must enforce configurable usage limits and prevent violations.

**Details**:
- Support token count limits and request count limits
- Support both "hard" limits (must not be exceeded) and "soft" limits (warnings only)
- Check limits before routing requests to providers
- Block requests to providers that have exceeded hard limits
- Allow soft limit violations but log warnings
- Support different limits for each time window (minute/day/month).  Enable enforcement based on 'OR' logic.

## Provider Management

### PM-1: HTTP Provider Configuration
**Requirement**: Support configuration of HTTP-based LLM providers.

**Specification**:
- Base URL for the provider API endpoint
- Optional API key/authorization headers
- Custom HTTP headers if needed
- Request timeout configuration
- Retry count before considering provider failed
- Health check endpoint configuration

### PM-2: Local Provider Configuration
**Requirement**: Support configuration of local executable model providers.   Do not implement any such providers at this time, simply create the structure and classes and methods.

**Specification**:
- Path to executable file
- Command line arguments to pass to executable
- Working directory for execution
- Process timeout configuration
- Maximum concurrent processes allowed
- Communication protocol: send JSON request via stdin, receive JSON response via stdout
- Error handling for process crashes or timeouts

### PM-3: Provider Health Monitoring
**Requirement**: Monitor provider health and availability.

**Specification**:
- Periodic health checks for HTTP providers (call health endpoint or test request)
- Process availability checks for local providers
- Mark providers as unhealthy if consecutive failures exceed threshold
- Automatically retry unhealthy providers after cooldown period
- Display provider health status in management interface

### PM-4: Provider Error Handling
**Requirement**: Handle provider errors gracefully with configurable recovery.

**Specification**:
- Configurable retry count per provider before marking as failed
- Two cooldown strategies:
  - Fixed time cooldown (e.g., 5 minutes)
  - Exponential backoff (increasing delay after each consecutive failure)
- Reset consecutive error count on successful request
- Continue tracking errors during cooldown period
- Log all provider errors with timestamps and details

## Virtual Provider System

### VP-1: Virtual Provider Definition
**Requirement**: Support creation of virtual providers that aggregate multiple base providers.

**Specification**:
- Virtual provider has same interface as base providers
- Composed of 2 or more base providers (HTTP or local)
- Each base provider assigned a priority level (1 = highest priority)
- Virtual provider inherits usage tracking capabilities
- Can be used in place of base providers in client requests

### VP-2: Priority-Based Routing
**Requirement**: Route requests through virtual providers using priority-based selection.

**Specification**:
- Always attempt highest priority provider first
- Move to next priority level only if current provider:
  - Is in cooldown period
  - Has exceeded hard limits
  - Is marked as unhealthy
  - Returns error after all retries exhausted
- Return to higher priority providers once they become available
- Track which provider ultimately fulfilled each request

### VP-3: Fallback Behavior
**Requirement**: Implement robust fallback behavior when providers fail.

**Specification**:
- On provider error, immediately try next priority provider
- Do not retry same provider once it enters cooldown
- If all providers in virtual provider are unavailable, return error to client
- For streaming requests, terminate stream on error (no mid-stream switching)
- Log all fallback events with provider details and reasons

### VP-4: Virtual Provider Limits
**Requirement**: Support limit enforcement at virtual provider level.

**Specification**:
- Virtual provider tracks aggregate usage across all constituent providers
- Virtual provider limits are separate from individual provider limits
- Check virtual provider limits before routing to any constituent provider
- Virtual provider limit violations prevent routing to any constituent provider
- Individual provider limits still enforced within virtual provider

## Usage Tracking & Limits

### UT-1: Usage Data Collection
**Requirement**: Collect comprehensive usage data from all provider interactions.

**Specification**:
- Parse token usage from provider responses when available
- Estimate token usage when not provided by provider (using character count approximation)
- Record timestamp for each request
- Track success/failure status
- Calculate costs using manually configured per-token pricing
- Store data for three time windows simultaneously

### UT-2: Time Window Management
**Requirement**: Manage usage data across different time windows with automatic resets.

**Specification**:
- Per-minute window: resets every minute at :00 seconds
- Daily window: resets at midnight local time
- Monthly window: resets on first day of month at midnight
- Automatic reset must not lose historical data needed for monitoring
- Support manual reset of any time window
- Handle system restarts gracefully (preserve data across restarts)

### UT-3: Cost Calculation
**Requirement**: Calculate estimated costs for all provider usage.

**Specification**:
- Configure cost per million input tokens and per million output tokens for each provider
- Support different currencies per provider
- Calculate costs in real-time as requests are processed
- Display cumulative costs in management interface
- Support cost limit enforcement (treat as token-based limit inferred from price data)

### UT-4: Limit Configuration
**Requirement**: Support flexible limit configuration for all providers.

**Specification**:
- Configure separate limits for each provider and time window combination
- Support token limits (max input tokens, output tokens, or total tokens)
- Support request count limits (max requests per time window)
- Mark each limit as "hard" (enforced) or "soft" (warning only)
- Allow multiple limit types per provider/time window
- Support disabling limits entirely for specific providers

## Configuration Management

### CM-1: Configuration Persistence
**Requirement**: Persist all configuration data to local JSON files.

**Specification**:
- Store provider configurations in separate JSON file
- Store virtual provider definitions in separate JSON file
- Store limit configurations in separate JSON file
- Auto-save configuration when changes are made through UI
- Support manual save operation
- Validate configuration before saving
- Create backup of previous configuration on save

### CM-2: Configuration Validation
**Requirement**: Validate all configuration changes before applying.

**Specification**:
- Validate provider URLs and connection parameters
- Verify local executable paths exist and are executable
- Check that virtual provider references valid base providers
- Validate limit values are positive numbers
- Ensure required fields are populated
- Display clear error messages for invalid configurations

### CM-3: Dynamic Configuration Updates
**Requirement**: Support updating configuration without full system restart.

**Specification**:
- Allow adding/removing/modifying providers through UI
- Support enabling/disabling providers without restart
- Require restart only for fundamental changes (port, core settings)
- Provide "restart service" button in UI for when restart is needed
- Preserve usage data and active connections during restart
- Display current configuration status and restart requirements

### CM-4: Configuration Import/Export
**Requirement**: Support backup and migration of configurations.

**Specification**:
- Export all configuration as single JSON file
- Import configuration from JSON file
- Validate imported configuration before applying

## Web User Interface

### UI-1: Dashboard Overview
**Requirement**: Provide overview dashboard showing system status and usage.

**Specification**:
- Display list of all providers with current status (healthy/unhealthy/cooldown)
- Show current usage statistics for each provider across all time windows
- Display usage as percentage of configured limits
- Show cost totals and trends
- Highlight providers approaching or exceeding limits
- Refresh data when page is reloaded (no real-time updates required)
- Avoid complexities like Websockets - simplicity is the goal.

### UI-2: Provider Configuration Interface
**Requirement**: Provide forms for configuring all provider types.

**Specification**:
- Tabbed interface for different provider types (HTTP/Local/Virtual)
- Form validation with immediate feedback
- Test connection button for HTTP providers
- Test execution button for local providers
- Provider enable/disable toggles
- Delete provider with confirmation
- Clone/duplicate provider configurations

### UI-3: Virtual Provider Management
**Requirement**: Provide interface for creating and managing virtual providers.

**Specification**:
- Drag-and-drop interface for adding base providers to virtual provider
- Priority assignment interface (drag to reorder or numeric input)
- Visual representation of provider hierarchy and priorities
- Test virtual provider routing logic
- Preview which provider would be selected for current conditions

### UI-4: Limits Configuration
**Requirement**: Provide interface for configuring usage limits.

**Specification**:
- Grid interface showing all providers vs. all time windows
- Separate input fields for token limits and request limits
- Hard/soft limit toggle switches
- Bulk edit capabilities for applying same limits to multiple providers
- Limit calculator showing usage projections
- Clear indication of current usage vs. limits

### UI-5: Usage Analytics
**Requirement**: Provide visual analytics for usage tracking.

**Specification**:
- Charts showing usage trends over time for each provider
- Cost breakdown by provider and time period
- Error rate visualization
- Comparison charts between providers
- Export usage data as CSV
- Filter views by time range and provider

### UI-6: System Controls
**Requirement**: Provide administrative controls for system management.

**Specification**:
- Service restart button with confirmation
- Manual usage reset controls (by provider and time window)
- Configuration save/load interface
- Configuration validation results display

## API Requirements

### API-1: OpenAI Compatibility
**Requirement**: Implement complete OpenAI Chat Completions API compatibility.

**Specification**:
- Endpoint: `POST /v1/chat/completions`
- Accept all standard OpenAI parameters: model, messages, temperature, max_tokens, stream, top_p, frequency_penalty, presence_penalty, stop
- Return responses in exact OpenAI format with id, object, created, model, choices, usage fields
- Support streaming responses with proper SSE formatting
- Handle streaming termination correctly with [DONE] message
- Return appropriate HTTP status codes and error messages

### API-2: Management API
**Requirement**: Provide REST API for configuration and monitoring.

**Specification**:
- `GET /api/providers` - List all providers with status
- `POST /api/providers` - Create new provider
- `PUT /api/providers/{id}` - Update provider configuration
- `DELETE /api/providers/{id}` - Remove provider
- `GET /api/usage` - Get usage statistics for all providers
- `POST /api/usage/reset` - Reset usage for specific provider/time window
- `GET /api/limits` - Get current limit configurations
- `PUT /api/limits` - Update limit configurations
- `POST /api/system/restart` - Trigger system restart

### API-3: Direct Provider Access
**Requirement**: Support direct access to specific providers bypassing virtual provider logic.

**Specification**:
- Accept provider selection via custom header (e.g., `X-Provider-ID: provider-name`)
- Accept provider selection via path (e.g., `/{providername}/v1/chat`)
- Route directly to specified provider without checking virtual provider rules
- Still enforce provider-specific limits and tracking
- Return error if specified provider is unavailable
- Log direct provider usage separately from virtual provider usage

## Data Persistence

### DP-1: Usage Data Storage
**Requirement**: Store usage data in single file with periodic persistence.

**Specification**:
- Serialize all usage data to JSON file every 5 minutes
- Store data in memory for fast access during operation
- Include metadata: timestamps, provider IDs, time windows
- Maintain rolling window of data (maximum 1 day retention)
- Handle file corruption gracefully (create new file if corrupted)
- Ensure atomic writes to prevent partial data corruption

### DP-2: Configuration Storage
**Requirement**: Store configuration data in structured JSON files.

**Specification**:
- Single file for all data including for providers, virtual providers, and limits
- Human-readable JSON format with proper indentation
- Create backup files before overwriting configuration

### DP-3: Data Recovery
**Requirement**: Handle data recovery scenarios gracefully.

**Specification**:
- Continue operation if usage data file is missing (start with empty data)
- Load blank configuration if configuration files are missing

## Error Handling & Recovery

### EH-1: Provider Error Management
**Requirement**: Handle all provider error scenarios gracefully.

**Specification**:
- Network timeouts and connection failures
- Invalid API responses or malformed JSON
- HTTP error status codes (4xx, 5xx)
- Local process crashes or hangs
- Provider rate limiting responses
- Authentication failures
- Log all errors with context and timestamps

### EH-2: Request Error Handling
**Requirement**: Handle client request errors appropriately.

**Specification**:
- Invalid request format or missing required parameters
- Unsupported model names or parameters
- Requests that exceed all provider capabilities
- System overload or resource exhaustion
- Return proper HTTP status codes and error messages in OpenAI format

### EH-3: System Recovery
**Requirement**: Implement robust system recovery mechanisms.

**Specification**:
- Graceful startup after unexpected shutdowns


## Implementation Guidelines

### IG-1: Technology Constraints
**Requirement**: Use specified technologies and patterns.

**Specification**:
- TypeScript for all code (frontend and backend)
- Class-based architecture with interfaces and inheritance
- Express.js or similar for HTTP server
- React for frontend UI
- No external databases (file-based storage only)

### IG-2: Code Organization
**Requirement**: Maintain clean, maintainable code structure.

**Specification**:
- Separate modules for providers, usage tracking, limits, configuration
- Interface-based design for extensibility
- Clear separation between business logic and HTTP handling
- Comprehensive error handling throughout
- Unit testable components

### IG-4: Documentation
**Requirement**: Provide comprehensive documentation.

**Specification**:
- API documentation for all endpoints
- Configuration reference with examples
- Architecture documentation for future developers
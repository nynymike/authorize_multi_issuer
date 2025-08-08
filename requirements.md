# Requirements Document

## Introduction

The authorize_multi_issuer interface is a Cedar-based authorization system that processes multiple JWT tokens from various issuers and creates joined Cedar entities for policy evaluation. The system handles scenarios where developers may have multiple tokens from the same issuer (e.g., different scopes) and joins them into single queryable entities with aggregated claims.

The interface accepts an array of token objects, validates them, extracts claims, and joins tokens of the same type from the same issuer. This enables policies to check for combined scopes across multiple tokens while maintaining clean, ergonomic Cedar policy syntax through a flattened structure with predictable naming.

## Requirements

### Requirement 1

**User Story:** As a developer, I want to send multiple tokens from different issuers in a single authorization request, so that I can authenticate with multiple services simultaneously.

#### Acceptance Criteria

1. WHEN a developer sends an array of token objects THEN the system SHALL accept each token with a "mapping" and "payload" field
2. WHEN the system receives tokens THEN it SHALL support tokens from multiple different issuers in the same request
3. WHEN processing tokens THEN the system SHALL handle JWT payloads in the "payload" field
4. WHEN processing tokens THEN the system SHALL support arbitrary token mappings like "Acme::DolphinToken" without pre-defining schemas

### Requirement 2

**User Story:** As a developer, I want to include multiple tokens from the same issuer with different scopes, so that I can aggregate permissions across multiple token requests.

#### Acceptance Criteria

1. WHEN multiple tokens of the same type come from the same issuer THEN the system SHALL join them into a single entity
2. WHEN joining tokens THEN the system SHALL merge compatible claims like scopes into combined sets
3. WHEN joining tokens THEN the system SHALL preserve common claims like location and subject
4. WHEN joining tokens THEN the system SHALL use the latest expiration time and earliest issued time

### Requirement 3

**User Story:** As a policy author, I want to write ergonomic Cedar policies with clean syntax, so that I can easily create and maintain authorization rules.

#### Acceptance Criteria

1. WHEN tokens are processed THEN the system SHALL create flattened field names using predictable patterns
2. WHEN creating field names THEN the system SHALL use the format "{issuer_domain}_{token_type_simplified}"
3. WHEN policies are evaluated THEN the system SHALL support clean syntax like "context.token_collection.dolphin_sea_access_token.claims.location == 'miami'"
4. WHEN policies reference tokens THEN the system SHALL avoid repetitive nested structures that are hard to read

### Requirement 4

**User Story:** As a developer, I want to check for combined scopes across multiple tokens, so that I can implement authorization rules that require permissions from different token requests.

#### Acceptance Criteria

1. WHEN multiple access tokens from the same issuer have different scopes THEN the system SHALL merge all scopes into the joined token
2. WHEN policies check for scopes THEN the system SHALL enable checking for any scope from the combined set
3. WHEN evaluating scope requirements THEN the system SHALL support policies that require multiple scopes that originally came from different tokens
4. WHEN tokens are joined THEN the system SHALL maintain all original scope information without loss

### Requirement 5

**User Story:** As a system administrator, I want the authorization system to validate all tokens before processing, so that only authentic tokens are used for authorization decisions.

#### Acceptance Criteria

1. WHEN receiving tokens THEN the system SHALL validate JWT signatures for all tokens
2. WHEN validating tokens THEN the system SHALL verify token expiration times and not-before times
3. WHEN validating tokens THEN the system SHALL check token revocation status
4. IF any token fails validation THEN the system SHALL reject the entire authorization request with specific error details

### Requirement 6

**User Story:** As a policy author, I want to write policies that check specific token claims, so that I can create fine-grained authorization rules based on token content.

#### Acceptance Criteria

1. WHEN tokens are processed THEN the system SHALL preserve all JWT claims in the joined token entities
2. WHEN policies are evaluated THEN the system SHALL support checking any claim value using Cedar's "has" and equality operators
3. WHEN policies reference claims THEN the system SHALL support syntax like "token.claims.location == 'miami'"
4. WHEN custom token types are used THEN the system SHALL handle their claims the same way as standard token types

### Requirement 7

**User Story:** As a developer, I want to receive clear feedback about authorization decisions, so that I can understand why access was granted or denied.

#### Acceptance Criteria

1. WHEN authorization succeeds THEN the system SHALL return a success response with decision context
2. WHEN authorization fails THEN the system SHALL return specific error details about which policies failed
3. WHEN token validation fails THEN the system SHALL indicate which tokens were invalid and why
4. WHEN policy evaluation occurs THEN the system SHALL provide debugging information for troubleshooting
# Implementation Plan

Convert the feature design into a series of prompts for a code-generation LLM that will implement each step in a test-driven manner. Prioritize best practices, incremental progress, and early testing, ensuring no big jumps in complexity at any stage. Make sure that each prompt builds on the previous prompts, and ends with wiring things together. There should be no hanging or orphaned code that isn't integrated into a previous step. Focus ONLY on tasks that involve writing, modifying, or testing code.

## 1. Core Infrastructure Setup

- [ ] 1.1 Create authorize_multi_issuer module structure
  - Create new module `authorize_multi_issuer` in the Cedarling codebase
  - Define module structure with submodules for token processing, entity creation, and collection building
  - Add module exports and integration points with existing Cedarling architecture
  - _Requirements: 1.1, 1.2, 1.3_

- [ ] 1.2 Define core data structures and types
  - Create `TokenInput` struct with mapping and payload fields
  - Create `AuthorizeMultiIssuerRequest` struct for the main interface
  - Create `AuthorizeMultiIssuerResponse` struct for response handling
  - Define error types for token validation and processing failures
  - _Requirements: 1.1, 1.4, 7.2, 7.3_

- [ ] 1.3 Create unit tests for core data structures
  - Write tests for TokenInput serialization/deserialization
  - Write tests for request/response struct validation
  - Write tests for error type creation and formatting
  - _Requirements: 1.1, 7.1, 7.2, 7.3, 7.4_

## 2. Token Processing Integration

- [ ] 2.1 Integrate with existing Cedarling token validation
  - Identify and import existing JWT validation functions from Cedarling
  - Create wrapper functions to validate individual tokens from TokenInput array
  - Implement signature verification, expiration checking, and revocation status checking
  - Add error handling for token validation failures
  - _Requirements: 5.1, 5.2, 5.3, 5.4_

- [ ] 2.2 Implement JWT claim extraction
  - Create functions to parse JWT payloads and extract all claims
  - Convert JWT claims to Cedar-compatible data types (String, Long, Set, Record)
  - Handle arbitrary claim structures without pre-defined schemas
  - Preserve all original JWT claims without interpretation
  - _Requirements: 6.1, 6.3, 6.4_

- [ ] 2.3 Create comprehensive token validation tests
  - Write tests for valid token validation scenarios
  - Write tests for invalid token rejection (expired, invalid signature, revoked)
  - Write tests for claim extraction with various JWT structures
  - Write tests for error handling and specific error messages
  - _Requirements: 5.1, 5.2, 5.3, 5.4, 7.3_

## 3. Dynamic Entity Factory Implementation

- [ ] 3.1 Create dynamic Cedar entity generation
  - Implement function to create Cedar Token entities from validated JWT tokens
  - Generate unique token identifiers (using JWT jti or generated IDs)
  - Map arbitrary token types based on the mapping field (e.g., "Acme::DolphinToken")
  - Convert all JWT claims to Cedar Record<String, EntityValue> format
  - _Requirements: 6.1, 6.2, 6.4_

- [ ] 3.2 Implement token type handling
  - Create mapping logic for standard token types (Jans::Access_Token, Jans::Id_Token, etc.)
  - Handle custom token types dynamically without pre-defined schemas
  - Ensure consistent entity structure across different token mappings
  - Add validation for supported token mapping formats
  - _Requirements: 1.4, 6.4_

- [ ] 3.3 Create entity factory tests
  - Write tests for standard token type entity creation
  - Write tests for custom token type entity creation
  - Write tests for claim conversion to Cedar-compatible types
  - Write tests for entity structure consistency
  - _Requirements: 6.1, 6.2, 6.4_

## 4. Token Collection Builder Implementation

- [ ] 4.1 Implement token grouping by issuer and type
  - Create functions to group tokens by issuer domain
  - Group tokens by token type within each issuer
  - Identify tokens that need to be joined (same issuer + same type)
  - Create data structures to hold grouped tokens for processing
  - _Requirements: 2.1, 2.2_

- [ ] 4.2 Implement token joining logic
  - Create functions to join tokens of the same type from the same issuer
  - Implement scope merging for access tokens (combine scope arrays)
  - Preserve common claims (location, subject) across joined tokens
  - Use latest expiration time and earliest issued time for joined tokens
  - Handle claim conflicts with defined precedence rules
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 4.1, 4.2, 4.3, 4.4_

- [ ] 4.3 Implement flattened field naming
  - Create function to generate predictable field names using "{issuer_domain}_{token_type_simplified}" pattern
  - Convert issuer domains to safe field names (e.g., "idp.dolphin.sea" → "dolphin_sea")
  - Convert token types to simplified names (e.g., "Jans::Access_Token" → "access_token")
  - Handle custom token types in field naming (e.g., "Acme::DolphinToken" → "acme_dolphin_token")
  - _Requirements: 3.1, 3.2_

- [ ] 4.4 Create TokenCollection entity structure
  - Build the final TokenCollection entity with flattened field names
  - Add each joined token as a direct field in the collection
  - Include total_token_count field for metadata
  - Ensure the structure supports ergonomic Cedar policy syntax
  - _Requirements: 3.1, 3.2, 3.3, 3.4_

- [ ] 4.5 Create comprehensive collection builder tests
  - Write tests for token grouping by issuer and type
  - Write tests for token joining with scope merging
  - Write tests for field name generation with various issuer/type combinations
  - Write tests for TokenCollection entity creation
  - Write tests for handling edge cases (single tokens, no tokens, conflicting claims)
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 3.1, 3.2, 4.1, 4.2, 4.3, 4.4_

## 5. Cedar Policy Engine Integration

- [ ] 5.1 Integrate with existing Cedarling policy engine
  - Identify existing Cedar policy evaluation functions in Cedarling
  - Create interface to pass TokenCollection context to Cedar engine
  - Ensure TokenCollection entities are properly formatted for Cedar evaluation
  - Handle policy evaluation results and decision context
  - _Requirements: 6.2, 7.1_

- [ ] 5.2 Implement policy evaluation with token context
  - Create function to evaluate Cedar policies against TokenCollection context
  - Pass flattened token structure to Cedar for ergonomic policy syntax
  - Handle policy evaluation errors and provide debugging information
  - Return detailed authorization decisions with context
  - _Requirements: 6.2, 7.1, 7.4_

- [ ] 5.3 Create policy engine integration tests
  - Write tests for policy evaluation with simple token scenarios
  - Write tests for policies checking specific token claims
  - Write tests for policies requiring multiple tokens or combined scopes
  - Write tests for policy evaluation error handling
  - _Requirements: 6.2, 7.1, 7.4_

## 6. Main Interface Implementation

- [ ] 6.1 Implement authorize_multi_issuer main function
  - Create the main entry point function that accepts AuthorizeMultiIssuerRequest
  - Orchestrate the complete flow: validation → entity creation → collection building → policy evaluation
  - Handle errors at each stage and provide appropriate error responses
  - Return AuthorizeMultiIssuerResponse with decision and context
  - _Requirements: 1.1, 1.2, 1.3, 7.1, 7.2_

- [ ] 6.2 Add request/response serialization
  - Implement JSON serialization/deserialization for request and response types
  - Handle array of TokenInput objects in the request
  - Format response with decision, context, and error details
  - Ensure compatibility with existing Cedarling API patterns
  - _Requirements: 1.1, 1.2, 7.1, 7.2, 7.3_

- [ ] 6.3 Create end-to-end integration tests
  - Write tests for complete authorize_multi_issuer flow with valid tokens
  - Write tests for various token combinations (single issuer, multiple issuers, joined tokens)
  - Write tests for error scenarios (invalid tokens, policy failures)
  - Write tests for response formatting and error details
  - _Requirements: 1.1, 1.2, 1.3, 7.1, 7.2, 7.3, 7.4_

## 7. API Integration and Exposure

- [ ] 7.1 Add HTTP/REST endpoint for authorize_multi_issuer
  - Create new REST endpoint in Cedarling's web server
  - Map HTTP requests to AuthorizeMultiIssuerRequest structs
  - Handle HTTP request validation and error responses
  - Return appropriate HTTP status codes and JSON responses
  - _Requirements: 1.1, 7.1, 7.2, 7.3_

- [ ] 7.2 Add WebAssembly (WASM) interface if applicable
  - Create WASM bindings for authorize_multi_issuer function
  - Handle WASM-specific serialization and memory management
  - Ensure compatibility with existing Cedarling WASM interfaces
  - _Requirements: 1.1, 7.1_

- [ ] 7.3 Create API integration tests
  - Write tests for HTTP endpoint with various request scenarios
  - Write tests for WASM interface functionality
  - Write tests for API error handling and response formatting
  - Write tests for concurrent request handling
  - _Requirements: 1.1, 7.1, 7.2, 7.3_

## 8. Documentation and Examples

- [ ] 8.1 Create API documentation
  - Document the authorize_multi_issuer interface with request/response schemas
  - Provide examples of token input arrays and expected responses
  - Document error codes and troubleshooting guidance
  - Add integration examples for different programming languages
  - _Requirements: 7.4_

- [ ] 8.2 Create Cedar policy examples
  - Provide example policies using the flattened token structure
  - Show policies for common use cases (scope checking, multi-issuer validation)
  - Document the field naming convention and token claim access patterns
  - Include examples for custom token types
  - _Requirements: 3.3, 3.4, 6.2, 6.3_

- [ ] 8.3 Update Cedarling configuration documentation
  - Document any new configuration options for authorize_multi_issuer
  - Update existing Cedarling setup guides to include the new interface
  - Provide migration guidance for existing single-token policies
  - _Requirements: 7.4_

## 9. Performance and Security Testing

- [ ] 9.1 Implement performance benchmarks
  - Create benchmarks for token validation with large token arrays
  - Benchmark token joining and collection building performance
  - Test policy evaluation performance with complex token structures
  - Identify and optimize performance bottlenecks
  - _Requirements: 4.4_

- [ ] 9.2 Create security validation tests
  - Write tests for token tampering detection
  - Test handling of malicious or malformed token inputs
  - Validate proper error handling without information leakage
  - Test rate limiting and resource exhaustion scenarios
  - _Requirements: 5.1, 5.2, 5.3, 5.4_

## 10. Final Integration and Deployment

- [ ] 10.1 Integration with existing Cedarling features
  - Ensure compatibility with existing Cedarling configuration systems
  - Test integration with existing logging and monitoring
  - Verify compatibility with existing Cedarling deployment patterns
  - Update build scripts and packaging for the new interface
  - _Requirements: 7.1, 7.4_

- [ ] 10.2 Create deployment and migration guides
  - Document deployment steps for the new authorize_multi_issuer interface
  - Provide migration guidance from single-token authorization patterns
  - Create troubleshooting guides for common integration issues
  - Document rollback procedures if needed
  - _Requirements: 7.4_
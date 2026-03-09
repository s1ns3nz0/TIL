## 1. feat (New Feature)
### Use this when you add a new capability or a new endpoint to the system.
feat(auth): add OAuth2 provider for Google login
feat(storage): implement S3 multipart upload for large files
feat(ui): create reusable Modal component with accessibility support

## 2. fix (Bug Fix)
### Use this when you are correcting an error or unintended behavior.
fix(api): resolve null pointer exception in user profile retrieval
fix(vpc): correct NAT Gateway routing table association
fix(css): prevent horizontal scrolling on mobile viewports

## 3. docs (Documentation)
### Use this for any changes to the documentation, including READMEs, inline code comments, or API schemas.
docs: update installation instructions for macOS users
docs(api): document rate limiting headers in Swagger
docs: add architectural diagram to README

## 4. style (Formatting)
### Use this for changes that do not affect the logic of the code (white-space, formatting, missing semi-colons).
style: run Prettier on all source files
style(js): convert var declarations to let and const
style: fix indentation in Terraform module variables

## 5. refactor (Code Improvement)
### Use this for code changes that neither fix a bug nor add a feature, but improve the structure or readability.
refactor(db): extract query logic into a dedicated repository pattern
refactor: simplify conditional logic in traffic controller
refactor(iac): modularize VPC resources for reuse across environments

## 6. perf (Performance)
### Use this specifically for code changes that improve execution speed or resource consumption.
perf(image): optimize lazy loading for gallery thumbnails
perf(sql): add index to user_email column for faster lookups
perf: reduce bundle size by tree-shaking unused dependencies

## 7. test (Testing)
### Use this when adding new tests or refactoring existing ones.
test(unit): add boundary cases for date parser
test(e2e): implement Playwright flow for checkout process
test: increase code coverage for security utility functions

## 8. chore (Maintenance/Tools)
### Use this for tasks related to the build process, package managers, or auxiliary tools.
chore: update dependencies to latest stable versions
chore(git): add .terraform/ directory to .gitignore
chore(ci): update GitHub Action to use Node.js v20
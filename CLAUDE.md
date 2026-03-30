# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Important Privacy Notice
**All data referenced in this repository and in any EDI files (837P, 270, 271, 835, etc.) is fake test data. No real PHI (Protected Health Information) is ever shared in this development environment. All patient information, provider details, and claim data are synthetic and for testing purposes only.**

## Essential Development Commands

### Building and Setup
- **Fresh build**: `composer install --no-dev && npm install && npm run build && composer dump-autoload -o`
- **Development build**: `npm run dev-build` (builds SASS themes)
- **Theme compilation**: `gulp --dev && gulp watch` (watches for SASS changes)

### Docker Development Environment
- **Start development**: `cd docker/development-easy && docker compose up`
- **Access application**: http://localhost:8300 or https://localhost:9300 (admin/pass)
- **Database access**: http://localhost:8310 (phpMyAdmin) or port 8320 (direct MySQL)
- **Reset environment**: `docker compose exec openemr /root/devtools dev-reset-install-demodata`

### Testing Commands (via Docker)
- **All tests**: `docker compose exec openemr /root/devtools clean-sweep-tests`
- **Unit tests**: `docker compose exec openemr /root/devtools unit-test`
- **API tests**: `docker compose exec openemr /root/devtools api-test`
- **E2E tests**: `docker compose exec openemr /root/devtools e2e-test` (view at http://localhost:7900, password: openemr123)
- **Individual test suites**: `docker compose exec openemr /root/devtools [services|fixtures|validators|controllers|common]-test`

### Code Quality
- **PSR12 check**: `docker compose exec openemr /root/devtools psr12-report`
- **PSR12 fix**: `docker compose exec openemr /root/devtools psr12-fix`
- **Theme lint**: `docker compose exec openemr /root/devtools lint-themes-report`
- **PHP parse errors**: `docker compose exec openemr /root/devtools php-parserror`
- **Full cleanup**: `docker compose exec openemr /root/devtools clean-sweep`

### JavaScript/CSS Development
- **JS lint**: `npm run lint:js` or `npm run lint:js-fix`
- **Stylelint**: `npm run stylelint` or `npm run stylelint-fix`
- **JS tests**: `npm run test:js` or `npm run test:js-coverage`

## Architecture Overview

### Core Framework Structure
- **PHP Version**: 8.2+ with modern PHP features
- **MVC Framework**: Laminas components (formerly Zend) for routing, forms, validation
- **Dependency Injection**: Symfony DI container
- **Templating**: Twig templates with Smarty legacy support
- **Database**: MySQL/MariaDB with extensive migration system in `/sql/`

### Directory Structure
- **`/src/`**: Modern PSR-4 PHP classes organized by domain (Services, Controllers, RestControllers, etc.)
- **`/interface/`**: Traditional PHP interface files and UI components
- **`/library/`**: Legacy shared libraries and includes
- **`/modules/`**: Core modules and custom module system
- **`/templates/`**: Twig templates for modern UI components
- **`/public/`**: Web-accessible assets (built by Gulp)

### API Architecture
- **REST API**: Comprehensive REST endpoints defined in `_rest_routes.inc.php`
- **FHIR API**: Full FHIR R4 implementation in `/src/FHIR/`
- **OAuth2**: Complete OAuth2 implementation with scopes and client management
- **Swagger Documentation**: Auto-generated API docs available at `/swagger`

### Frontend Architecture
- **Theme System**: SASS-based themes compiled via Gulp, extending Bootstrap 4.6
- **JavaScript**: jQuery-based with modern ES6+ components
- **Module UI**: Angular 1.8 for complex interactive components
- **Asset Pipeline**: Gulp handles SASS compilation, minification, and RTL support

### Database Design
- **Multi-tenant**: Site-based multisite architecture in `/sites/`
- **Migrations**: Version-controlled SQL migrations in `/sql/`
- **Audit System**: Comprehensive audit logging with `audit_master` and `audit_details`
- **Character Set**: UTF8MB4 for full Unicode support

### Module System
- **Core Modules**: Built-in modules in `/interface/modules/`
- **Custom Modules**: Extensible module system with composer-based installation
- **Module Types**: Zend modules, custom modules, and legacy modules supported

### Security Architecture
- **Authentication**: Multi-factor authentication with LDAP support
- **Authorization**: ACL (Access Control Lists) via GACL library
- **CSRF Protection**: Built-in CSRF tokens for form submissions
- **Input Validation**: Comprehensive validation using Particle\Validator and custom validators

## EDI/Billing Architecture

### EDI File Generation
- **X12 Generator Classes**: Located in `/src/Billing/` (X125010837P.php for 837P claims)
- **EDI Interface**: Billing management UI in `/interface/billing/`
- **Billing Processor**: Central processing via `BillingProcessor` class
- **File Storage**: EDI files stored in `sites/default/documents/edi/[partner_name]/`

### Key EDI Components
- **837P Generation**: Professional claims via `X125010837P` class
- **270 Eligibility**: Eligibility inquiry via `EDI270` class
- **Database Storage**: Metadata in `claims` and `x12_remote_tracker` tables
- **X12 Partners**: Configuration in interface settings for trading partners

### EDI Formatting Customization
- **Segment Formatting**: Modify in X12 generator classes (src/Billing/)
- **Partner-specific Rules**: Can be configured per X12 partner
- **BCBSMA Customizations**: Specific formatting requirements already identified

## Development Workflow

### Docker Development
The primary development method uses Docker with extensive devtools:
- Development environment in `/docker/development-easy/`
- Comprehensive devtools script at `/root/devtools` within container
- Supports xdebug, profiling, SSL testing, multisite development

### Theme Development
- Themes in `/interface/themes/` using SASS
- Three theme types: `light` (default), `manila` (legacy-compatible), `colors` (color variants)
- RTL support automatically generated for all themes
- Build with `npm run dev` for development or `npm run build` for production

### Testing Strategy
- **Unit Tests**: Service layer and utility testing
- **API Tests**: REST and FHIR endpoint testing
- **E2E Tests**: Full browser automation with Selenium
- **Integration Tests**: Services, fixtures, validators, controllers testing

### Code Standards
- **PHP**: PSR-12 coding standards enforced
- **JavaScript**: ESLint configuration in `eslint.config.mjs`
- **CSS/SASS**: Stylelint with SASS guidelines
- **Documentation**: PHPDoc standards for all PHP classes

## Common Development Patterns

### Creating REST Endpoints
1. Define routes in `_rest_routes.inc.php`
2. Implement controller in `/src/RestControllers/`
3. Create service layer in `/src/Services/`
4. Add validation in `/src/Validators/`
5. Update Swagger docs with `docker compose exec openemr /root/devtools build-api-docs`

### Adding New Themes
1. Create SASS files in `/interface/themes/[theme-name]/`
2. Follow existing theme structure (import core, define overrides)
3. Build with `docker compose exec openemr /root/devtools build-themes`
4. RTL variants generated automatically

### Module Development
1. Use module template in `/interface/modules/custom_modules/`
2. Include `composer.json` for dependencies
3. Follow module registration patterns
4. Test with multisite if applicable

### Modifying EDI Generation
1. Locate generator class in `/src/Billing/` (e.g., X125010837P.php)
2. Modify segment generation methods
3. Test output format against payer requirements
4. Store test files in `sites/default/documents/edi/` for validation

## Key Files to Know
- **`composer.json`**: PHP dependencies and autoloading configuration
- **`package.json`**: Node.js dependencies and build scripts
- **`gulpfile.js`**: Asset compilation and theme building
- **`phpunit.xml`**: Test suite configuration
- **`_rest_routes.inc.php`**: All REST API route definitions
- **`setup.php`**: Database installation and upgrade logic
- **`src/Billing/X125010837P.php`**: 837P claim generation
- **`interface/billing/billing_process.php`**: Main billing processor interface

## Walnut Health EDI Integration Notes

### Current EDI Status
**Local Development (Working):**
- 837P generation with BCBSMA customizations ✓
- File storage in `sites/default/documents/edi/` ✓
- Database metadata in `claims` and `x12_remote_tracker` tables ✓
- Formatting fixes identified for BCBSMA submission ✓

**AWS Production (Needs Work):**
- Custom Docker image needed with local EDI modifications
- SFTP/HTTPS transport configuration required
- ACM certificate for walnuthealth.org needed
- IP whitelisting for AWS NAT Gateway required

### BCBSMA 837P Formatting Requirements
1. GS05: Time format should be HHMMSSDD
2. GS06: Group control number adjustment
3. NM109: Use V7BS instead of EIN
4. N302: Include suite/floor numbers in addresses
5. Segment separators: Use tildes (~) not line breaks
6. File format: Single line string, not line-by-line

### Testing Environments
- **SFTP DEV**: sftp.staging.bluecrossma.com
- **SFTP PROD**: sftp.bluecrossma.com
- **270/271 HTTPS**: Via Apigee gateway (walnuthealth.org certificate required)
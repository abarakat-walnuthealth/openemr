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

## Key Files to Know
- **`composer.json`**: PHP dependencies and autoloading configuration
- **`package.json`**: Node.js dependencies and build scripts
- **`gulpfile.js`**: Asset compilation and theme building
- **`phpunit.xml`**: Test suite configuration
- **`_rest_routes.inc.php`**: All REST API route definitions
- **`setup.php`**: Database installation and upgrade logic


## Additional User Planning Instructions
### Brief
The most urgent things to confirm are as follows:

#### 1. We can get EDI running on OpenEMR, both locally and on AWS
- Need to enable the billing view in the front end
- Need to see generation of these files: 270, 837P, 276
- Need to see capability for reading these files: 999, 271, 277, 835

#### 2. We can submit an 837P to BCBSMA
- Anticipate w/ OpenEMR, we will have new IP addresses for whitelisting
- Re-coordinate w/ Tracy / EDI team to get them listed
- Ensure OpenEMR hook can connect to BCBSMA SFTP: sftp.staging.bluecrossma.com (DEV) & sftp.bluecrossma.com (PROD)

#### 3. We can submit a 270 to BCBSMA
- Need domain (walnuthealth.org) w/ certificate plugged into ACM for auto-update
- Connect to Apigee for HTTPS / REST exchange

#### 4. We can receive an 835 back from BCBSMA: Per convo w/ Tracy, need a real file to confirm this: need to ask her if we can get a fake one to make sure our system reads it correctly.

#### 5. We can receive a 271 back from BCBSMA
- Need to establish HTTPS pathway through Apigee first
- Confirm we can receive 271 back from them

#### 6. We can modify these files if we need to, so that we are in control of our own destiny for billing
- ⚠️This will probably require iteration in PHP

#### 7. We can store and access all EDI files in the OpenEMR database, and can use them for downstream analysis for our financial accounting equations and business hypotheses
- Establish where OpenEMR stores these X12 files
- Figure out how to hook into them
- Confirm we can extract, read and analyze them

#### 8. We can submit a 276 and receive a 277 back from BCBSMA
#### 9. We can submit a 275 and receive a 277 back from BCBSMA
#### 10. We confirm OpenEMR has formatting flexibility in EDI rules across payors
#### 11. We can confirm that various transport protocols (SFTP, HTTPS) are supported, and can be easily added

Below, I have created a plan of attack for how we can move these items forward

### Testing Plan
#### Local Instance (openemr/openemr repo - this repository)
The local instance will be useful for confirming the following functionality from OpenEMR:


- Generation of EDI files
- Formatting of EDI files can be modified
- Storage & Access of EDI files for our downstream analyses in a database

In my opinion, it will be hard / a waste of time to try and test the following with the local instance:

##### Connection to SFTP / HTTPS for actual transfer
- When we want to communicate w/ BCBSMA, we will want to do so through cloud infrastructure (indeed, we may very well NEED to do this).
- ∆, I don’t think we should spend time testing the SFTP / HTTPS connectors through the local instance, as (i) it might not be possible (ii) even if it is, i suspect we’ll need to do something different for the AWS instance anyways.

##### Receipt of the 999, 271, 277, TA1, 835

##### Correct formatting for any file, except the 837P
Since we have a portal we can upload 837P files to, we can copy 837Ps that are generated from the local instance, and submit them to STAGING using sftp.staging.bluecrossma.com web portal. We do not have such a portal for 270, 275 or 276. I’ve reached out to Tracy to push again on the Optum portal for the 270s - stay tuned.

I would change my opinion on this if we discovered that transport set up on the local instance is quick, easy and sustainable for us to actually run, and the set up on the AWS instance is significantly harder / more expensive.

##### Proposed Tests
DONE: Confirm we can generate an 837P and see it in the interface, or access it programmatically
RESULT: We can see the 837P in the interface, and access it programmatically from the file storage system, within openemr/sites/default/documents/edi/<partner name>. We specify the <partner name> for the file path in the interface settings, under X12 partners.

DONE: Confirm the 837P is correctly formatted for BCBSMA receipt by submitting it to the sftp.staging.bluecrossma.com web portal, and seeing the 999, 277, TA1 generated in response in the portal as well. 
RESULT: It's not perfectly formatted, but that's okay right now - we can do the formatting stuff on AWS, since we've confirmed there is a pathway to editing the file formatting rules. However, we do need to invest more to understand how this will be managed across partners within the code. It may require additional library or file generation. An interim progress update with observed formatting differences that we should keep is described below:

    2025-09-12 10:13 AM: I managed to get the [837P out of OpenEMR]([20250912-first-openemr.837]), and do a line-for-line comparison with our [Golden example](/Golden-BCBSMA.V7BS.Claim.202507311921.837). 

    I also submitted the 837P from OpenEMR, and received an incomplete response. My analysis of continued TODOs for BCBSMA submission:

    1. GS Segment 
    - GS05: The time should be formatted HHMMSSDD; 
    - GS06: OpenEMR has a different group control number (should be 1?)
    2. BHT Segment
    - BHT03 may need to be 9 characters long (to test)
    3. FIXED: Loop1000A - Submitter Name / NM1 Segment
    - Using EIN (931416822) instead of V7BS in NM109 - it should be V7BS
    4. Loop2000A / PRV Segment
    - We don't include this, but it looks valid and correct to me
    5. Loop2000A - Billing Provider / N3 Segment - Missing SUITE 6 from address (N302)
    6. Loop2000B - Subscriber HL / Missing DMG Segment (?)
    7. Loop2010BB - Payer Name / N3 Segment - Missing #1300 from address (N302)
    8. Loop2300 - Claim Information
    - CLM01 - Different submitter ID; 
    - CLM10 - (PT sig. source) is included : in the Golden it isn't
    9. Loop2310B - Rendering Provider Name / PRV Segment - We don't include this segment in the golden example
    10. Loop2310C - Remove supervising provider (user error)
    11. In addition to these, every segment seems to be missing a tilde at the end as a segment separator, and the file is printed line-by-line: it needs to be printed as one long string, just like in the golden example

DONE: Confirm that we can change the 837P formatting to meet BCBSMA’s requirements, using the 837P generators
- Whether the previous test is a success or a failure, we’ll need this functionality anyways (as we anticipate BCBSMA will update their conditions & rules regularly)
RESULT: We can do this, but it might be slow going.

DONE: Confirm 837Ps are stored in a database, and confirm we can access that database
RESULT: 837P metadata is stored in claims, x12_remote_tracker tables; the 837P file itself is stored within the local filesystem; the database metadata contains the file name

- For each of 270/275/276:
- TODO: Confirm we can generate the file, and see it in the interface, or access it programmatically
- TODO: Confirm we can change the file formatting as we need to in the code
- TODO: Confirm the files are stored in a database, and confirm we can access that database

#### AWS Instance (openemr/openemr-on-eks repo - a repo in our parent directory, but useful context to understand what we're doing in this directory)
Several additional pieces of functionality become possible when we move to openemr-on-eks deployment.

First, we’ll need to re-confirm the tests above work on the AWS instance for EDI file generation, viewing, formatting/editing, storage and downstream access:

TODO: Confirm 837P generation and view it in the interface
TODO: Confirm 837P formatting / copy over any formatting changes we needed to make on local through sftp.staging.bluecrossma.com
TODO: Confirm we can change the 837P formatting in the openemr-on-eks repo
TODO: Confirm 837Ps are stored in a database and confirm we can access that database
TODO: Repeat for 270/275/276

Next, we’ll want to make sure that the EKS instance can connect through SFTP and HTTPS (Claude says that both are ready).


##### Proposed - 837P Testing via SFTP
TODO: Confirm our IP address - if it’s different, reach out to Tracy to confirm we are whitelisted again
TODO: Confirm the SFTP setup in OpenEMR can point to both sftp.staging.bluecrossma.com and sftp.bluecrossma.com
TODO: Ensure our credentials still work for that submission, and if not get new ones
- Probably want to get a different set of credentials for the AWS instance anyways, to keep “Daniel Barron” and “Walnut Health Automated submissions” separate for auditing purposes
TODO: Generate and submit an 837P to sftp.staging.bluecrossma.com

##### Proposed - 270 Testing via HTTPS
TODO: Set up AWS Certificate Manager (ACM) on the openemer-on-eks deployment
TODO: Confirm the domain walnuthealth.org
TODO: Set up the certificate, and send this BCBSMA
TODO: Confirm they’ve whitelisted it
TODO: Figure out how the Apigee connection works, and hook into it

##### Proposed - 837P Response Testing (999/277/835) via SFTP:
TODO: Investigate how openemr-on-eks receives 999/277/835 files: can these be viewed in the interface? Can they be viewed in a parsed manner?
TODO: Where are these files stored?
TODO: Can these files be accessed?
TODO: For the 835, we need to test very carefully, and also ensure that payment is processed - this needs to be enumerated: let’s circle back here later

##### Proposed - 270 Response Testing (999/271) via HTTPS:
TODO: Investigate how openemr-on-eks receives 999/271: can these be viewed in the interface? Are they parsed correctly?
TODO: Where are these files stored?
TODO: Can these files be accessed?

##### Proposed - 275 / 276 Testing
Let’s pause here for now - we don’t know exactly how this works yet (e.g. maybe its YET ANOTHER transport protocol / system w/ BCBSMA), and the above is PLENTY to start with






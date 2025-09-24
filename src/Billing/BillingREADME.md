# OpenEMR Billing Module Documentation

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Core Components](#core-components)
- [Processing Workflow](#processing-workflow)
- [EDI/X12 Generation](#edix12-generation)
- [File Storage](#file-storage)
- [Database Schema](#database-schema)
- [Common Customization Points](#common-customization-points)
- [Debugging Tips](#debugging-tips)

## Overview

The OpenEMR Billing Module handles the complete lifecycle of medical billing, from claim creation through EDI file generation and submission to insurance payers. The module supports multiple claim formats including X12 837P/I (electronic), HCFA 1500 (paper), and UB04 forms.

### Key Features
- Multi-format claim generation (X12 837P/I, HCFA, UB04)
- Batch processing of multiple claims
- Validation and error checking
- X12 partner configuration
- SFTP/file transfer integration
- ERA (Electronic Remittance Advice) processing

## Architecture

The billing module follows a layered architecture:

```
┌─────────────────────────────────────────────┐
│          UI Layer (interface/billing/)      │
├─────────────────────────────────────────────┤
│     Billing Processor (BillingProcessor)    │
├─────────────────────────────────────────────┤
│     Processing Tasks (Task Classes)         │
├─────────────────────────────────────────────┤
│     Format Generators (X12, HCFA, UB04)     │
├─────────────────────────────────────────────┤
│     Core Models (Claim, BillingUtilities)   │
├─────────────────────────────────────────────┤
│     Database / File Storage Layer           │
└─────────────────────────────────────────────┘
```

## Core Components

### 1. User Interface Layer (`/interface/billing/`)

- **`billing_report.php`**: Main billing management interface
  - Displays claims ready for billing
  - Provides action buttons (Generate, Validate, Clear, etc.)
  - Submits to `billing_process.php`

- **`billing_process.php`**: Entry point for claim processing
  - Receives POST data from UI
  - Instantiates BillingProcessor
  - Displays processing results

- **`edi_270.php`**: Eligibility inquiry interface
- **`edi_271.php`**: Eligibility response viewer
- **`edih_main.php`**: EDI history and file management

### 2. Billing Processor (`/src/Billing/BillingProcessor/`)

#### Core Classes

- **`BillingProcessor.php`**: Main orchestrator
  ```php
  // Entry point for all billing operations
  $processor = new BillingProcessor($_POST);
  $logger = $processor->execute();
  ```
  - Builds appropriate ProcessingTask based on user action
  - Manages claim preparation and processing loop
  - Returns logger with results

- **`BillingClaim.php`**: Individual claim wrapper
  - Encapsulates claim data (patient, encounter, payer)
  - Tracks claim status and actions
  - Provides abstraction over database records

- **`BillingClaimBatch.php`**: Batch file manager
  ```php
  // Manages X12 file generation
  $batch = new BillingClaimBatch('.txt', $context);
  $batch->append_claim($segments);
  $batch->write(); // Outputs to /sites/default/documents/edi/
  ```
  **IMPORTANT**: This class rebuilds ISA/GS segments with its own timestamps, overriding generator output!

- **`BillingClaimBatchControlNumber.php`**: Manages control numbers
  - Generates unique ISA13 interchange control numbers
  - Manages GS06 group control numbers
  - Ensures uniqueness across batches

- **`BillingLogger.php`**: Logging facility
  - Collects processing messages
  - Provides formatted output for UI display
  - Tracks errors and warnings

#### Processing Tasks (`/Tasks/`)

Each task implements `ProcessingTaskInterface` with three methods:

```php
interface ProcessingTaskInterface {
    public function setup(array $context);      // Pre-processing setup
    public function execute(BillingClaim $claim); // Process each claim
    public function complete(array $context);    // Post-processing cleanup
}
```

**Available Tasks:**

- **`GeneratorX12.php`**: Standard X12 837 generation
  - Creates single batch file for all claims
  - Supports both 837P and 837I formats
  - Handles validation-only mode

- **`GeneratorX12Direct.php`**: Per-payer X12 generation
  - Creates separate files per insurance company
  - Useful for direct submissions

- **`GeneratorHCFA.php`**: HCFA 1500 paper forms
- **`GeneratorHCFA_PDF.php`**: HCFA PDF generation
- **`GeneratorUB04*.php`**: UB04 form variants
- **`TaskReopen.php`**: Reopens billed claims
- **`TaskMarkAsClear.php`**: Marks claims as cleared

### 3. Format Generators (`/src/Billing/`)

#### X12 Generators

- **`X125010837P.php`**: Professional claims (837P)
  ```php
  public static function genX12837P($pid, $encounter, $x12_partner, &$log) {
      // Generates X12 segments for professional claims
      // Returns formatted EDI string
  }
  ```
  - Implements HIPAA 5010 format
  - Generates all required loops and segments
  - Handles provider, patient, and service information

- **`X125010837I.php`**: Institutional claims (837I)
  - Similar structure to 837P
  - Different segment requirements for facilities

- **`EDI270.php`**: Eligibility inquiry generator
  - Creates 270 eligibility check files
  - Supports real-time and batch inquiries

#### Core Models

- **`Claim.php`**: Primary claim data model
  ```php
  $claim = new Claim($pid, $encounter, $x12_partner);
  $claim->billingFacilityName();    // Facility info
  $claim->patientData();             // Patient demographics
  $claim->subscriberLastName();     // Insurance info
  $claim->diagArray();               // Diagnosis codes
  $claim->cptArray();                // Procedure codes
  ```
  - Aggregates data from multiple tables
  - Provides consistent interface for generators
  - Handles payer-specific logic

- **`BillingUtilities.php`**: Helper functions
  - Date formatting utilities
  - Code validation
  - Payer ID lookups
  - Common billing calculations

- **`ParseERA.php`**: ERA (835) file parser
  - Processes electronic remittance advice
  - Updates payment information
  - Handles adjustments and denials

## Processing Workflow

### Standard X12 Generation Flow

1. **User Action** (billing_report.php)
   - User selects claims and clicks "Generate X12"
   - Form submits to billing_process.php

2. **Processing Initialization** (billing_process.php)
   ```php
   $billingProcessor = new BillingProcessor($_POST);
   $logger = $billingProcessor->execute();
   ```

3. **Task Building** (BillingProcessor.php)
   ```php
   // Determines which task to create based on form input
   $task = $this->buildProcessingTaskFromPost($post);
   // Examples: GeneratorX12, TaskReopen, etc.
   ```

4. **Claim Preparation**
   - Queries database for selected claims
   - Creates BillingClaim objects
   - Validates claim readiness

5. **Task Execution Loop**
   ```php
   $task->setup($context);
   foreach ($claims as $claim) {
       $task->execute($claim);
   }
   $task->complete($context);
   ```

6. **X12 Generation** (GeneratorX12 -> X125010837P)
   - For each claim:
     - X125010837P::genX12837P() generates segments
     - Returns formatted EDI string with ~\n delimiters

7. **Batch Assembly** (BillingClaimBatch)
   - Receives segments from generator
   - **REBUILDS ISA/GS segments** with batch timestamps
   - Manages control numbers
   - Concatenates claims into single file

8. **File Output**
   - Writes to `/sites/default/documents/edi/[partner]/`
   - Filename: `YYYY-MM-DD-HHmmss-batch.txt`
   - Updates database tracking

9. **Result Display**
   - Logger returns to billing_process.php
   - Results displayed to user

### Validation Flow

When "Validate Only" is selected:
- Same flow as above, but:
- Uses fixed control numbers (000000001)
- Doesn't update claim status
- File saved with validation marker

## EDI/X12 Generation

### Segment Structure

X12 files consist of hierarchical segments:

```
ISA (Interchange Control Header)
  GS (Functional Group Header)
    ST (Transaction Set Header)
      ... claim data segments ...
    SE (Transaction Set Trailer)
  GE (Functional Group Trailer)
IEA (Interchange Control Trailer)
```

### Key Segment Modifications

The system modifies segments at different stages:

1. **Initial Generation** (X125010837P.php)
   - Creates all segments with placeholder data
   - ISA09/10: Uses "030911*1630" placeholders
   - GS04/05: Uses actual DateTime

2. **Batch Processing** (BillingClaimBatch.php)
   - **Overwrites ISA09/10**: Replaces with batch date/time
   - **Overwrites GS04/05**: Replaces with batch date/time
   - **Replaces control numbers**: ISA13, GS06, ST02

### BCBSMA Specific Requirements

For Blue Cross Blue Shield of Massachusetts:
- GS05 time must be HHMMSSDD format (8 digits)
- NM109 should use submitter ID (e.g., "V7BS")
- Addresses need suite/floor numbers in N302
- File must be single-line format (no line breaks between segments)

## File Storage

### Directory Structure
```
sites/default/documents/
├── edi/
│   ├── [partner_name]/     # Per-partner directories
│   │   ├── YYYY-MM-DD-HHmmss-batch.txt
│   │   └── ...
│   └── history/             # Processed files
```

### Database Tables

- **`claims`**: Main claims table
  - Tracks claim status (0=unbilled, 1=billed, 2=bill)
  - Links to patient and encounter
  - Stores payer information

- **`billing`**: Individual service lines
  - CPT/HCPCS codes
  - Modifiers and units
  - Fee amounts

- **`x12_partners`**: Trading partner configuration
  - Partner IDs and names
  - ISA/GS segment values
  - SFTP credentials (encrypted)

- **`x12_remote_tracker`**: EDI file tracking
  - Maps claims to batch files
  - Tracks submission status
  - Stores file locations

## Common Customization Points

### 1. Modifying X12 Segment Format

To change segment formatting (e.g., GS time format):

```php
// In X125010837P.php
$out .= "GS" .
    "*" . "HC" .
    "*" . $claim->x12gsgs02() .
    "*" . trim($claim->x12gs03()) .
    "*" . date('Ymd', $today) .
    "*" . date('His', $today) . substr(microtime(), 2, 2) . // Custom time format
    "*" . "1" .
    "*" . "X" .
    "*" . $claim->x12gsversionstring() .
    "~\n";
```

**BUT BEWARE**: BillingClaimBatch may override your changes!

### 2. Adding New Payer Requirements

1. Extend X12 generator class
2. Override specific segment methods
3. Register in BillingProcessor task builder

### 3. Custom Processing Tasks

Create new task in `/Tasks/`:

```php
class CustomTask extends AbstractProcessingTask {
    public function setup(array $context) {
        // Initialize resources
    }

    public function execute(BillingClaim $claim) {
        // Process each claim
    }

    public function complete(array $context) {
        // Finalize and output
    }
}
```

### 4. SFTP Integration

Configure in x12_partners table:
- x12_sftp_host
- x12_sftp_port
- x12_sftp_username
- x12_sftp_password (encrypted)
- x12_sftp_directory

The X12RemoteTracker handles queuing for transmission.

## Debugging Tips

### 1. Enable Logging

Add to generators:
```php
error_log("GS segment: " . $gs_segment);
file_put_contents('/tmp/x12_debug.log', $out, FILE_APPEND);
```

### 2. Check Database State

```sql
-- View pending claims
SELECT * FROM claims WHERE bill_process = 2;

-- Check X12 file tracking
SELECT * FROM x12_remote_tracker ORDER BY created_at DESC;

-- Review claim details
SELECT c.*, b.* FROM claims c
JOIN billing b ON c.encounter = b.encounter
WHERE c.id = ?;
```

### 3. Common Issues

**Issue**: X12 file format incorrect
- Check both generator AND BillingClaimBatch
- BillingClaimBatch rebuilds segments!

**Issue**: Claims not appearing
- Verify bill_process flag (0, 1, or 2)
- Check x12_partners configuration
- Confirm encounter has billing entries

**Issue**: Validation errors
- Review claim completeness (diagnosis, procedures)
- Verify insurance information
- Check provider NPI/Tax ID

### 4. File Inspection

```bash
# View X12 file with segment breaks
cat file.txt | tr '~' '\n' | less

# Check segment counts
cat file.txt | tr '~' '\n' | grep "^SE\*"

# Verify ISA length (must be 106 chars including ~)
head -c 106 file.txt
```

## Key Interfaces and Extension Points

### Adding New Claim Formats

1. Create generator in `/Tasks/`
2. Implement GeneratorInterface
3. Add to BillingProcessor::buildProcessingTaskFromPost()

### Custom Validation Rules

1. Implement GeneratorCanValidateInterface
2. Override validateClaim() method
3. Add validation logic

### Post-Processing Hooks

1. Extend AbstractProcessingTask
2. Override complete() method
3. Add post-processing logic (SFTP, email, etc.)

## Important Notes

### Timestamp Handling Issue

**CRITICAL**: The BillingClaimBatch class overrides timestamps set by X12 generators. If you modify time formats in X125010837P.php, you MUST also update:
- BillingClaimBatch::__construct() - sets $this->bat_hhmm
- BillingClaimBatch::append_claim() - rebuilds GS segment on line 233

### Partner-Specific Customization (Future Development)

OpenEMR has infrastructure for partner-specific EDI formatting through the `x12_partners` table:
- `processing_format` enum field (standard, medi-cal, cms, proxymed, oa_eligibility, availity_eligibility)
- Partner configuration accessible via `$claim->x12_partner` array
- Processing format available via `$claim->getTarget()` in BillingClaim

**Current BCBSMA Implementation**: Direct code modifications in BillingClaimBatch and X125010837P
**Future Enhancement**: Check partner type and apply conditional formatting:
```php
// Example future implementation
if ($claim->getTarget() === 'bcbsma' || stripos($claim->x12_partner['name'], 'BCBS') !== false) {
    // Apply BCBSMA-specific formatting
    $timeFormat = date('His', $time) . '00';  // HHMMSSDD
} else {
    // Standard formatting
    $timeFormat = date('Hi', $time);  // HHMM
}
```

For now, BCBSMA formatting is hardcoded. To extend for multiple partners later:
1. Add new processing_format enum values
2. Create conditional logic based on partner identification
3. Consider creating partner-specific generator classes

### Control Number Management

- ISA13: Interchange control (9 digits, zero-padded)
- GS06: Group control (increments per functional group)
- ST02: Transaction control (4 digits, zero-padded)

These are managed by BillingClaimBatchControlNumber to ensure uniqueness.

### Performance Considerations

- Batch processing can handle hundreds of claims
- File I/O is the primary bottleneck
- Database queries should be optimized for large claim sets
- Consider pagination for UI display

## Testing

### Manual Testing

1. Create test patient with insurance
2. Add encounters with procedures
3. Process through billing manager
4. Verify X12 output format
5. Check database updates

### Validation Testing

Use "Validate Only" button to test without changing claim status.

### X12 Validation Tools

- Use external X12 validators
- Check with clearinghouse test environments
- Verify with payer-specific tools

## Support Resources

- X12 Standards: https://x12.org/
- HIPAA Implementation Guides
- Clearinghouse documentation
- Payer-specific companion guides

---

*Last Updated: 2024*
*OpenEMR Version: 7.0.4*
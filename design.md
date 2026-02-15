# Design Document: Tomato Price Advisor

## Overview

The Tomato Price Advisor is a mobile-first decision support system that helps small-scale farmers in India evaluate whether a trader's offered price for tomatoes is fair. The system analyzes synthetic historical mandi price data, regional price variations, and market volatility to generate an explainable recommendation classified as Fair, Underpriced, or Risky.

The architecture prioritizes simplicity, low bandwidth requirements, and accessibility for users with limited digital literacy. The system operates primarily as a stateless service with optional local caching for offline capability.

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Mobile Client (PWA)                      │
│  ┌────────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │  Input Form    │  │ Assessment   │  │  Language       │ │
│  │  Validation    │  │ Display      │  │  Selector       │ │
│  └────────────────┘  └──────────────┘  └─────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │           Local Cache (IndexedDB)                      │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ HTTPS/REST
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Assessment Service                         │
│  ┌────────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │  Input         │  │  Price       │  │  Recommendation │ │
│  │  Validator     │  │  Analyzer    │  │  Generator      │ │
│  └────────────────┘  └──────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Price Data Service                         │
│  ┌────────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │  Location      │  │  Historical  │  │  Regional       │ │
│  │  Resolver      │  │  Data Store  │  │  Query Engine   │ │
│  └────────────────┘  └──────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Design Principles

1. **Stateless Operation**: No user accounts or persistent farmer data storage
2. **Offline-First**: Cache regional price data for 24-hour offline access
3. **Low Bandwidth**: Minimize payload sizes, compress responses, support progressive loading
4. **Explainability**: Every classification must include human-readable reasoning
5. **Fail-Safe**: Gracefully handle missing data with clear user communication

## Components and Interfaces

### 1. Mobile Client (Progressive Web App)

**Responsibilities:**
- Render input form with location, price, and date fields
- Validate inputs locally before API calls
- Display assessment results with visual indicators (color-coded classifications)
- Manage local cache for offline operation
- Handle language selection and UI localization

**Key Interfaces:**

```typescript
interface AssessmentRequest {
  location: string;           // District or mandi name
  offeredPrice: number;       // Price in INR per kg or quintal
  date: string;               // ISO 8601 date format
  unit: 'kg' | 'quintal';     // Price unit
  language: 'hi' | 'en' | 'ta' | 'te' | 'kn';
}

interface AssessmentResponse {
  classification: 'Fair' | 'Underpriced' | 'Risky';
  explanation: string;        // Localized explanation
  medianPrice: number;        // Regional median price
  fairPriceRange: {
    min: number;
    max: number;
  };
  dataQuality: 'high' | 'medium' | 'low';
  dataSource: string;         // Description of data used
  dateRange: {
    start: string;
    end: string;
  };
  nearbyMarkets: Array<{
    name: string;
    price: number;
    distance: number;         // km
  }>;
  riskFactors?: string[];     // Present only if classification is Risky
  disclaimer: string;         // Localized disclaimer
}
```

### 2. Assessment Service

**Responsibilities:**
- Orchestrate the price assessment workflow
- Validate input parameters
- Retrieve relevant price data from Price Data Service
- Perform price analysis and classification
- Generate localized explanations

**Key Components:**

**Input Validator:**
- Validates location against known districts/mandis
- Ensures price is positive numeric value
- Checks date is within acceptable range (not future, not older than 7 days)
- Normalizes units (converts quintal to kg if needed)

**Price Analyzer:**
- Calculates regional median price
- Computes fair price range (median ± 15%)
- Calculates price spread across regional markets
- Computes 7-day and 30-day moving averages
- Calculates volatility (standard deviation over 14 days)
- Identifies price trends (declining, stable, rising)

**Recommendation Generator:**
- Applies classification rules based on analysis results
- Generates localized explanations using template system
- Includes relevant risk factors and market context
- Formats output according to language preferences

**Classification Logic:**

```
IF offered_price < fair_range.min - 15%:
  classification = Underpriced
  
ELSE IF offered_price >= fair_range.min AND offered_price <= fair_range.max:
  IF volatility < 20% AND trend != declining:
    classification = Fair
  ELSE:
    classification = Risky
    
ELSE IF offered_price > fair_range.max:
  IF offered_price > fair_range.max + 20% AND volatility > 20%:
    classification = Risky
  ELSE:
    classification = Fair
```

### 3. Price Data Service

**Responsibilities:**
- Store and retrieve synthetic historical mandi price data
- Resolve location queries to specific markets
- Find nearby markets within specified radius
- Aggregate regional price statistics

**Key Components:**

**Location Resolver:**
- Maps district names to associated mandis
- Finds nearest markets using geographic coordinates
- Handles both English and local language location names
- Returns empty result when location is unknown (rather than error)

**Historical Data Store:**
- Stores synthetic daily price records for each mandi
- Schema: `{mandi_id, date, min_price, max_price, modal_price, arrivals}`
- Indexed by mandi_id and date for fast retrieval
- Retains 90 days of historical data

**Regional Query Engine:**
- Retrieves price data for multiple markets in a region
- Calculates aggregate statistics (median, mean, std dev)
- Filters data by date range
- Returns data quality indicators based on completeness

### 4. Translation Service

**Responsibilities:**
- Provide localized text for all user-facing messages
- Support template-based explanation generation
- Handle number and currency formatting per locale

**Supported Languages:**
- Hindi (hi)
- English (en)
- Tamil (ta)
- Telugu (te)
- Kannada (kn)

**Translation Approach:**
- Use template strings with placeholders for dynamic values
- Store translations in JSON files per language
- Fall back to English if translation missing
- Format numbers according to Indian numbering system (lakhs, crores)

## Data Models

### PriceRecord

```typescript
interface PriceRecord {
  mandiId: string;
  mandiName: string;
  district: string;
  state: string;
  date: string;              // ISO 8601 date
  commodity: 'tomato';
  minPrice: number;          // INR per quintal
  maxPrice: number;          // INR per quintal
  modalPrice: number;        // Most common price (INR per quintal)
  arrivals: number;          // Quantity in quintals
  latitude: number;
  longitude: number;
}
```

### RegionalPriceAnalysis

```typescript
interface RegionalPriceAnalysis {
  location: string;
  analysisDate: string;
  priceRecords: PriceRecord[];
  statistics: {
    medianPrice: number;
    meanPrice: number;
    minPrice: number;
    maxPrice: number;
    priceSpread: number;      // max - min
    priceSpreadPercent: number; // (spread / median) * 100
    volatility: number;        // Standard deviation
    volatilityPercent: number; // (volatility / median) * 100
  };
  trends: {
    sevenDayAverage: number;
    thirtyDayAverage: number;
    trendDirection: 'declining' | 'stable' | 'rising';
    trendStrength: number;     // Percentage change
  };
  nearbyMarkets: Array<{
    mandiId: string;
    mandiName: string;
    distance: number;
    currentPrice: number;
  }>;
  dataQuality: {
    level: 'high' | 'medium' | 'low';
    recordCount: number;
    dateRange: {
      start: string;
      end: string;
    };
    missingDays: number;
  };
}
```

### PriceAssessment

```typescript
interface PriceAssessment {
  requestId: string;
  timestamp: string;
  input: AssessmentRequest;
  analysis: RegionalPriceAnalysis;
  classification: 'Fair' | 'Underpriced' | 'Risky';
  fairPriceRange: {
    min: number;
    max: number;
  };
  riskFactors: Array<{
    type: 'high_volatility' | 'declining_trend' | 'high_regional_variation' | 'suspiciously_high';
    severity: 'low' | 'medium' | 'high';
    description: string;
  }>;
  explanation: {
    language: string;
    text: string;
    keyPoints: string[];
  };
  disclaimer: string;
}
```

### CachedRegionalData

```typescript
interface CachedRegionalData {
  location: string;
  cacheTimestamp: string;
  expiryTimestamp: string;    // 24 hours from cache time
  priceData: RegionalPriceAnalysis;
  version: string;            // Cache schema version
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Property Reflection

After analyzing all acceptance criteria, I identified several areas where properties can be consolidated:

**Consolidation Opportunities:**
1. Properties 1.1, 1.2, 1.5 (input validation) can be combined into a comprehensive input validation property
2. Properties 3.1, 3.2, 3.5 (price calculations) test related calculations that can be verified together
3. Properties 4.1, 4.3 (statistical calculations) are independent and should remain separate
4. Properties 5.1, 5.2, 5.3, 5.4 (classification logic) test different classification scenarios and should remain separate for clarity
5. Properties 6.2, 6.3, 6.4 (explanation content) can be combined into one property about explanation completeness
6. Properties 9.1, 9.2, 9.4, 9.5 (transparency requirements) can be combined into one property about required disclosures
7. Properties 11.1, 11.5 (privacy) can be combined into one property about data collection limits
8. Properties 12.1, 12.3, 12.5 (caching behavior) test related caching functionality

**Retained Separate Properties:**
- Input validation edge cases (1.3, 1.4) remain separate as they test specific error conditions
- Data retrieval (2.1, 2.2, 2.3) remain separate as they test different aspects of data fetching
- Regional analysis flags (3.3, 4.2, 4.4) remain separate as they test independent conditional logic
- Classification uniqueness (5.5) remains separate as it's a fundamental constraint
- Language support (7.2, 7.3) remain separate as they test different aspects of localization

### Input Validation Properties

Property 1: Valid input acceptance
*For any* valid combination of location (from predefined list), positive numeric offered price, and date (within past 7 days, not future), the System should accept the input and proceed with assessment.
**Validates: Requirements 1.1, 1.2, 1.5**

Property 2: Invalid price rejection
*For any* input where the offered price is non-numeric, negative, or zero, the System should reject the input and return an error message.
**Validates: Requirements 1.3**

Property 3: Invalid date rejection
*For any* input where the date is in the future or more than 7 days in the past, the System should reject the input and return an error message.
**Validates: Requirements 1.4**

### Data Retrieval Properties

Property 4: Historical data retrieval
*For any* valid location, the System should retrieve price data for that location and nearby markets.
**Validates: Requirements 2.1**

Property 5: Date range correctness
*For any* valid date, the System should retrieve price data spanning exactly 30 days prior to that date.
**Validates: Requirements 2.2**

Property 6: Fallback to nearby markets
*For any* location without direct price data, the System should retrieve data from markets within 100 km.
**Validates: Requirements 2.3**

### Regional Analysis Properties

Property 7: Price statistics calculation
*For any* set of regional price records, the System should correctly calculate the median price, price spread (max - min), and fair price range (median ± 15%).
**Validates: Requirements 3.1, 3.2, 3.5**

Property 8: High regional variation flagging
*For any* regional price data where the price spread exceeds 30% of the median price, the System should flag high regional variation.
**Validates: Requirements 3.3**

Property 9: Nearest markets identification
*For any* location query, the System should return exactly three nearest markets with their current prices, ordered by distance.
**Validates: Requirements 3.4**

### Trend and Volatility Properties

Property 10: Moving average calculation
*For any* price time series, the System should correctly calculate the 7-day and 30-day moving averages.
**Validates: Requirements 4.1**

Property 11: Downward trend detection
*For any* price data where the 7-day average is more than 10% below the 30-day average, the System should flag a downward trend.
**Validates: Requirements 4.2**

Property 12: Volatility calculation
*For any* price time series, the System should correctly calculate volatility as the standard deviation of daily prices over the past 14 days.
**Validates: Requirements 4.3**

Property 13: High volatility flagging
*For any* price data where volatility exceeds 20% of the median price, the System should flag high volatility.
**Validates: Requirements 4.4**

### Classification Properties

Property 14: Fair price classification
*For any* offered price that falls within the fair price range (median ± 15%) with low volatility (<20%) and no declining trend, the System should classify it as Fair.
**Validates: Requirements 5.1**

Property 15: Underpriced classification
*For any* offered price that is more than 15% below the fair price range minimum, the System should classify it as Underpriced.
**Validates: Requirements 5.2**

Property 16: Risky classification for high volatility
*For any* offered price within or above the fair price range but with high volatility (>20%) or declining trend (>10% decline), the System should classify it as Risky.
**Validates: Requirements 5.3**

Property 17: Risky classification for suspiciously high prices
*For any* offered price more than 20% above the fair price range maximum with high volatility, the System should classify it as Risky.
**Validates: Requirements 5.4**

Property 18: Unique classification
*For any* assessment, the System should provide exactly one classification from the set {Fair, Underpriced, Risky}.
**Validates: Requirements 5.5**

### Explanation Properties

Property 19: Explanation completeness
*For any* assessment, the explanation should include the median regional price, and when classified as Underpriced should state the gap amount, and when classified as Risky should list specific risk factors.
**Validates: Requirements 6.2, 6.3, 6.4**

Property 20: Explanation length constraint
*For any* generated explanation, the text should contain between 3 and 4 sentences.
**Validates: Requirements 6.5**

### Localization Properties

Property 21: Language consistency
*For any* selected language preference, all explanations and messages in the response should be in that language.
**Validates: Requirements 7.2**

Property 22: Multi-script location acceptance
*For any* location name provided in English or local language script (Hindi, Tamil, Telugu, Kannada), the System should accept and process it correctly.
**Validates: Requirements 7.3**

### Caching Properties

Property 23: Cache utilization
*For any* repeated request for the same location within 24 hours, the System should use cached data rather than fetching new data.
**Validates: Requirements 8.3**

Property 24: Cache expiry warning
*For any* assessment using cached data older than 24 hours, the System should include a warning that the assessment may be outdated.
**Validates: Requirements 12.3**

Property 25: Cache refresh on reconnection
*For any* transition from offline to online state, the System should automatically refresh cached data in the background.
**Validates: Requirements 12.5**

### Transparency Properties

Property 26: Required disclosures
*For any* assessment response, the System should include a disclaimer stating recommendations are advisory, data source information, the date range used, and clarification that it does not predict future prices.
**Validates: Requirements 9.1, 9.2, 9.4, 9.5**

Property 27: Data quality disclosure
*For any* assessment where data quality is poor or limited (fewer than 20 records or missing more than 5 days), the System should explicitly state this limitation in the response.
**Validates: Requirements 9.3**

### Privacy Properties

Property 28: No PII collection or storage
*For any* assessment request, the System should not collect or store personally identifiable information (names, phone numbers, addresses) and should not persist farmer-specific data beyond the current session.
**Validates: Requirements 10.5, 11.1, 11.5**

Property 29: Anonymized aggregation
*For any* aggregated usage data, all location and price information should be anonymized such that individual farmers cannot be identified.
**Validates: Requirements 11.3**

### Batch Processing Properties

Property 30: Summary completeness
*For any* batch of assessments submitted by an extension worker, the summary view should include all submitted assessments with their classifications.
**Validates: Requirements 10.3**

## Error Handling

### Input Validation Errors

**Invalid Location:**
- Return HTTP 400 with error code `INVALID_LOCATION`
- Message: "Location not recognized. Please select from the list of available districts or mandis."
- Include list of nearby valid locations as suggestions

**Invalid Price:**
- Return HTTP 400 with error code `INVALID_PRICE`
- Message: "Price must be a positive number."

**Invalid Date:**
- Return HTTP 400 with error code `INVALID_DATE`
- Message: "Date must be within the past 7 days and not in the future."

**Missing Required Fields:**
- Return HTTP 400 with error code `MISSING_FIELDS`
- Message: "Please provide location, offered price, and date."
- List specific missing fields

### Data Availability Errors

**No Data Available:**
- Return HTTP 200 with assessment response containing:
  - `dataQuality: 'low'`
  - Explanation stating insufficient data for reliable assessment
  - Suggestion to try again later or contact extension worker

**Insufficient Regional Data:**
- Return HTTP 200 with assessment response containing:
  - `dataQuality: 'medium'`
  - Warning in explanation about limited data
  - Proceed with assessment using available data

### System Errors

**Service Unavailable:**
- Return HTTP 503 with error code `SERVICE_UNAVAILABLE`
- Message: "Service temporarily unavailable. Please try again in a few minutes."
- Client should retry with exponential backoff

**Timeout:**
- Return HTTP 504 with error code `TIMEOUT`
- Message: "Request timed out. Please check your connection and try again."

**Internal Error:**
- Return HTTP 500 with error code `INTERNAL_ERROR`
- Message: "An unexpected error occurred. Please try again."
- Log error details for debugging (without PII)

### Offline Mode Handling

**Cache Available:**
- Use cached data
- Display cache timestamp prominently
- Show offline indicator
- Proceed with assessment

**Cache Expired:**
- Display warning about outdated data
- Show cache age
- Allow user to proceed with stale data or wait for connection

**No Cache Available:**
- Display message: "Cannot perform assessment offline. Please connect to the internet."
- Show last successful connection time if available

## Testing Strategy

### Dual Testing Approach

This system requires both unit testing and property-based testing for comprehensive coverage:

**Unit Tests** focus on:
- Specific examples of each classification type (Fair, Underpriced, Risky)
- Edge cases like empty datasets, single market data, boundary prices
- Error conditions and validation failures
- Integration between components
- Localization for each supported language

**Property-Based Tests** focus on:
- Universal properties that hold across all valid inputs
- Statistical calculations (median, standard deviation, moving averages)
- Classification logic consistency across random inputs
- Data integrity through transformations (caching, serialization)
- Comprehensive input coverage through randomization

### Property-Based Testing Configuration

**Framework Selection:**
- **JavaScript/TypeScript**: Use `fast-check` library
- **Python**: Use `hypothesis` library

**Test Configuration:**
- Minimum 100 iterations per property test
- Each test must reference its design document property
- Tag format: `Feature: tomato-price-advisor, Property {number}: {property_text}`

**Example Property Test Structure:**

```typescript
// Feature: tomato-price-advisor, Property 7: Price statistics calculation
test('median and price spread calculation', () => {
  fc.assert(
    fc.property(
      fc.array(fc.record({
        price: fc.float({min: 1, max: 100}),
        mandi: fc.string()
      }), {minLength: 1, maxLength: 50}),
      (priceRecords) => {
        const analysis = calculateRegionalAnalysis(priceRecords);
        const prices = priceRecords.map(r => r.price).sort((a, b) => a - b);
        const expectedMedian = prices[Math.floor(prices.length / 2)];
        const expectedSpread = Math.max(...prices) - Math.min(...prices);
        
        expect(analysis.statistics.medianPrice).toBeCloseTo(expectedMedian);
        expect(analysis.statistics.priceSpread).toBeCloseTo(expectedSpread);
        expect(analysis.fairPriceRange.min).toBeCloseTo(expectedMedian * 0.85);
        expect(analysis.fairPriceRange.max).toBeCloseTo(expectedMedian * 1.15);
      }
    ),
    { numRuns: 100 }
  );
});
```

### Unit Test Coverage

**Input Validation:**
- Test each validation rule with specific invalid inputs
- Test boundary conditions (exactly 7 days old, exactly 0 price)
- Test missing field combinations

**Classification Logic:**
- Test specific scenarios for each classification
- Example: Price at 50 INR, median 60 INR, low volatility → Underpriced
- Example: Price at 65 INR, median 60 INR, volatility 25% → Risky
- Example: Price at 58 INR, median 60 INR, volatility 10% → Fair

**Localization:**
- Test explanation generation in each supported language
- Verify number formatting (Indian numbering system)
- Test fallback to English when translation missing

**Caching:**
- Test cache hit/miss scenarios
- Test cache expiry at 24-hour boundary
- Test offline mode with and without cache

**Error Handling:**
- Test each error condition produces correct error code and message
- Test graceful degradation with partial data
- Test timeout and retry behavior

### Integration Testing

**End-to-End Flows:**
- Submit valid request → Receive Fair classification
- Submit underpriced offer → Receive Underpriced with explanation
- Submit request with high volatility → Receive Risky with risk factors
- Submit request offline with cache → Receive assessment with cache indicator
- Submit batch requests as extension worker → Receive summary

**Component Integration:**
- Assessment Service ↔ Price Data Service
- Assessment Service ↔ Translation Service
- Mobile Client ↔ API Gateway
- Cache ↔ API calls

### Test Data

**Synthetic Price Data Generation:**
- Generate realistic price distributions (normal distribution with occasional spikes)
- Include seasonal patterns (higher prices in off-season)
- Simulate regional variation (urban vs rural markets)
- Create scenarios with high/low volatility periods
- Include missing data scenarios (gaps in time series)

**Location Data:**
- Maintain list of test districts and mandis with coordinates
- Include locations with varying market density
- Test isolated locations (no markets within 100 km)

### Performance Testing

While not part of correctness properties, performance should be validated:
- Response time under 5 seconds on simulated 2G connection
- Payload size under 50 KB for assessment response
- Cache lookup time under 100ms
- Support for 100 concurrent users

### Accessibility Testing

- Test with screen readers (TalkBack for Android, VoiceOver for iOS)
- Verify text contrast ratios meet WCAG AA standards
- Test touch target sizes (minimum 44x44 pixels)
- Verify keyboard navigation works without mouse

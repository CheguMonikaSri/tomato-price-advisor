# Requirements Document

## Introduction

This document specifies requirements for an AI-powered decision support system that helps small-scale tomato farmers in India evaluate whether a trader's offered price is fair. The system analyzes historical and regional market price data to provide explainable recommendations, empowering farmers to make informed selling decisions while accounting for perishability, regional variation, and market volatility.

## Glossary

- **System**: The tomato price advisor decision support system
- **Farmer**: A small-scale vegetable farmer in India who grows and sells tomatoes
- **Trader**: A village trader or mandi buyer who offers to purchase tomatoes from farmers
- **Mandi**: A regulated agricultural market where produce is traded
- **Offered_Price**: The price per unit (kg or quintal) that a trader proposes to pay the farmer
- **Fair_Price**: A price that falls within an acceptable range based on recent market data and regional trends
- **Underpriced**: A price significantly below the fair market range
- **Risky**: A price that may seem attractive but carries high volatility or uncertainty
- **Price_Assessment**: The system's classification of an offered price as Fair, Underpriced, or Risky
- **Extension_Worker**: An agricultural extension officer or FPO (Farmer Producer Organization) representative who assists farmers
- **Historical_Price_Data**: Synthetic or publicly modeled mandi price records used for analysis
- **Regional_Market**: A set of nearby mandis within a defined geographic radius
- **Volatility**: The degree of price fluctuation in recent market data
- **Price_Spread**: The difference between highest and lowest prices across regional markets

## Requirements

### Requirement 1: Price Input and Context Capture

**User Story:** As a farmer, I want to enter the trader's offered price along with my location and date, so that the system can evaluate whether the price is fair.

#### Acceptance Criteria

1. WHEN a farmer provides location, offered price, and date, THE System SHALL accept and validate these inputs
2. THE System SHALL require location (district or mandi name), offered price (numeric value), and date as mandatory fields
3. WHEN the offered price is non-numeric or negative, THE System SHALL reject the input and display an error message
4. WHEN the date is in the future or more than 7 days in the past, THE System SHALL reject the input and display an error message
5. THE System SHALL accept location as either a district name or a specific mandi name from a predefined list

### Requirement 2: Historical Price Data Retrieval

**User Story:** As a system, I want to retrieve relevant historical mandi price data for the farmer's region, so that I can analyze price trends and fairness.

#### Acceptance Criteria

1. WHEN the System receives a valid location, THE System SHALL retrieve historical price data for that location and nearby markets
2. THE System SHALL retrieve price data for the past 30 days from the specified date
3. WHEN historical data is unavailable for the specified location, THE System SHALL retrieve data from the nearest available markets within 100 km
4. WHEN no data is available within 100 km, THE System SHALL inform the user that analysis cannot be performed
5. THE System SHALL use only synthetic or publicly modeled price data

### Requirement 3: Regional Price Analysis

**User Story:** As a system, I want to analyze price variations across nearby markets, so that I can determine a fair price range for the region.

#### Acceptance Criteria

1. WHEN analyzing regional prices, THE System SHALL calculate the median price across all nearby markets for the past 7 days
2. THE System SHALL calculate the price spread as the difference between the highest and lowest prices in the region
3. WHEN the price spread exceeds 30% of the median price, THE System SHALL flag high regional variation
4. THE System SHALL identify the nearest three markets and their current prices
5. THE System SHALL calculate a fair price range as the median price ± 15%

### Requirement 4: Price Trend and Volatility Assessment

**User Story:** As a system, I want to assess recent price trends and volatility, so that I can identify risky market conditions.

#### Acceptance Criteria

1. WHEN analyzing price trends, THE System SHALL calculate the 7-day moving average and 30-day moving average
2. WHEN the 7-day average is declining by more than 10% compared to the 30-day average, THE System SHALL flag a downward trend
3. THE System SHALL calculate price volatility as the standard deviation of daily prices over the past 14 days
4. WHEN volatility exceeds 20% of the median price, THE System SHALL flag high volatility
5. THE System SHALL consider both trend direction and volatility when assessing risk

### Requirement 5: Price Classification

**User Story:** As a farmer, I want the system to classify the offered price as Fair, Underpriced, or Risky, so that I can quickly understand my situation.

#### Acceptance Criteria

1. WHEN the offered price falls within the fair price range and volatility is low, THE System SHALL classify it as Fair
2. WHEN the offered price is below the fair price range by more than 15%, THE System SHALL classify it as Underpriced
3. WHEN the offered price is within or above the fair price range but volatility is high or prices are declining rapidly, THE System SHALL classify it as Risky
4. WHEN the offered price is significantly above the fair price range (more than 20% higher) and volatility is high, THE System SHALL classify it as Risky
5. THE System SHALL provide exactly one classification per assessment

### Requirement 6: Explainable Recommendation Generation

**User Story:** As a farmer with limited digital literacy, I want to receive a simple explanation of why the price is classified as Fair, Underpriced, or Risky, so that I can understand the reasoning.

#### Acceptance Criteria

1. WHEN generating an explanation, THE System SHALL use simple language appropriate for users with limited digital literacy
2. THE System SHALL include the median regional price in the explanation
3. WHEN the price is classified as Underpriced, THE System SHALL state how much below the fair range it is
4. WHEN the price is classified as Risky, THE System SHALL explain the specific risk factors (volatility, declining trend, or regional variation)
5. THE System SHALL limit explanations to 3-4 short sentences in the user's preferred language
6. THE System SHALL avoid technical jargon and use familiar terms like "market price" instead of "median"

### Requirement 7: Multi-Language Support

**User Story:** As a farmer, I want to receive recommendations in my local language, so that I can understand the system's advice without language barriers.

#### Acceptance Criteria

1. THE System SHALL support Hindi, English, Tamil, Telugu, and Kannada as output languages
2. WHEN a user selects a language preference, THE System SHALL generate all explanations and messages in that language
3. THE System SHALL accept location and mandi names in both English and local language scripts
4. WHEN a translation is unavailable, THE System SHALL fall back to English with a notification
5. THE System SHALL maintain consistent terminology across all supported languages

### Requirement 8: Mobile-First and Low-Bandwidth Design

**User Story:** As a farmer in a rural area with limited internet connectivity, I want the system to work on basic smartphones with slow connections, so that I can access it when needed.

#### Acceptance Criteria

1. THE System SHALL function on mobile devices with screen sizes as small as 4 inches
2. THE System SHALL complete price assessment within 5 seconds on a 2G connection
3. THE System SHALL minimize data transfer by caching historical price data locally when possible
4. THE System SHALL provide a text-based interface that does not require high-resolution graphics
5. WHEN network connectivity is lost during assessment, THE System SHALL display a clear error message and allow retry

### Requirement 9: Transparency and Limitation Disclosure

**User Story:** As a farmer, I want to understand the limitations of the system's recommendations, so that I can make informed decisions without over-relying on the tool.

#### Acceptance Criteria

1. THE System SHALL display a disclaimer stating that recommendations are advisory and not guaranteed
2. THE System SHALL inform users that it uses synthetic or modeled data, not real-time market transactions
3. WHEN data quality is poor or limited, THE System SHALL explicitly state this limitation in the recommendation
4. THE System SHALL clarify that it does not predict future prices or guarantee selling outcomes
5. THE System SHALL display the data source and date range used for the assessment

### Requirement 10: Extension Worker Support Mode

**User Story:** As an extension worker, I want to assist multiple farmers by entering their information and viewing batch assessments, so that I can efficiently support my community.

#### Acceptance Criteria

1. WHEN an extension worker logs in, THE System SHALL provide access to a batch input mode
2. THE System SHALL allow entry of up to 10 farmer assessments in a single session
3. THE System SHALL display a summary view showing all assessments with their classifications
4. THE System SHALL allow the extension worker to export assessment results as a simple text report
5. THE System SHALL maintain farmer privacy by not storing personally identifiable information

### Requirement 11: Data Privacy and Security

**User Story:** As a farmer, I want my location and price information to be kept private, so that traders cannot exploit my data.

#### Acceptance Criteria

1. THE System SHALL not store farmer-specific transaction data beyond the current session
2. THE System SHALL not share individual farmer queries with third parties
3. WHEN aggregating usage data for system improvement, THE System SHALL anonymize all location and price information
4. THE System SHALL not require user registration or personal identification
5. THE System SHALL operate without collecting phone numbers, names, or other personally identifiable information

### Requirement 12: Offline Capability for Recent Data

**User Story:** As a farmer in an area with intermittent connectivity, I want to access recent price assessments even when offline, so that I can make decisions in the field.

#### Acceptance Criteria

1. WHEN the System successfully retrieves regional price data, THE System SHALL cache it locally for 24 hours
2. WHEN operating offline, THE System SHALL use cached data and display the cache timestamp
3. WHEN cached data is older than 24 hours, THE System SHALL warn the user that the assessment may be outdated
4. THE System SHALL clearly indicate when operating in offline mode
5. WHEN returning online, THE System SHALL automatically refresh cached data in the background

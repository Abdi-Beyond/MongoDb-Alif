#
# *DynamoDB Schema Design for Real Estate Application*

This document outlines the DynamoDB schema for a real estate application, designed for high performance and scalability using a query-first, single-table approach. It supports complex search queries, such as filtering by city, deal type, price range, bedrooms, bathrooms, and property type, with examples and scalability considerations.

## 1. Core Design Strategy 🎯

- **Query-First Approach**: The schema is optimized for the application's primary read patterns, ensuring efficient queries without expensive scans.
- **Single Table**: All data is stored in a single DynamoDB table to simplify management and reduce costs.
- **Main Table**: Optimized for writes and direct lookups by propertyID.
- **Global Secondary Indexes (GSIs)**: Tailored for search patterns, using composite keys to support filtering by city, deal type, property type, bedrooms, and bathrooms, sorted by price.
- **Sharding**: Used to prevent hot partitions in high-traffic cities (e.g., New York).

## 2. Main Table: Properties

**Purpose**: Single source of truth for all property data, handling creations, updates, and deletions.

**Primary Key**:

- **Partition Key (PK)**: propertyID (String, UUID)
- **Sort Key (SK)**: createdAt (String, ISO 8601 Timestamp)

**Attributes**:

- Core: city, dealType, propertyType, bedrooms, bathrooms, listingPrice, images, userID
- Composite attributes for GSIs (described below)

**Example Item**:

```json
{
  "propertyID": "uuid-123-abc",
  "createdAt": "2025-09-11T19:45:00Z",
  "city": "NewYork",
  "dealType": "CashDeal",
  "propertyType": "SingleHouse",
  "bedrooms": 2,
  "bathrooms": 1,
  "listingPrice": 300000,
  "userID": "user-456",
  "images": ["img1.jpg", "img2.jpg"],
  "gsi1_pk": "NewYork#CashDeal",
  "gsi2_pk": "NewYork#SingleHouse",
  "gsi3_pk": "NewYork#bed2#bath1",
  "gsi4_pk": "NewYork#CashDeal#SingleHouse#bed2#bath1"
}
```

## 3. Composite Attributes for GSIs

Composite attributes are stored on each item to enable efficient GSI queries by combining multiple fields into a single partition key.

- **gsi1_pk**: {city}#{dealType}
  - Example: NewYork#CashDeal
- **gsi2_pk**: {city}#{propertyType}
  - Example: NewYork#SingleHouse
- **gsi3_pk**: {city}#{bed_bucket}#{bath_bucket}
  - Example: NewYork#bed2#bath1
  - Buckets: bed1, bed2, bed3plus; bath1, bath2plus
- **gsi4_pk**: {city}#{dealType}#{propertyType}#{bed_bucket}#{bath_bucket}
  - Example: NewYork#CashDeal#SingleHouse#bed2#bath1

## 4. Global Secondary Indexes (GSIs)

GSIs power the application's search functionality, optimized for specific query patterns.

### GSI 1: CityDealTypeIndex

- **Purpose**: Find properties by city and deal type, sorted by price.
- **Partition Key**: gsi1_pk
- **Sort Key**: listingPrice
- **Query Example**: Find all Cash Deal properties in New York under $400,000.

  ```javascript
  // AWS SDK for JavaScript v3
  const params = {
    TableName: "Properties",
    IndexName: "CityDealTypeIndex",
    KeyConditionExpression: "gsi1_pk = :pk AND listingPrice <= :price",
    ExpressionAttributeValues: {
      ":pk": "NewYork#CashDeal",
      ":price": 400000
    }
  };
  ```

### GSI 2: CityTypeIndex

- **Purpose**: Find properties by city and property type, sorted by price.
- **Partition Key**: gsi2_pk
- **Sort Key**: listingPrice
- **Query Example**: Find all Single House properties in New York between $20,000 and $400,000.

  ```javascript
  const params = {
    TableName: "Properties",
    IndexName: "CityTypeIndex",
    KeyConditionExpression: "gsi2_pk = :pk AND listingPrice BETWEEN :min AND :max",
    ExpressionAttributeValues: {
      ":pk": "NewYork#SingleHouse",
      ":min": 20000,
      ":max": 400000
    }
  };
  ```

### GSI 3: CityBedBathIndex

- **Purpose**: Find properties by city, bedrooms, and bathrooms, sorted by price.
- **Partition Key**: gsi3_pk
- **Sort Key**: listingPrice
- **Query Example**: Find 2+ bedroom, 1 bathroom properties in New York.

  ```javascript
  const params = {
    TableName: "Properties",
    IndexName: "CityBedBathIndex",
    KeyConditionExpression: "gsi3_pk = :pk AND listingPrice BETWEEN :min AND :max",
    ExpressionAttributeValues: {
      ":pk": "NewYork#bed2#bath1",
      ":min": 20000,
      ":max": 400000
    }
  };
  ```

### GSI 4: CityDealTypePropertyBedBathIndex

- **Purpose**: Support complex queries combining city, deal type, property type, bedrooms, and bathrooms, sorted by price.
- **Partition Key**: gsi4_pk
- **Sort Key**: listingPrice
- **Query Example**: Find Cash Deal, Single House properties with 2 bedrooms and 1 bathroom in New York between $20,000 and $400,000.

  ```javascript
  const params = {
    TableName: "Properties",
    IndexName: "CityDealTypePropertyBedBathIndex",
    KeyConditionExpression: "gsi4_pk = :pk AND listingPrice BETWEEN :min AND :max",
    ExpressionAttributeValues: {
      ":pk": "NewYork#CashDeal#SingleHouse#bed2#bath1",
      ":min": 20000,
      ":max": 400000
    }
  };
  ```

## 5. Example Search Query

**Query**: Find properties in New York with:

- **Deal Type**: \[Cash Deal, SubTo\]
- **Price Range**: $20,000 - $400,000
- **Beds**: &gt;1
- **Bath**: 1
- **Property Type**: \[Single House, Condo/Townhouse, Multi-family 2-4, Multi-family 5+\]

**Using GSI 4 (Recommended)**:

- Query CityDealTypePropertyBedBathIndex for all combinations:
  - Deal Types: CashDeal, SubTo
  - Property Types: SingleHouse, CondoTownhouse, MultiFamily2-4, MultiFamily5plus
  - Bed Buckets: bed2, bed3plus
  - Bath Bucket: bath1
- Total queries: 2 deal types × 4 property types × 2 bed buckets = 16 queries.
- Example query (one of 16):

  ```javascript
  const params = {
    TableName: "Properties",
    IndexName: "CityDealTypePropertyBedBathIndex",
    KeyConditionExpression: "gsi4_pk = :pk AND listingPrice BETWEEN :min AND :max",
    ExpressionAttributeValues: {
      ":pk": "NewYork#CashDeal#SingleHouse#bed2#bath1",
      ":min": 20000,
      ":max": 400000
    }
  };
  ```

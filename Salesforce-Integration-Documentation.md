# OfferCraft Salesforce Integration Documentation

## Table of Contents
- [Overview](#overview)
- [Authentication Mechanism](#authentication-mechanism)
- [Salesforce APIs Used](#salesforce-apis-used)
- [Data Models](#data-models)
- [Integration Operations](#integration-operations)
- [Error Handling & Logging](#error-handling--logging)
- [Configuration](#configuration)
- [Security Considerations](#security-considerations)
- [Code Architecture](#code-architecture)

## Overview

The OfferCraft Salesforce integration provides seamless synchronization between OfferCraft's reward management system and Salesforce CRM. This integration enables unified customer experience management by pushing reward data and managing player information across both platforms.

**Integration Type**: Outbound Only (OfferCraft → Salesforce)
**Architecture**: REST API based with OAuth 2.0 authentication
**Primary Purpose**: Reward delivery, player management, and offer validation

## Authentication Mechanism

### OAuth 2.0 Resource Owner Password Credentials Grant

OfferCraft uses the OAuth 2.0 Password Grant flow to authenticate with Salesforce.

#### Authentication Flow
```csharp
// Authentication request payload
{
    "grant_type": "password",
    "client_id": "{ClientId}",
    "client_secret": "{ClientSecret}",
    "username": "{Username}",
    "password": "{Password + SecurityToken}"
}
```

#### Authentication Details
- **Grant Type**: `password` (OAuth 2.0 Resource Owner Password Credentials)
- **Login Endpoint**: `https://login.salesforce.com/services/oauth2/token` (configurable)
- **Request Method**: `POST`
- **Content Type**: `application/x-www-form-urlencoded`

#### Required Credentials
| Parameter | Description | Example |
|-----------|-------------|---------|
| `ClientId` | Salesforce Connected App Consumer Key | `3MVG9...` |
| `ClientSecret` | Salesforce Connected App Consumer Secret | `1234567890...` |
| `Username` | Salesforce user username | `integration@company.com` |
| `Password` | Salesforce user password + security token | `mypassword{securitytoken}` |

#### Token Management
- **Token Type**: Bearer Token
- **Usage**: `Authorization: Bearer {access_token}`
- **Caching**: Tokens cached per OfferCraft account with configurable lifetime
- **Retry Logic**: Automatic re-authentication on 401 Unauthorized responses
- **Session Lifetime**: Configurable (default based on Salesforce org settings)

#### Authentication Response
```json
{
    "id": "https://login.salesforce.com/id/...",
    "issued_at": "1693747200000",
    "instance_url": "https://yourorg.salesforce.com",
    "signature": "...",
    "access_token": "00D..."
}
```

## Salesforce APIs Used

### Base URL Structure
```
{instance_url}/services/data/{api_version}/
```

### 1. SOQL Query API (`/query`)

#### Player Lookup Query
```sql
SELECT Id, Account_Number__c, Name, FirstName, LastName, PersonEmail, 
       Point_Balance__c, PIN__c, 
       (SELECT Name, Description__c, ImageUrl__c, OffercraftLink__c, 
               PosCode__c, RewardCodeId__c, ExpirationDate__c, 
               RedemptionDate__c, RewardStatus__c 
        FROM OffercraftRewards__r) 
FROM Account 
WHERE Account_Number__c = '{playerId}'
```

#### Offer Validation Query
```sql
SELECT Id 
FROM Offer__c 
WHERE Id IN ({comma_separated_offer_ids})
```

### 2. Standard Object REST API (`/sobjects`)

#### Create Reward Record
```http
POST /sobjects/OffercraftReward__c/
Content-Type: application/json
Authorization: Bearer {access_token}

{reward_data_json}
```

#### Update Player PIN
```http
POST /sobjects/Account/{account_id}?_HttpMethod=PATCH
Content-Type: application/json
Authorization: Bearer {access_token}

{
    "PIN__c": 1234
}
```

## Data Models

### 1. RewardInfo (Pushed to OffercraftReward__c)

```csharp
public class RewardInfo
{
    public string Name { get; set; }                      // Reward title
    
    [JsonProperty("Description__c")]
    public string Description { get; set; }               // Detailed description
    
    [JsonProperty("CustomProperties__c")]
    public JToken CustomProperties { get; set; }          // JSON custom properties
    
    [JsonProperty("ImageUrl__c")]
    public string ImageUrl { get; set; }                  // Reward image URL
    
    [JsonProperty("OffercraftLink__c")]
    public string OffercraftLink { get; set; }            // Deep link to reward
    
    [JsonProperty("PosCode__c")]
    public string PosCode { get; set; }                   // Point-of-sale code
    
    [JsonProperty("RewardCodeId__c")]
    public string RewardCodeId { get; set; }              // Unique reward identifier
    
    [JsonProperty("IsPreview__c")]
    public bool IsPreview { get; set; }                   // Preview/live flag
    
    [JsonProperty("Account__c")]
    public string PlayerSystemId { get; set; }            // Link to Salesforce Account
    
    [JsonProperty("ExpirationDate__c")]
    public DateTime? ExpirationDate { get; set; }         // Reward expiration
    
    [JsonProperty("RedemptionDate__c")]
    public DateTime? RedemptionDate { get; set; }         // When redeemed
    
    [JsonProperty("RewardStatus__c")]
    public string RewardStatus { get; set; }              // Active/Redeemed/Expired
    
    [JsonProperty("OfferRecordId__c")]
    public string OfferReference { get; set; }            // Reference to Offer record
}
```

### 2. PlayerInfo (Retrieved from Account)

```csharp
public class PlayerInfo
{
    [JsonProperty("Id")]
    public string PlayerSystemId { get; set; }            // Salesforce Account ID
    
    [JsonProperty("Account_Number__c")]
    public string PlayerId { get; set; }                  // External player ID
    
    [JsonProperty("Name")]
    public string FullName { get; set; }                  // Player full name
    
    [JsonProperty("FirstName")]
    public string FirstName { get; set; }                 // First name
    
    [JsonProperty("LastName")]
    public string LastName { get; set; }                  // Last name
    
    [JsonProperty("PersonEmail")]
    public string Email { get; set; }                     // Email address
    
    [JsonProperty("PIN__c")]
    public double? PIN { get; set; }                      // Security PIN
    
    [JsonProperty("Point_Balance__c")]
    public double Points { get; set; }                    // Player points balance
    
    [JsonProperty("OffercraftRewards__r")]
    public QueryResponse<RewardInfo> Rewards { get; set; } // Related rewards
}
```

### 3. SalesforceApiSettings (Configuration)

```csharp
public class SalesforceApiSettings
{
    public Guid AccountId { get; set; }                   // OfferCraft account ID
    public string ClientId { get; set; }                  // Connected App Consumer Key
    public string ClientSecret { get; set; }              // Connected App Consumer Secret
    public string Username { get; set; }                  // Salesforce username
    public string Password { get; set; }                  // Password + security token
    public TimeSpan SessionLifetime { get; set; }         // Token cache lifetime
    public string ApiVersion { get; set; }                // Salesforce API version (e.g., "v58.0")
    public string LoginUrl { get; set; }                  // OAuth login endpoint
}
```

## Integration Operations

### 1. Player Lookup (`FindPlayer`)

**Purpose**: Retrieve player information and existing rewards from Salesforce

**Implementation**:
```csharp
public PlayerInfo FindPlayer(string playerId)
{
    var relativeUri = string.Format(
        "/query?q=SELECT Id, Account_Number__c, Name, FirstName, LastName, " +
        "PersonEmail, Point_Balance__c, PIN__c, " +
        "(select Name,Description__c,ImageUrl__c,OffercraftLink__c,PosCode__c," +
        "RewardCodeId__c,ExpirationDate__c,RedemptionDate__c,RewardStatus__c " +
        "from OffercraftRewards__r) " +
        "FROM Account WHERE Account_Number__c = '{0}'", playerId);
    
    // Execute SOQL query and return PlayerInfo
}
```

**Returns**: `PlayerInfo` object with player details and related rewards

### 2. Reward Creation (`Push`)

**Purpose**: Create new reward records in Salesforce for delivered rewards

**Implementation**:
```csharp
public void Push(IEnumerable<Instance> instances)
{
    foreach (var instance in instances)
    {
        RewardInfo rewardModel = GetRewardModel(instance, campaignRewardItems);
        AddPlayerReward(instance.ExternalId, rewardModel);
    }
}

private void Push(RewardInfo rewardModel)
{
    string uri = "/sobjects/OffercraftReward__c/";
    var rewardJson = JsonConvert.SerializeObject(rewardModel);
    DoApiRequest(uri, rewardJson, out resultString);
}
```

**Data Flow**:
1. OfferCraft Instance → RewardInfo mapping
2. Player lookup to get Salesforce Account ID
3. POST to OffercraftReward__c object
4. Tracking and logging

### 3. PIN Update (`UpdatePlayerPin`)

**Purpose**: Update player's security PIN in Salesforce

**Implementation**:
```csharp
public void UpdatePlayerPin(PlayerInfo playerInfo, int pinCode)
{
    var relativeUri = string.Format("/sobjects/Account/{0}?_HttpMethod=PATCH",
        playerInfo.PlayerSystemId);
    var content = JsonConvert.SerializeObject(new { PIN__c = pinCode });
    DoApiRequest(relativeUri, content, out resultString);
}
```

**HTTP Method**: PATCH (via POST with `_HttpMethod=PATCH` parameter)

### 4. Offer Validation (`VerifyOfferIds`)

**Purpose**: Validate that offer IDs exist in Salesforce before processing

**Implementation**:
```csharp
public bool VerifyOfferIds(string[] ids)
{
    var queryIds = string.Join(", ", ids.Select(id => "'" + id + "'"));
    var uri = string.Format("/query?q=SELECT Id FROM Offer__c WHERE Id IN ({0})", queryIds);
    
    // Execute query and verify all IDs are found
    return !ids.Any(id => receivedIds.All(receivedId => !receivedId.StartsWith(id)));
}
```

**Returns**: `true` if all offer IDs exist in Salesforce

## Error Handling & Logging

### Retry Logic
- **Authentication Failures**: Automatic re-authentication on 401 responses
- **Max Retries**: 2 attempts (initial + 1 retry)
- **Token Refresh**: Automatic token refresh on expiration

### Exception Handling
```csharp
try
{
    // Salesforce API operation
}
catch (Exception ex)
{
    Logger.LogException(ex);
    Trace.TraceError("Salesforce operation failed: " + ex);
    throw; // Re-throw for upstream handling
}
```

### Logging Scope
```csharp
using (Logger.InitScope<SalesforceLogEntryCorrelation>(
    ClientAccount.AccountId, 
    nameof(Operation), 
    _internalSettings?.IntegrationSubKey))
{
    // Log requests and responses with encryption option
    Logger.LogRequest(requestData, encryptLogRecord: _internalSettings.EncryptLogRecord);
    Logger.LogResponse(responseData, success, encryptLogRecord: _internalSettings.EncryptLogRecord);
}
```

## Configuration

### Connection Settings
Configuration is stored in `SalesforceApiSettings` and loaded from JSON:

```json
{
    "ClientId": "3MVG9...",
    "ClientSecret": "1234567890...",
    "Username": "integration@company.com",
    "Password": "password{securitytoken}",
    "SessionLifetime": "01:00:00",
    "ApiVersion": "v58.0",
    "LoginUrl": "https://login.salesforce.com/services/oauth2/token"
}
```

### Environment-Specific Settings
- **Production**: `https://login.salesforce.com/services/oauth2/token`
- **Sandbox**: `https://test.salesforce.com/services/oauth2/token`

## Security Considerations

### Data Protection
- **Log Encryption**: Configurable encryption for request/response logs
- **Credential Storage**: Credentials stored securely in configuration
- **Token Caching**: In-memory token cache with expiration
- **HTTPS Only**: All communications over HTTPS/TLS

### Access Control
- **Connected App**: Requires Salesforce Connected App configuration
- **User Permissions**: Integration user must have appropriate object permissions
- **IP Restrictions**: Consider IP restrictions on Connected App (if applicable)

### Required Salesforce Permissions
- **Account**: Read access for player lookups and PIN updates
- **OffercraftReward__c**: Create, Read access for reward management
- **Offer__c**: Read access for offer validation

## Code Architecture

### Key Classes
- **`SalesforceRestService`**: Main integration service class
- **`SalesforceApiSettings`**: Configuration model
- **`RewardInfo`**: Reward data transfer object
- **`PlayerInfo`**: Player data transfer object
- **`AuthResponse`**: OAuth response model

### Dependencies
- **HttpClientService**: HTTP communication layer
- **Logger**: Correlation-based logging system
- **JsonConvert**: JSON serialization (Newtonsoft.Json)
- **ExternalRewardServiceBase**: Base class for external integrations

### Service Registration
The service is registered in the dependency injection container with:
- Account-specific configuration
- Repository dependencies
- Logging dependencies
- Cache services

### Thread Safety
- **Token Caching**: Thread-safe with per-account locking
- **Global Lock**: Prevents race conditions during authentication
- **Concurrent Requests**: Supports multiple concurrent API calls per account

## Monitoring & Maintenance

### Health Checks
- Authentication success/failure rates
- API response times
- Error frequencies by operation type

### Performance Metrics
- Token cache hit rates
- Average response times per operation
- Daily/monthly API usage

### Troubleshooting Common Issues
1. **Authentication Failures**: Check credentials, security token, IP restrictions
2. **Rate Limiting**: Monitor API usage against Salesforce limits
3. **Data Validation**: Ensure required fields are populated
4. **Network Issues**: Check connectivity and firewall rules

---

## OfferCraft Outbound Services Integration Table

| Service Name | Technology/Protocol | Authentication | Data Sent | Purpose | Key Details |
|-------------|-------------------|---------------|-----------|---------|-------------|
| **Salesforce CRM** | REST API / SOQL | OAuth 2.0 Password Grant | Player info, Reward records, PIN updates | CRM integration, Customer 360 view | Custom objects: OffercraftReward__c, Account updates |
| **IGT Gaming System** | XML over HTTP | Basic Auth / API Key | Comp points, Reward redemptions | Gaming system integration | XML messaging, Comp add/void operations |
| **MGT (Gaming Platform)** | REST API | API Key Authentication | Third-party offers, Player data | Offer management integration | GET/POST/DELETE operations for offers |
| **Agilysys (POS/PMS)** | Proxy Integration | Proxy Authentication | Reward data, Transaction info | Hotel/POS system integration | Pull/Push model via proxy service |
| **Synkros (Konami)** | REST API | Certificate-based + API Key | Patron groups, Reward associations | Casino management system | Player group management, Reward tracking |
| **Live Casino Hotel** | REST API | OAuth 2.0 Client Credentials | Lead data, Customer information | Lead generation, Customer acquisition | Token-based authentication with Basic auth |
| **Apple Wallet** | Apple PassKit API | Certificate + Pass Type ID | Pass updates, Device registrations | Mobile wallet integration | PKPass generation, Push notifications |
| **Sightline Payments** | REST API | API Key + Merchant Auth | Card balance inquiries, Profile data | Payment processing integration | Loyalty card balance, Cardholder profiles |
| **SendGrid** | Email API | API Key Authentication | Email templates, Delivery data | Email delivery service | SMTP relay, Template management |
| **SMS Providers** | REST/SMTP APIs | API Key/SMTP Auth | SMS notifications, Delivery status | SMS messaging service | Multiple provider support |
| **Short Link Generator** | Internal REST API | Internal Token | URL shortening requests | Link management | Internal microservice for URL shortening |
| **External File APIs** | HTTP/HTTPS | Various | JSON export files, Email data | Data export and external file delivery | Batch file exports for external systems |

### Integration Patterns by Technology

#### **REST API Integrations** (Most Common)
- **Services**: Salesforce, MGT, Synkros, Live Casino Hotel, Sightline Payments, Apple Wallet
- **Authentication**: OAuth 2.0, API Keys, Certificate-based
- **Data Format**: JSON
- **Operations**: CRUD operations, Real-time data sync

#### **XML-based Integrations**
- **Services**: IGT Gaming System
- **Authentication**: Basic Auth, API Keys
- **Data Format**: XML with custom schemas
- **Operations**: Gaming-specific comp point management

#### **Proxy Integrations**
- **Services**: Agilysys, Some IGT implementations
- **Authentication**: Proxy-mediated authentication
- **Data Format**: JSON/XML (depending on target system)
- **Operations**: Pull/Push models via intermediary services

#### **Email/Messaging Services**
- **Services**: SendGrid, SMS Providers
- **Authentication**: API Keys, SMTP credentials
- **Data Format**: Email templates, JSON for APIs
- **Operations**: Message delivery, Status tracking

### Data Flow Categories

#### **Customer Data Synchronization**
- **Salesforce**: Complete customer profiles and reward history
- **IGT/MGT/Synkros**: Gaming-specific player data and transactions
- **Sightline Payments**: Payment and loyalty card information

#### **Reward Delivery & Management**
- **All Gaming Systems**: Reward issuance and redemption tracking
- **Apple Wallet**: Mobile reward pass delivery
- **External File APIs**: Batch reward data exports

#### **Communication & Notifications**
- **SendGrid**: Email notifications and marketing
- **SMS Providers**: Text message alerts and notifications
- **Apple Wallet**: Push notifications for pass updates

#### **Lead Generation & Marketing**
- **Live Casino Hotel**: Customer lead data for acquisition
- **Short Link Generator**: Trackable marketing links
- **External File APIs**: Marketing data exports

### Security & Compliance

#### **Authentication Methods Used**
- **OAuth 2.0**: Salesforce, Live Casino Hotel (highest security)
- **API Keys**: MGT, SendGrid, SMS providers (standard security)
- **Certificate-based**: Synkros, Apple Wallet (enhanced security)
- **Basic Auth**: IGT (legacy systems)
- **Proxy Auth**: Agilysys (mediated security)

#### **Data Protection**
- **TLS/HTTPS**: All integrations use encrypted transport
- **Token Caching**: OAuth tokens cached with expiration
- **Log Encryption**: Configurable encryption for sensitive data
- **Credential Management**: Secure storage via Azure Key Vault

#### **Monitoring & Reliability**
- **Retry Logic**: Automatic retry for failed requests
- **Circuit Breakers**: Fail-fast patterns for unreliable services
- **Logging**: Comprehensive request/response logging
- **Health Checks**: Service availability monitoring

---

## OfferCraft Inbound Services Integration Table

| Service Name | API Endpoint | Technology/Protocol | Authentication | Data Received | Purpose | Key Details |
|-------------|------------|-------------------|---------------|---------------|---------|-------------|
| **SendGrid Email Webhooks** | `/api/webhooks/rewardDelivery/email` | REST API (JSON) | API Key in URL | Email delivery events, bounce/spam reports | Email delivery tracking & analytics | Processes email status updates from SendGrid |
| **SMS Provider Webhooks** | `/api/webhooks/rewardDelivery/sms` | REST API (JSON) | API Key + Parameters | SMS delivery status, error reports | SMS delivery tracking | Handles SMS delivery confirmations and failures |
| **Apple Wallet PassKit API** | `/api/wallet/v1/*` `/api/passbook/wallet/v1/*` | Apple PassKit Protocol | Certificate + Pass Type ID | Device registrations, pass updates, logs | Mobile wallet management | Apple Wallet device registration and pass updates |
| **Fast API (Reward Redemption)** | `/api/redeem` | REST API (JSON) | Anonymous | Reward redemption requests | External reward redemption | High-performance reward processing endpoint |
| **External Discount API** | `/api/ext/discount/lookup` `/api/ext/discount/apply` | REST API (JSON) | API Key Authentication | POS transaction data, discount requests | POS system integration | Real-time discount calculation and application |
| **Player Verification API** | `/webservices/PMS.asmx` | SOAP Web Service | NTLM/Basic Auth | Player lookup requests, PIN verification | Agilysys POS integration | Legacy SOAP service for player verification |
| **External Emails API** | `/api/emails` | REST API (JSON) | Anonymous | Email file requests | Batch email file delivery | File-based email export system |
| **Portal API (Internal)** | `/api/*` | REST API (JSON) | JWT/Session Auth | Campaign management, reporting data | Internal admin operations | Complete campaign and user management |
| **Passbook API (Legacy)** | `/api/passbook/wallet/v1/*` | Apple PassKit Protocol | Certificate Auth | Pass management requests | Legacy mobile wallet support | Deprecated Apple Wallet integration |
| **External Connector Services** | `/api/ext/*` | REST API (JSON) | API Key Authentication | Gaming system data, player info | Gaming platform integration | Bridge to external gaming systems |
| **MicroServices RPC** | Internal RPC | gRPC/HTTP | Internal Auth | Internal service calls | Inter-service communication | Internal microservices communication |
| **Short Link Generator** | Internal API | REST API (JSON) | Internal Auth | URL shortening requests | Link management service | Internal URL shortening for campaigns |

### Inbound Integration Patterns by Category

#### **Webhook/Callback Services** (Event-Driven)
- **SendGrid Email Webhooks**: Real-time email delivery status updates
- **SMS Provider Webhooks**: SMS delivery confirmation and error handling
- **Apple Wallet PassKit**: Device registration and update notifications

**Pattern**: Event-driven callbacks from external services
**Authentication**: API keys, certificate-based authentication
**Data Flow**: External services → OfferCraft (status updates, confirmations)

#### **External API Endpoints** (Request-Response)
- **Fast API**: High-performance reward redemption processing
- **External Discount API**: POS system discount calculations
- **Player Verification API**: Gaming system player authentication

**Pattern**: Synchronous request-response for real-time processing
**Authentication**: API keys, SOAP authentication
**Data Flow**: External systems → OfferCraft (processing requests)

#### **File/Batch Services** (Asynchronous)
- **External Emails API**: Batch email file delivery
- **External Connector Services**: Gaming platform data synchronization

**Pattern**: Asynchronous batch processing and file transfers
**Authentication**: API keys, anonymous access with validation
**Data Flow**: External systems → OfferCraft (bulk data transfer)

#### **Internal Services** (Microservices)
- **Portal API**: Internal admin and management operations
- **MicroServices RPC**: Inter-service communication
- **Short Link Generator**: Internal utility services

**Pattern**: Internal service mesh communication
**Authentication**: Internal JWT, session-based authentication
**Data Flow**: Internal components ↔ OfferCraft services

### Data Types Received by Category

#### **Delivery & Status Tracking**
- **Email Events**: Delivered, opened, clicked, bounced, spam reports
- **SMS Events**: Delivered, failed, pending status updates
- **Apple Wallet**: Device registrations, pass update requests

#### **Transaction Processing**
- **Reward Redemptions**: Player ID, reward codes, redemption data
- **Discount Calculations**: Receipt data, transaction amounts, item details
- **Player Verification**: Player credentials, PIN codes, account lookups

#### **Batch Data Operations**
- **Email Files**: Bulk email export requests, file downloads
- **Gaming Data**: Player profiles, transaction history, system status

#### **Administrative Operations**
- **Campaign Management**: Campaign configurations, scheduling, reporting
- **User Management**: Authentication, permissions, audit trails

## Comprehensive Inbound Integrations Table

Based on a deep analysis of the OfferCraft codebase, here are all the inbound calls (APIs that external systems call into OfferCraft):

| **Service Name** | **Endpoint/Technology** | **Technology** | **Authentication** | **Data Received** | **Source System** | **Key Details** |
|------------------|-------------------------|----------------|-------------------|-------------------|-------------------|-----------------|
| **SendGrid Email Webhooks** | `/api/webhooks/rewardDelivery/email` | REST API (JSON) | API Key | Email delivery events | SendGrid | Webhook for email delivery status tracking |
| **SMS Webhooks** | `/api/webhooks/rewardDelivery/sms` | REST API (JSON) | Query Parameters | SMS delivery events | SMS Provider | Webhook for SMS delivery status tracking |
| **Google Pay Webhooks** | `/api/googleservice/webHook` | REST API (JSON) | Anonymous | Google Pay wallet events | Google Pay | Mobile wallet pass updates and notifications |
| **Apple Wallet PassKit** | `/api/passbook/wallet/v1/*` | Apple PassKit | Certificate-based | Device registrations, pass updates | Apple Wallet | Mobile wallet pass management and updates |
| **Fast API (Reward Redemption)** | `/api/redeem` | REST API (JSON) | Anonymous | Reward redemption requests | External reward systems | High-performance reward processing endpoint |
| **External Discount Lookup** | `/api/ext/discount/lookup` | REST API (JSON) | API Key | POS receipt data for discount calculation | POS Systems (Agilysys) | Real-time discount calculation for receipts |
| **External Discount Apply** | `/api/ext/discount/apply` | REST API (JSON) | API Key | Payment and discount application data | POS Systems (Agilysys) | Apply calculated discounts to transactions |
| **Player Management SOAP** | `/webservices/PMS.asmx` | SOAP/XML | API Key | Player lookup and verification requests | Property Management Systems | Legacy SOAP service for player operations |
| **External Emails API** | `/api/emails` | REST API (File) | Anonymous | Email file requests and uploads | External email systems | Email content management and delivery |
| **Player Verification API** | `/api/integration/verifyPlayer` | REST API (JSON) | API Key | Player verification requests | External player systems | Player identity verification |
| **Player Lookup API** | `/api/integration/findPlayer` | REST API (JSON) | API Key | Player search requests | External player systems | Player information retrieval |
| **Hotel Check-in API** | `/api/hotels/checkin` | REST API (JSON) | API Key | Hotel check-in data | Hotel management systems | Guest check-in processing |
| **Hotel Check-out API** | `/api/hotels/checkout` | REST API (JSON) | API Key | Hotel check-out data | Hotel management systems | Guest check-out processing |
| **LMS Integration API** | `/api/lmsIntegration` | REST API (JSON) | API Key | Learning management notifications | Learning Management Systems | Training and certification tracking |
| **External Drawings API** | `/api/ext/drawings` | REST API (JSON) | API Key | Drawing and sweepstakes data | External drawing systems | Contest and promotion management |
| **External Instance Redemption** | `/api/fep/rewards` | REST API (JSON) | API Key | External reward redemption requests | External reward platforms | Cross-platform reward processing |
| **Kiosk Demo API** | `/api/kioskDemo/*` | REST API (JSON) | API Key | Kiosk person profile data | Demo kiosks | Demo and testing environment |
| **External Coupon Book API** | `/api/ext/couponBook/*` | REST API (JSON) | API Key | Coupon book requests | External coupon systems | Digital coupon management |
| **External Guest API** | `/api/ext/guest/*` | REST API (JSON) | API Key | Guest profile and activity data | Guest management systems | Guest services integration |
| **External PVL API** | `/api/ext/pvl/*` | REST API (JSON) | API Key | Player verification list data | Player verification systems | Player status and verification |
| **External Settings API** | `/api/ext/settings/*` | REST API (JSON) | API Key | Configuration and settings data | External configuration systems | System configuration management |
| **External Files API** | `/api/ext/files/*` | REST API (File) | API Key | File upload and download requests | External file systems | Document and media management |
| **External Logs API** | `/api/ext/logs/*` | REST API (JSON) | API Key | Log and audit data | External logging systems | System monitoring and audit trails |
| **Portal API (Internal)** | `/api/*` (Portal) | REST API (JSON) | JWT/Session | Various portal operations | OfferCraft Portal | Internal portal management |
| **MicroServices RPC** | Various internal endpoints | gRPC/HTTP | Internal Auth | Service-to-service communication | Internal microservices | Inter-service communication |

### Security & Authentication Methods

#### **API Key Authentication** (Most Common)
- **Services**: SendGrid webhooks, SMS webhooks, External Discount API, External Connector Services
- **Security Level**: Medium - suitable for server-to-server communication
- **Implementation**: URL parameter or header-based API key validation

#### **Certificate-based Authentication** (Highest Security)
- **Services**: Apple Wallet PassKit API
- **Security Level**: High - PKI-based authentication
- **Implementation**: X.509 certificates for Apple Wallet integration

#### **SOAP/Web Service Authentication**
- **Services**: Player Verification API (Agilysys PMS)
- **Security Level**: Medium - legacy authentication methods
- **Implementation**: NTLM, Basic Auth over HTTPS

#### **Internal Authentication**
- **Services**: Portal API, MicroServices RPC
- **Security Level**: High - internal service authentication
- **Implementation**: JWT tokens, session-based authentication

### Performance & Reliability Characteristics

#### **High-Performance Endpoints**
- **Fast API**: Optimized for high-throughput reward processing
- **External Discount API**: Real-time POS transaction processing
- **MicroServices RPC**: Low-latency internal communication

#### **Event Processing**
- **Webhook Services**: Asynchronous event processing with retry logic
- **Apple Wallet**: Real-time device and pass management
- **Batch Services**: Scheduled and on-demand file processing

#### **Monitoring & Logging**
- **Request Tracking**: All inbound requests logged with correlation IDs
- **Error Handling**: Comprehensive error reporting and alerting
- **Rate Limiting**: Protection against abuse and overload

### Integration Architecture Benefits

#### **Real-time Processing**
- Immediate response to external events and transactions
- Live status updates for email, SMS, and payment processing
- Real-time discount calculations for POS systems

#### **Scalable Design**
- Microservices architecture supports horizontal scaling
- Event-driven patterns handle high-volume webhook processing
- Batch processing capabilities for large data operations

#### **Multi-Protocol Support**
- REST APIs for modern integrations
- SOAP services for legacy system compatibility
- Apple PassKit for mobile wallet integration
- gRPC for high-performance internal communication

---

*Last Updated: September 10, 2025*
*Version: 1.1*

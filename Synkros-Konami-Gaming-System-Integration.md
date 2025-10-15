# Synkros (Konami) Gaming System Integration Documentation

## Table of Contents

- [Overview](#overview)
- [Architecture Components](#architecture-components)
- [Communication Protocol](#communication-protocol)
- [API Operations & Commands](#api-operations--commands)
- [Key Integration Features](#key-integration-features)
- [Data Models](#data-models)
- [Database Configuration](#database-configuration)
- [Message Flow Architecture](#message-flow-architecture)
- [Authentication & Security](#authentication--security)
- [Integration Patterns](#integration-patterns)
- [Configuration](#configuration)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)

## Overview

The Synkros (Konami) Gaming System integration in OfferCraft is a **REST API-based integration** that handles patron group management, reward associations, and comprehensive casino management system operations. This integration enables seamless communication between OfferCraft's reward management system and Konami's Synkros gaming platform.

### Key Capabilities

- **Patron Group Management**: Create, manage, and associate patrons with groups
- **Reward Processing**: Point deposits, comp point deposits, and free play awards
- **Real-time Synchronization**: Pull/Push model for data synchronization
- **Comp Management**: Comp inquiries, redemptions, and voucher management
- **Player Data Integration**: Comprehensive player profile and transaction management
- **Certificate-based Security**: Enhanced security with API key authentication

## Architecture Components

### 1. Core Integration Files Structure

```
├── src/OfferCraft.Core.Types/Models/Integration/Synkros/
│   ├── SynkrosApiSettings.cs            # Configuration model
│   ├── SynkrosApiConstants.cs           # API constants and enums
│   ├── SynkrosApiHelper.cs              # Helper utilities
│   ├── MessageBase.cs                   # Base message class
│   ├── AwardFreePlayRequest.cs          # Free play award request
│   ├── PointDepositRequest.cs           # Point deposit request
│   ├── CompPointDepositRequest.cs       # Comp point deposit
│   ├── CompInquiryRequest.cs            # Comp inquiry operations
│   ├── CompRedemptionRequest.cs         # Comp redemption processing
│   ├── PatronGetActiveCardRequest.cs    # Patron card operations
│   ├── CommandViewQueryRequest.cs       # Database view queries
│   └── [30+ additional model files]
├── src/OfferCraft.Core.Types/Enums/Integration/Synkros/
│   ├── CommandLookup.cs                 # API command enumerations
│   ├── ResponseStatusLookup.cs          # Response status codes
│   ├── ClauseOperators.cs               # Query clause operators
│   └── ComparisonOperators.cs           # Query comparison operators
├── utils/OfferCraft.ExternalConnectorService/
│   └── OfferCraft.ExternalConnector.Integrations/Services/
│       └── SynkrosApiService.cs         # Main integration service
└── src/OfferCraft.Integration/Push/
    └── SynkrosPushService.cs            # Push notification service
```

### 2. Service Layer Components

- **SynkrosApiService**: Main production integration service
- **SynkrosPushService**: Push notification and real-time updates
- **SynkrosSettingsProvider**: Configuration and settings management
- **RewardStorageApiDataService**: Base class for reward storage operations

## Communication Protocol

### Protocol Specifications

- **Transport**: HTTPS REST API
- **Data Format**: XML with custom schema validation
- **Authentication**: Certificate-based + API Key
- **Content-Type**: `application/xml` / `text/xml`
- **Encoding**: UTF-8
- **Message Queue**: Azure Service Bus integration

### XML Message Structure

```xml
<?xml version="1.0" encoding="utf-8"?>
<Message Command="Point Deposit" UnitID="CASINO_UNIT_001" Version="1.0">
  <PatronId>12345</PatronId>
  <CardId>CARD_67890</CardId>
  <Amount>1000</Amount>
  <!-- Command-specific content -->
</Message>
```

### Bridge Types

The integration supports two bridge types:

1. **Proxy Bridge**: Direct proxy communication
2. **Apigee Bridge**: API gateway integration

## API Operations & Commands

### Supported Commands (CommandLookup)

| Command | XML Enum | Description | Request Type |
|---------|----------|-------------|--------------|
| **System Check** | `"System Check"` | System health and connectivity verification | `SystemCheckRequest` |
| **Point Deposit** | `"Point Deposit"` | Deposit loyalty points to patron account | `PointDepositRequest` |
| **Comp Point Deposit** | `"Comp Point Deposit"` | Deposit comp points to patron account | `CompPointDepositRequest` |
| **Award Free Play** | `"Award Free Play"` | Award free play credits to patron | `AwardFreePlayRequest` |
| **Comp Inquiry** | `"Comp Inquiry"` | Query patron comp balance and history | `CompInquiryRequest` |
| **Comp Redemption** | `"Comp Redemption"` | Process comp point redemptions | `CompRedemptionRequest` |
| **Comp Voucher List** | `"Comp Voucher List"` | Retrieve patron comp voucher list | `CompVoucherListRequest` |
| **Command View Query** | `"Command View Query"` | Execute database view queries | `CommandViewQueryRequest` |
| **Validate PIN** | `"Validate PIN"` | Validate patron PIN authentication | `PatronRequestBase` |
| **Point Inquiry** | `"Point Inquiry"` | Query patron loyalty point balance | `PatronRequestBase` |
| **Get Patron Address** | `"Get Patron Address"` | Retrieve patron address information | `PatronRequestBase` |
| **Get Patron Email** | `"Get Patron Email"` | Retrieve patron email information | `PatronRequestBase` |
| **Patron Get Active Card** | `"Patron Get Active Card"` | Get patron's active card information | `PatronGetActiveCardRequest` |
| **Alter Patron Group** | `"Alter Patron Group"` | Modify patron group associations | `AlterPatronToGroupAssociationRequest` |

## Key Integration Features

### 1. Patron Management

#### **Patron Group Operations**
- Create and manage patron groups
- Associate patrons with specific groups
- Modify group memberships
- Query group associations

#### **Patron Data Retrieval**
- Get active card information
- Retrieve patron addresses and contact info
- Validate patron PIN authentication
- Query patron account details

### 2. Reward Processing

#### **Point Management**
```csharp
// Point Deposit Example
public class PointDepositRequest : PointDepositRequestBase
{
    public PointDepositRequest()
    {
        this.Command = CommandLookup.PointDeposit;
    }
}
```

#### **Free Play Awards**
```csharp
// Free Play Award with expiration
public class AwardFreePlayRequest : PatronRequestBase
{
    [XmlElement]
    public long Amount { get; set; }

    [XmlElement]
    public long? DaysValid { get; set; } // Max 30 days per Synkros specs
}
```

#### **Comp Point Operations**
- Comp point deposits and withdrawals
- Comp balance inquiries
- Comp redemption processing
- Comp voucher management

### 3. Database View Integration

#### **Supported Views**
```csharp
public static class Views
{
    public const string CompsView = "VW_OFFERCRAFT_COMPS";
}
```

#### **View Columns**
- **Common Columns**: `PTNID` (Patron ID), `CARDID` (Card ID)
- **Comp Columns**: `COMPID`, `COMPNAME`, `COMPTYPE`, `VOUCHERID`, `COMP_VALUE`
- **Offer Columns**: `OFFERID`, `OFFERNAME`, `ISSUE_DATE`, `EXPIRE_DATE`
- **Tier Columns**: `CARDTYPE`, `METERVALUE`, `NEXTLEVELNEED`

### 4. Real-time Processing

The integration supports both **Pull** and **Push** models:

#### **Pull Model**
```csharp
public IEnumerable<ExternalRedeemModel> Get(DateTime startDate, DateTime endDate)
{
    var unredeemedRewards = RewardStorage.GetRewards(r =>
        r.DataSourceType == this.DataSourceType &&
        !string.IsNullOrWhiteSpace(r.PosCode) &&
        DateTime.UtcNow - r.AddedOn >= RedemptionCheckDelay);
    
    return GetRedeemedRewards(unredeemedRewards);
}
```

#### **Push Model**
```csharp
public bool Insert(ExternalRedemptionRewardModel rewardModel)
{
    // Process different discount types
    switch (rewardModel.DiscountInfo?.DiscountType)
    {
        case DiscountTypeLookup.Points:
            return ProcessPointDeposit(rewardModel);
        case DiscountTypeLookup.CompPoints:
            return ProcessCompPointDeposit(rewardModel);
        case DiscountTypeLookup.FreePlay:
            return ProcessFreePlayAward(rewardModel);
        default:
            return ProcessCompRedemption(rewardModel);
    }
}
```

## Data Models

### Core Configuration Model

```csharp
public class SynkrosApiSettings : BaseApiIntegrationSettings
{
    public string Url { get; set; }                        // Synkros API endpoint
    public string UnitId { get; set; }                     // Casino unit identifier
    public string Version { get; set; }                    // API version
    public SynkrosBridgeType SynkrosBridge { get; set; }   // Proxy or Apigee
    
    // Timeout configurations
    public int InternalTimeoutSec { get; set; }            // Message queue timeout
    public int ExternalTimeoutSec { get; set; }            // External service timeout
    
    // Caching configurations
    public int? ProfileCacheLifetimeMinutes { get; set; } = 30;
    public int? RewardsCacheLifetimeMinutes { get; set; } = 15;
    public int? RewardNotificationHandlerDelayMinutes { get; set; } = 2;
    public int? TierUpdateNotificationHandlerDelayMinutes { get; set; }
    
    // Security and messaging
    public AuthorizationSettings Authorization { get; set; }
    public MessageQueueSettings MessageQueue { get; set; }
}
```

### Bridge Type Configuration

```csharp
public enum SynkrosBridgeType
{
    Proxy,      // Direct proxy communication
    Apigee      // API gateway integration
}
```

### Message Base Structure

```csharp
public abstract class MessageBase
{
    [XmlAttribute(AttributeName = "Command")]
    public CommandLookup Command { get; set; }

    [XmlAttribute(AttributeName = "UnitID")]
    public string UnitId { get; set; }

    [XmlAttribute(AttributeName = "Version")]
    public string Version { get; set; }
}
```

### Response Status Handling

```csharp
public class SynkrosApiConstants
{
    public class Statuses
    {
        public const string GOOD = "GOOD";    // Successful operation
        public const string FAIL = "FAIL";    // Failed operation
        public const string ACKD = "ACKD";    // Acknowledged
    }
    
    // Free play constraints
    public const long FreePlayMaxDaysValid = 30;
    
    // Supported date formats
    public static string[] DateTimeFormats = new[]
    {
        "ddd MMM dd HH:mm:ss PST yyyy",
        "yyyy-MM-dd HH:mm:ss.f",
        "yyyy-MM-dd"
    };
}
```

## Database Configuration

### External API Integration Setup

```sql
-- Synkros (Konami) Gaming System Configuration
-- ExternalApiType = 5 identifies Synkros integrations
INSERT INTO [dbo].[ExternalApiTypes] ([ExternalApiTypeId], [Name])
VALUES (5, N'Synkros')

-- Client integration setup
INSERT INTO [dbo].[ExternalApiIntegrations] 
VALUES (
    @ClientAccountID,                           -- Client account identifier
    @PseudoAccountHash + N'.synkros.reward-access.com',  -- Integration key
    5,                                          -- Synkros API type
    @ExternalApiConnection,                     -- JSON configuration
    1,                                          -- Push integration model
    @Login,                                     -- Authentication login
    @Pass                                       -- Authentication password
)
```

### Configuration JSON Structure

```json
{
  "actions": [
    {"type": "RewardIssuance", "model": "Push"},
    {"type": "RewardRedemption", "model": "Pull"}
  ],
  "apiSettings": {
    "monitorEnable": true,
    "workersSettings": [
      {
        "workerType": "Push",
        "sleepInterval": 30,
        "failureSleepInterval": 120,
        "processTimeStep": 1800
      },
      {
        "workerType": "Pull",
        "sleepInterval": 60,
        "failureSleepInterval": 300,
        "processTimeStep": 2400
      }
    ]
  }
}
```

## Message Flow Architecture

### 1. Outbound Processing Flow

```
OfferCraft Rewards → External Connector → Synkros API Service
                                                ↓
                      Synkros Gaming System ← XML Message (HTTPS)
                                                ↓
                      Response Processing ← Synkros Response → Status Update
```

### 2. Inbound Processing Flow (Pull Model)

```
Synkros Gaming System → API Polling → Redemption Status Check
                                           ↓
OfferCraft Database ← Status Update ← Response Processing
```

### 3. Message Queue Integration

```
Azure Service Bus ← Push Notifications ← Synkros System
        ↓
Message Processing → OfferCraft Updates → Real-time Sync
```

## Authentication & Security

### Authentication Methods

#### **Certificate-based Authentication**
```csharp
public class AuthorizationSettings
{
    public CryptoProviderType CryptoProviderType { get; set; }  // Symmetric/Asymmetric
    public string PrivateKeyName { get; set; }                 // Private key identifier
    public string PrivateKeySecret { get; set; }               // Private key secret
    public string PublicKeyName { get; set; }                  // Public key identifier
    public string Password { get; set; }                       // Authentication password
    public string Salt { get; set; }                           // Password salt
}
```

#### **API Key Authentication**
- **UnitId**: Casino unit identifier for API requests
- **Version**: API version specification
- **Secure Headers**: Custom authentication headers

#### **Encryption Support**
```csharp
public enum CryptoProviderType
{
    Symmetric,      // AES, DES encryption
    Asymmetric      // RSA encryption
}
```

### Security Features

- **Message Encryption**: Support for both symmetric and asymmetric encryption
- **Certificate Management**: X.509 certificate-based authentication
- **Secure Transport**: HTTPS-only communication
- **Message Integrity**: XML schema validation and digital signatures
- **Audit Logging**: Comprehensive security event logging
- **Access Control**: Role-based access to API operations

## Integration Patterns

### 1. Push Model Integration

```csharp
public bool Insert(ExternalRedemptionRewardModel rewardModel)
{
    // Validate reward model
    Guard.IsNotNull(rewardModel, nameof(rewardModel));
    
    if (rewardModel.DiscountInfo == null)
        throw new ArgumentException("Reward has no discount information specified");
    
    // Create appropriate request based on discount type
    MessageBase apiRequest = CreateApiRequest(rewardModel);
    
    // Send to Synkros system
    using (var client = new HttpClient(CreateHttpHandler()))
    {
        return SendRequest(client, apiRequest);
    }
}
```

### 2. Pull Model Integration

```csharp
public IEnumerable<ExternalRedeemModel> Get(DateTime startDate, DateTime endDate)
{
    // Get unredeemed rewards from local storage
    var unredeemedRewards = RewardStorage.GetRewards(filter);
    
    // Check redemption status with Synkros system
    return ProcessRedemptionStatus(unredeemedRewards);
}
```

### 3. Message Queue Integration

```csharp
public class MessageQueueSettings
{
    public string ConnectionString { get; set; }       // Azure Service Bus connection
    public string OutboundQueueName { get; set; }      // Outgoing messages queue
    public string InboundQueueName { get; set; }       // Incoming messages queue
}
```

## Configuration

### Production Configuration Example

```json
{
  "SynkrosApiSettings": {
    "Url": "https://synkros-api.casino.com/atr/servlet/ATR",
    "UnitId": "CASINO_UNIT_001",
    "Version": "1.0",
    "SynkrosBridge": "Apigee",
    "InternalTimeoutSec": 30,
    "ExternalTimeoutSec": 60,
    "ProfileCacheLifetimeMinutes": 30,
    "RewardsCacheLifetimeMinutes": 15,
    "RewardNotificationHandlerDelayMinutes": 2,
    "Authorization": {
      "CryptoProviderType": "Asymmetric",
      "PrivateKeyName": "synkros-private-key",
      "PrivateKeySecret": "ENCRYPTED_PRIVATE_KEY",
      "PublicKeyName": "synkros-public-key",
      "Password": "ENCRYPTED_PASSWORD",
      "Salt": "SALT_VALUE"
    },
    "MessageQueue": {
      "ConnectionString": "Endpoint=sb://casino-servicebus.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=KEY",
      "OutboundQueueName": "synkros-outbound",
      "InboundQueueName": "synkros-inbound"
    }
  }
}
```

### Endpoint Configuration

```
Base URL: https://casino.synkros.reward-access.com
API Path: /atr/servlet/ATR?
```

### Worker Configuration

```json
{
  "workersSettings": [
    {
      "workerType": "Push",
      "sleepInterval": 30,           // Real-time push processing
      "failureSleepInterval": 120,   // Quick failure recovery
      "processTimeStep": 1800        // 30-minute processing window
    },
    {
      "workerType": "Pull",
      "sleepInterval": 60,           // Regular polling interval
      "failureSleepInterval": 300,   // Failure recovery delay
      "processTimeStep": 2400        // 40-minute processing window
    }
  ]
}
```

## Testing

### Integration Testing

```csharp
[TestClass]
public class SynkrosTests
{
    [TestMethod]
    public void TestPointDeposit()
    {
        // Test point deposit functionality
        var request = new PointDepositRequest
        {
            UnitId = "TEST_UNIT",
            Version = "1.0",
            PatronId = "12345",
            Amount = 1000
        };
        
        var response = synkrosService.ProcessPointDeposit(request);
        Assert.AreEqual(SynkrosApiConstants.Statuses.GOOD, response.Status);
    }
    
    [TestMethod]
    public void TestFreePlayAward()
    {
        // Test free play award with expiration
        var request = new AwardFreePlayRequest
        {
            UnitId = "TEST_UNIT",
            Version = "1.0",
            PatronId = "12345",
            Amount = 5000,
            DaysValid = 7  // 7 days expiration
        };
        
        var response = synkrosService.ProcessFreePlayAward(request);
        Assert.IsTrue(response.IsSuccessful);
    }
}
```

### Mock Testing Scenarios

1. **Point Deposit Testing**
   - Valid point deposits
   - Insufficient balance handling
   - Invalid patron ID handling
   - Network timeout scenarios

2. **Comp Operations Testing**
   - Comp inquiry operations
   - Comp redemption processing
   - Invalid comp voucher handling
   - Expired comp handling

3. **Free Play Testing**
   - Free play award processing
   - Expiration date validation (max 30 days)
   - Invalid amount handling
   - Patron eligibility validation

4. **Patron Management Testing**
   - Patron group associations
   - Active card retrieval
   - PIN validation
   - Address/email retrieval

## Troubleshooting

### Common Issues and Solutions

#### **1. Certificate Authentication Failures**
```
Error: "Certificate validation failed"
```
**Solutions:**
- Verify certificate installation in certificate store
- Check certificate expiration dates
- Validate private/public key pair matching
- Ensure proper certificate permissions

#### **2. API Response Status Issues**
```
Error: "FAIL status received from Synkros API"
```
**Solutions:**
- Check request XML format compliance
- Validate UnitId and Version parameters
- Verify patron ID exists in Synkros system
- Review error messages in response XML

#### **3. Free Play Expiration Errors**
```
Error: "DaysValid exceeds maximum allowed"
```
**Solutions:**
- Ensure DaysValid ≤ 30 (SynkrosApiConstants.FreePlayMaxDaysValid)
- Validate expiration date calculations
- Check business rule compliance

#### **4. Message Queue Issues**
```
Error: "Azure Service Bus connection failed"
```
**Solutions:**
- Verify Service Bus connection string
- Check queue existence and permissions
- Validate network connectivity to Azure
- Review Service Bus pricing tier limits

### Logging and Monitoring

#### **Key Metrics to Monitor**
- **API Response Time**: Average response time for Synkros API calls
- **Success Rate**: Percentage of successful API operations
- **Queue Depth**: Message queue backlog levels
- **Certificate Expiration**: Certificate validity monitoring
- **Error Rate**: Failed operation percentage by operation type

#### **Logging Configuration**
```csharp
Logger.Info($"Synkros API Service processing {operationType}: PatronId={patronId}");
Logger.Error($"Synkros API error: {response.Error}, PatronId={patronId}");
Logger.Debug($"Request XML: {requestXml}");
Logger.Debug($"Response XML: {responseXml}");
```

### Performance Optimization

#### **Caching Configuration**
```json
{
  "cachingSettings": {
    "profileCacheLifetimeMinutes": 30,      // Patron profile caching
    "rewardsCacheLifetimeMinutes": 15,      // Reward data caching
    "enableDistributedCache": true,         // Multi-instance cache sharing
    "cacheCompressionEnabled": true         // Cache data compression
  }
}
```

#### **Connection Pooling**
```json
{
  "connectionSettings": {
    "maxConcurrentConnections": 20,         // HTTP connection pool size
    "connectionTimeout": 30000,             // 30-second timeout
    "keepAliveEnabled": true,               // HTTP keep-alive
    "retryAttempts": 3,                     // Retry failed requests
    "retryDelay": 5000                      // 5-second retry delay
  }
}
```

## Integration Checklist

### Pre-Implementation
- [ ] Synkros system access credentials obtained
- [ ] Certificate authentication configured
- [ ] Network connectivity to Synkros system established
- [ ] Azure Service Bus configured (if using message queues)
- [ ] Database schema deployed for ExternalApiIntegrations

### Implementation
- [ ] SynkrosApiSettings configured with correct endpoints
- [ ] Certificate-based authentication implemented
- [ ] Message queue integration configured
- [ ] Worker processes configured for Push/Pull operations
- [ ] Error handling and retry logic implemented
- [ ] Comprehensive logging configured

### Testing
- [ ] Unit tests for all API operations implemented
- [ ] Integration tests with Synkros system completed
- [ ] Performance testing under load conducted
- [ ] Certificate expiration monitoring tested
- [ ] Failover and recovery scenarios tested

### Production Deployment
- [ ] Production configuration deployed
- [ ] Certificate management procedures established
- [ ] Monitoring and alerting configured
- [ ] Documentation updated
- [ ] Support team trained on Synkros integration

## Support and Maintenance

### Support Contacts
- **Development Team**: For integration issues and enhancements
- **Konami Synkros Support**: For Synkros system-specific issues
- **Infrastructure Team**: For network, certificates, and Azure Service Bus issues
- **Database Team**: For database view and schema issues

### Maintenance Schedule
- **Daily**: Monitor integration health, certificate validity, and queue depths
- **Weekly**: Review error logs, performance metrics, and cache effectiveness
- **Monthly**: Update documentation, review configuration, and certificate rotation planning
- **Quarterly**: Performance optimization, security review, and disaster recovery testing

### Version History
- **v2.8.7**: Updated Synkros API connection string format
- **v2.8.3**: Added API model and updated integrations for Push model
- **v2.8.0**: Initial Synkros (Konami) Gaming System integration

---

## Key Differences from Other Gaming Integrations

### **Synkros vs IGT Integration**
| Feature | Synkros (Konami) | IGT Gaming System |
|---------|------------------|-------------------|
| **Protocol** | HTTPS REST API | XML over HTTP |
| **Authentication** | Certificate-based + API Key | Basic Auth / API Key |
| **Data Format** | XML with REST transport | Pure XML messaging |
| **Message Queue** | Azure Service Bus | Direct HTTP communication |
| **Caching** | Multi-level caching (30/15 min) | No built-in caching |
| **Bridge Support** | Proxy/Apigee bridges | Direct integration only |
| **Security** | Certificate + Encryption | Basic authentication |

### **Unique Synkros Features**
- **Database View Integration**: Direct access to Synkros database views
- **Patron Group Management**: Advanced patron grouping and association features
- **Certificate-based Security**: Enhanced security with X.509 certificates
- **Message Queue Integration**: Asynchronous processing via Azure Service Bus
- **Multi-bridge Support**: Flexible integration via Proxy or Apigee gateways
- **Advanced Caching**: Configurable multi-level caching for performance optimization

---

*Last Updated: October 8, 2025*  
*Document Version: 1.0*
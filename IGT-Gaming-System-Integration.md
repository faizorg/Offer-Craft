# IGT Gaming System Integration Documentation

## Table of Contents
- [Overview](#overview)
- [Architecture Components](#architecture-components)
- [Communication Protocol](#communication-protocol)
- [API Endpoints & Operations](#api-endpoints--operations)
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

The IGT (International Game Technology) Gaming System integration in OfferCraft is a **legacy XML-based integration** that handles comp point management and reward operations for casino gaming systems. This integration enables seamless communication between OfferCraft's reward management system and IGT's gaming platform.

### Key Capabilities
- **Comp Point Management**: Create, void, issue, and redeem comp points
- **Real-time Processing**: Synchronous XML message processing
- **Batch Operations**: Bulk reward synchronization
- **Gaming System Integration**: Native integration with IGT gaming platforms
- **Mock Server Support**: Complete testing environment included

## Architecture Components

### 1. Core Integration Files Structure
```
├── src/OfferCraft.Core.Types/Models/Integration/IGT/
│   ├── IgtApiSettings.cs              # Configuration model
│   ├── GenericMessageRoot.cs          # Base message container
│   ├── Comp.cs                        # Comp entity model
│   ├── CompBody.cs                    # Comp operation body
│   ├── MessageHeader.cs               # Message metadata
│   └── [30+ additional model files]
├── utils/IGTApiMockServer/             # Complete mock implementation
│   ├── Controllers/
│   │   ├── IGTController.cs           # Main API controller
│   │   └── IGTCompController.cs       # Comp-specific operations
│   ├── Services/
│   │   └── IGTApiMockService.cs       # Mock service implementation
│   └── Repositories/
│       └── IGTStrorageRepository.cs   # Test data storage
└── utils/OfferCraft.ExternalConnectorService/
    └── OfferCraft.ExternalConnector.Integrations/Services/
        └── IgtApiService.cs           # Production service implementation
```

### 2. Service Layer Components
- **IgtApiService**: Main production integration service
- **IGTApiMockService**: Mock server for testing and development
- **IGTStrorageRepository**: Test data persistence layer
- **ExternalDataServiceFactory**: Service factory for IGT integration

## Communication Protocol

### Protocol Specifications
- **Transport**: HTTP/HTTPS
- **Data Format**: XML with custom schema validation
- **Schema**: CRM.xsd for message structure validation
- **Authentication**: Basic Authentication / API Key
- **Content-Type**: `text/xml`
- **Encoding**: UTF-8

### XML Message Structure
```xml
<?xml version="1.0" encoding="utf-8"?>
<CrmAcresMessage xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
                 xsi:noNamespaceSchemaLocation="CRM.xsd">
  <Header>
    <TimeStamp>2024-01-01T12:00:00.000</TimeStamp>
    <Operation Data="Comp" Operand="Add" Command="Update">
      <WhereClause>PlayerId=12345</WhereClause>
      <MaxRecords>100</MaxRecords>
      <TotalRecords>1</TotalRecords>
    </Operation>
  </Header>
  <Body>
    <!-- Operation-specific content -->
  </Body>
</CrmAcresMessage>
```

## API Endpoints & Operations

### Main Controller (`IGTController`)
```csharp
[RoutePrefix("api")]
public class IGTController : ApiController
{
    /// <summary>
    /// Common endpoint for all IGT operations
    /// Processes XML messages and routes to appropriate handlers
    /// </summary>
    [HttpPost]
    [Route("")]
    public HttpResponseMessage ProcessCrmAcresMessage()
}
```

### Comp Management Controller (`IGTCompController`)
```csharp
[RoutePrefix("api/comp")]
public class IGTCompController : ApiController
{
    /// <summary>
    /// Create new comp points for players
    /// </summary>
    [HttpPost, Route("create")]
    public HttpResponseMessage Create([FromBody]GenericMessageRoot<CompBody> crmAcresMessage)

    /// <summary>
    /// Void previously approved comp points
    /// </summary>
    [HttpPost, Route("void")]
    public HttpResponseMessage Void([FromBody]GenericMessageRoot<CompBody> crmAcresMessage)

    /// <summary>
    /// Issue comp points to specific players
    /// </summary>
    [HttpPost, Route("issuance")]
    public HttpResponseMessage Issuance([FromBody]GenericMessageRoot<CompIssuanceBody> crmAcresMessage)

    /// <summary>
    /// Process comp point redemptions
    /// </summary>
    [HttpPost, Route("redemption")]
    public HttpResponseMessage Redemption([FromBody]GenericMessageRoot<CompRedemptionBody> crmAcresMessage)
}
```

## Key Integration Features

### 1. Comp Point Operations

#### **Comp Creation**
- Request and approve comp points for players
- Validate player eligibility
- Generate unique issuance IDs
- Return confirmation with comp details

#### **Comp Voiding**
- Void previously approved comp points
- Require void reason codes
- Maintain audit trail of void operations
- Support bulk void operations

#### **Comp Issuance**
- Issue approved comp points to players
- Track issuance status
- Generate redemption numbers
- Support partial issuance

#### **Comp Redemption**
- Process comp point redemptions
- Validate redemption eligibility
- Update player balances
- Generate redemption receipts

### 2. Supported Message Types

| Message Type | Operation | Description |
|-------------|-----------|-------------|
| `Comp` | Add/Delete | Standard comp operations |
| `PlayerCompListMessage` | Information | Player comp history retrieval |
| `CompIssuance` | Information | Comp issuance processing |
| `CompRedemption` | Information | Redemption transaction processing |
| `SiteInfo` | Information | Gaming site configuration data |

### 3. Player Management
- Player identification and validation
- Gaming profile integration
- Comp balance tracking
- Transaction history maintenance

## Data Models

### Core Configuration Model
```csharp
public class IgtApiSettings : BaseApiIntegrationSettings
{
    public string Url { get; set; }                    // IGT API endpoint
    public string LicenseKey { get; set; }             // IGT license key
    public string VendorName { get; set; }             // Vendor identification
    public string VendorKey { get; set; }              // Vendor authentication key
    public string CurrentUserId { get; set; }          // Current user ID
    public string CurrentUserLoginName { get; set; }   // Current user login
    public string CurrentUserFirstName { get; set; }   // User first name
    public string CurrentUserLastName { get; set; }    // User last name
    public string AuthorizerId { get; set; }           // Authorizer ID
    public string AuthorizerLoginName { get; set; }    // Authorizer login
    public string AuthorizerFirstName { get; set; }    // Authorizer first name
    public string AuthorizerLastName { get; set; }     // Authorizer last name
    public string SiteId { get; set; }                 // Gaming site ID
    public string SiteDescription { get; set; }        // Site description
    public string VoidReasonId { get; set; }           // Default void reason
    public string VoidDescription { get; set; }        // Void description
    public AuthorizationSettings Authorization { get; set; }
    public MessageQueueSettings MessageQueue { get; set; }
}
```

### Message Structure Models
```csharp
// Base message container
public class GenericMessageRoot<T>
{
    public MessageHeader Header { get; set; }
    public T Body { get; set; }
    public string NoNamespaceSchemaLocation { get; set; }
}

// Message header with operation details
public class MessageHeader
{
    public DateTime TimeStamp { get; set; }
    public MessageHeaderOperation Operation { get; set; }
}

// Operation specification
public class MessageHeaderOperation
{
    public OperationDataAttributeLookup Data { get; set; }
    public OperationOperandAttributeLookup Operand { get; set; }
    public string Command { get; set; }
    public string WhereClause { get; set; }
    public int MaxRecords { get; set; }
    public int TotalRecords { get; set; }
}
```

### Comp Entity Model
```csharp
public class Comp
{
    public CompSourceLookup Source { get; set; }
    public string RedemptionNumber { get; set; }
    public UserRole CurrentUser { get; set; }
    public UserRole Authorizer { get; set; }
    public Site Site { get; set; }
    public string VoidReasonId { get; set; }
    public string VoidDescription { get; set; }
    public decimal? Quantity { get; set; }
    public decimal? Value { get; set; }
    public DateTime? ExpirationDate { get; set; }
    public ItemNumber ItemNumber { get; set; }
}
```

## Database Configuration

### External API Integration Setup
```sql
-- IGT Gaming System Configuration
-- ExternalApiType = 3 identifies IGT integrations
INSERT INTO [dbo].[ExternalApiIntegrations] 
VALUES (
    @ClientAccountID,           -- Client account identifier
    'igt.offercraft.com',      -- Integration key
    3,                         -- IGT API type
    @ExternalApiConnection,    -- JSON configuration
    2,                         -- Pull integration model
    @Login,                    -- Basic auth username
    @Pass                      -- Basic auth password
)
```

### Configuration JSON Structure
```json
{
  "actions": [
    {"type": "RewardIssuance", "model": "Pull"},
    {"type": "RewardRedemption", "model": "Pull"}
  ],
  "apiSettings": {
    "monitorEnable": false,
    "workersSettings": [
      {
        "workerType": "Pull",
        "sleepInterval": 60,
        "failureSleepInterval": 60,
        "processTimeStep": 2600
      },
      {
        "workerType": "Push",
        "sleepInterval": 60,
        "failureSleepInterval": 60,
        "processTimeStep": 2600
      },
      {
        "workerType": "LogTransfer",
        "sleepInterval": 120,
        "failureSleepInterval": 120,
        "processTimeStep": 900
      },
      {
        "workerType": "SettingsWatcher",
        "sleepInterval": 60,
        "failureSleepInterval": 60
      }
    ]
  }
}
```

## Message Flow Architecture

### 1. Inbound Message Processing Flow
```
HTTP Request (XML) → IGT Controller → Message Parser → Schema Validation
                                          ↓
Operation Router → Business Logic → Data Processing → Response Generation
                                          ↓
                              XML Response → HTTP Response
```

### 2. Outbound Integration Flow
```
OfferCraft Rewards → External Connector Service → IGT API Service
                                                        ↓
                     IGT Gaming System ← XML Message Transformation
                                                        ↓
                     Response Processing ← IGT Response → Status Update
```

### 3. Detailed Processing Steps

#### **Inbound Processing**
1. **HTTP Request Reception**: Receive XML payload via HTTP POST
2. **Content Validation**: Validate content-type and encoding
3. **XML Parsing**: Parse XML into structured objects
4. **Schema Validation**: Validate against CRM.xsd schema
5. **Operation Extraction**: Extract operation type and parameters
6. **Route to Handler**: Route to appropriate business logic handler
7. **Business Processing**: Execute comp point operations
8. **Response Generation**: Generate structured XML response
9. **HTTP Response**: Return HTTP response with XML content

#### **Outbound Processing**
1. **Reward Synchronization**: Pull new rewards from OfferCraft API
2. **Data Transformation**: Convert rewards to IGT comp format
3. **Message Generation**: Create XML messages for IGT
4. **API Communication**: Send XML messages to IGT system
5. **Response Processing**: Process IGT system responses
6. **Status Update**: Update reward status in OfferCraft
7. **Error Handling**: Handle communication and processing errors

## Authentication & Security

### Authentication Methods

#### **Basic Authentication**
```csharp
// Configuration for basic auth
public class AuthorizationSettings
{
    public string Login { get; set; }        // Username
    public string Password { get; set; }     // Password
    public string Salt { get; set; }         // Password salt
}
```

#### **API Key Authentication**
```csharp
// API key configuration
public class IgtApiSettings
{
    public string LicenseKey { get; set; }   // IGT license key
    public string VendorKey { get; set; }    // Vendor authentication key
}
```

#### **Encryption Support**
```csharp
public enum CryptoProviderType
{
    Symmetric,      // Symmetric encryption (AES, DES)
    Asymmetric      // Asymmetric encryption (RSA)
}

public class AuthorizationSettings
{
    public CryptoProviderType CryptoProviderType { get; set; }
    public string PrivateKeyName { get; set; }
    public string PrivateKeySecret { get; set; }
    public string PublicKeyName { get; set; }
}
```

### Security Features
- **Message Encryption**: Support for symmetric and asymmetric encryption
- **Authentication**: Multiple authentication methods supported
- **Message Integrity**: XML schema validation ensures message integrity
- **Audit Logging**: Comprehensive logging of all operations
- **Secure Transport**: HTTPS support for encrypted communication

## Integration Patterns

### 1. Pull Model Integration
```csharp
public IEnumerable<ExternalRedeemModel> Get(DateTime startDate, DateTime endDate)
{
    // Get unredeemed rewards from local storage
    var unredeemedRewards = RewardStorage.GetRewards(r =>
        r.DataSourceType == this.DataSourceType &&
        !string.IsNullOrWhiteSpace(r.PosCode) &&
        !string.IsNullOrWhiteSpace(r.ExternalRewardId) &&
        DateTimeNow - r.AddedOn >= RedemptionCheckDelay);

    // Check redemption status with IGT system
    return GetRedeemedRewards(unredeemedRewards);
}
```

**Pull Model Characteristics:**
- Periodically queries IGT system for status updates
- Checks redemption status for issued rewards
- Updates OfferCraft with current reward states
- Configurable polling intervals

### 2. Push Model Integration
```csharp
public bool Insert(ExternalRedemptionRewardModel rewardModel)
{
    if (rewardModel.RewardState != "Available")
        return true; // Skip non-available rewards

    // Convert OfferCraft reward to IGT comp format
    var igtCompMessage = ConvertToIgtComp(rewardModel);
    
    // Send to IGT system
    var response = SendToIgtSystem(igtCompMessage);
    
    return response.IsSuccessful;
}
```

**Push Model Characteristics:**
- Immediately sends new rewards to IGT system
- Real-time synchronization of reward data
- Converts OfferCraft rewards to IGT comp format
- Handles immediate response processing

### 3. Message Queue Integration
```csharp
public class MessageQueueSettings
{
    public string ConnectionString { get; set; }    // Queue connection
    public string InboundQueueName { get; set; }    // Incoming messages
    public string OutboundQueueName { get; set; }   // Outgoing messages
}
```

**Queue-based Processing:**
- Asynchronous message processing
- Improved reliability and fault tolerance
- Message persistence during outages
- Load balancing capabilities

## Configuration

### Production Configuration Example
```json
{
  "IgtApiSettings": {
    "Url": "https://igt-api.casino.com/api",
    "LicenseKey": "IGT-LICENSE-KEY-HERE",
    "VendorName": "OfferCraft",
    "VendorKey": "VENDOR-KEY-HERE",
    "CurrentUserId": "SYSTEM_USER",
    "CurrentUserLoginName": "offercraft_api",
    "SiteId": "SITE_001",
    "SiteDescription": "Main Casino Floor",
    "VoidReasonId": "VOID_REASON_001",
    "VoidDescription": "System void",
    "Authorization": {
      "CryptoProviderType": "Symmetric",
      "Password": "ENCRYPTED_PASSWORD",
      "Salt": "SALT_VALUE"
    },
    "MessageQueue": {
      "ConnectionString": "Server=queue-server;Database=IGTQueue;",
      "InboundQueueName": "IGT_Inbound",
      "OutboundQueueName": "IGT_Outbound"
    }
  }
}
```

### Worker Configuration
```json
{
  "workersSettings": [
    {
      "workerType": "Pull",
      "sleepInterval": 60,           // Seconds between pull operations
      "failureSleepInterval": 300,   // Seconds to wait after failures
      "processTimeStep": 2600        // Processing time window
    },
    {
      "workerType": "Push",
      "sleepInterval": 30,           // Immediate push processing
      "failureSleepInterval": 120,   // Quick failure recovery
      "processTimeStep": 1800        // Shorter processing window
    }
  ]
}
```

## Testing

### Mock Server Implementation
The IGT integration includes a complete mock server for testing:

```csharp
// Mock server controller
[RoutePrefix("api")]
public class IGTController : ApiController
{
    private IGTApiMockService _igtApiMockService;
    
    [HttpPost, Route("")]
    public HttpResponseMessage ProcessCrmAcresMessage()
    {
        var xmlDoc = Request.ReadXDocument();
        return _igtApiMockService.ProcessCrmAcresMessage(xmlDoc);
    }
}
```

### Test Database Configuration
```xml
<!-- IGT Mock Server Web.config -->
<connectionStrings>
  <add name="IGTStorageConnection" 
       connectionString="Data Source=(LocalDB)\MSSQLLocalDB;AttachDbFilename=|DataDirectory|\MSSqlStorage.mdf;Integrated Security=True;Connect Timeout=30" 
       providerName="System.Data.SqlClient" />
</connectionStrings>
```

### Testing Scenarios
1. **Comp Creation Testing**
   - Valid comp creation requests
   - Invalid player ID handling
   - Insufficient authorization testing
   - Duplicate comp prevention

2. **Comp Void Testing**
   - Valid void operations
   - Invalid void reason handling
   - Already voided comp handling
   - Unauthorized void attempts

3. **Integration Testing**
   - End-to-end message flow
   - Error handling scenarios
   - Timeout handling
   - Network failure recovery

## Troubleshooting

### Common Issues and Solutions

#### **1. XML Schema Validation Errors**
```
Error: "Incoming xml message do not have required header operation node"
```
**Solution:** Ensure XML message includes proper header structure:
```xml
<Header>
  <Operation Data="Comp" Operand="Add" Command="Update">
    <!-- Operation details -->
  </Operation>
</Header>
```

#### **2. Authentication Failures**
```
Error: "Unauthorized access"
```
**Solutions:**
- Verify Basic Auth credentials
- Check API key validity
- Ensure proper encryption settings
- Validate certificate configuration

#### **3. Connection Timeout Issues**
```
Error: "Request timeout"
```
**Solutions:**
- Increase HTTP timeout values
- Check network connectivity
- Verify IGT system availability
- Review firewall settings

#### **4. Message Processing Errors**
```
Error: "Unknown headerOperation params"
```
**Solutions:**
- Verify supported operation types
- Check message format compliance
- Validate enum values
- Review XML schema compliance

### Logging and Monitoring

#### **Enable Detailed Logging**
```csharp
// In IgtApiService constructor
var loggingHandler = CreateHttpHandler(ConnectorCommonSettings.HttpRequestLogType);
Logger.Info($"IGT API Service initialized for {DataSourceType}");
```

#### **Key Metrics to Monitor**
- **Message Processing Rate**: Messages processed per minute
- **Error Rate**: Percentage of failed operations
- **Response Time**: Average API response time
- **Queue Depth**: Message queue backlog
- **Authentication Failures**: Failed authentication attempts

### Performance Optimization

#### **Recommended Settings**
```json
{
  "performanceSettings": {
    "batchSize": 100,              // Messages per batch
    "connectionPoolSize": 10,      // HTTP connection pool
    "requestTimeout": 30000,       // 30 second timeout
    "retryAttempts": 3,           // Retry failed requests
    "retryDelay": 5000            // 5 second retry delay
  }
}
```

## Integration Checklist

### Pre-Implementation
- [ ] IGT system access credentials obtained
- [ ] Network connectivity established
- [ ] SSL certificates configured (if required)
- [ ] Database schema deployed
- [ ] Configuration values set

### Implementation
- [ ] IgtApiSettings configured
- [ ] Authentication method selected and configured
- [ ] Message queue configured (if using)
- [ ] Worker processes configured
- [ ] Error handling implemented
- [ ] Logging configured

### Testing
- [ ] Mock server tests passing
- [ ] Unit tests implemented
- [ ] Integration tests completed
- [ ] Performance testing conducted
- [ ] Security testing performed

### Production Deployment
- [ ] Production configuration deployed
- [ ] Monitoring configured
- [ ] Alerting set up
- [ ] Documentation updated
- [ ] Support team trained

---

## Support and Maintenance

### Support Contacts
- **Development Team**: For integration issues and enhancements
- **IGT Support**: For IGT system-specific issues
- **Infrastructure Team**: For network and security issues

### Maintenance Schedule
- **Daily**: Monitor integration health and performance
- **Weekly**: Review error logs and performance metrics
- **Monthly**: Update documentation and review configuration
- **Quarterly**: Performance optimization and security review

### Version History
- **v2.9.9.7**: Updated IGT ExternalApiIntegration settings
- **v2.9.9.4**: Added IGT proxy integration support
- **v2.9.5**: Initial IGT Gaming System integration

---

*Last Updated: October 8, 2025*
*Document Version: 1.0*
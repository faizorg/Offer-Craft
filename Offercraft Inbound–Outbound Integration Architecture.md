# Offercraft Inbound–Outbound Integration Architecture

This document describes how **Offercraft** integrates with multiple inbound and outbound systems via a centralized **Application Integration Services** layer.

---

## 🖼️ Architecture Diagram

![Offercraft Integration Architecture](Application_Integration_Diagram.png)

*(Figure 1: Offercraft Inbound–Outbound Integration Flow)*

---

## 🧩 Inbound Services
These services send data **into** the Offercraft ecosystem:

- **SendGrid Email Webhooks** – captures and processes email event data.  
- **Player Verification API** – verifies player identity or credentials.  
- **Player Lookup API** – retrieves player details or account information.  
- **Hotel Check-in/out API** – syncs hotel stay data with Offercraft.  
- **LMS Integration API** – integrates with loyalty management systems.  

All inbound traffic flows through the **Application Integration Services** layer before reaching Offercraft microservices.

---

## ⚙️ Application Integration Services
This acts as the **middleware or gateway layer**, managing all communication between Offercraft microservices and external systems.

Responsibilities include:
- Routing requests between internal and external services  
- Data transformation and mapping  
- Security enforcement and rate limiting  
- Logging and monitoring  

> _In a real-world setup, this layer could be implemented using API Gateway, Ocelot, or any other integration middleware._

---

## 🧱 Offercraft Microservices
These are the core backend services that handle business logic and operations:

- **Identity Service** – manages user authentication and identity.  
- **Reporting Service** – generates reports and analytics.  
- **Campaign Service** – manages campaign-related operations.  
- **Communication Service** – handles messaging and notifications.  
- **...and others** – additional supporting services as needed.

All microservices interact with external systems via the **Application Integration Services** layer.

---

## 🔗 Outbound Services
These are external systems that receive data or are updated by Offercraft:

- **IGT Gaming System**  
- **Synkros (Konami)**  
- **Live Casino Hotel**  
- **Short Link Generator**  
- **External File APIs**

Outbound requests are routed through the **Application Integration Services** for consistent integration and monitoring.

---

## 🔄 Data Flow Overview
1. **Inbound Services** → send data → **Application Integration Services**  
2. **Integration Layer** → routes to → **Offercraft Microservices**  
3. **Offercraft Microservices** → send responses → **Integration Layer**  
4. **Integration Layer** → pushes updates → **Outbound Services**

---

## ✅ Summary
This architecture follows a **hub-and-spoke integration model**, where the **Application Integration Services** serve as the central hub.  
It ensures that all inbound and outbound communication with Offercraft passes through a unified, secure, and manageable integration layer — enhancing scalability, traceability, and performance.

---


# Standards & API Reference

> Project: Electric Utility Customer Portal · Generated: 2026-05-07

---

## Industry Standards & Specifications

### IEC / ISO Standards

**IEC 61968 — Application Integration at Electric Utilities (Distribution Management)**
- URL: https://www.iec.ch/tc57 and https://en.wikipedia.org/wiki/IEC_61968
- The IEC 61968 series defines the Common Information Model (CIM) extensions for distribution management, covering asset management, work management, meter reading and control, geographic location, and customer information systems. Part 9 (IEC 61968-9:2024) specifies the information content for meter reading and control message types including customer data synchronisation and switching. Part 11 defines CIM extensions for distribution; Part 13 defines the CIM RDF exchange format. Essential for interoperability between CIS, OMS, GIS, MDMS, and the customer portal.

**IEC 61970 — Energy Management System Application Program Interface (EMS-API)**
- URL: https://webstore.iec.ch/en/publication/62698 (IEC 61970-301:2020)
- Defines the CIM base model for energy management system APIs. IEC 61970-301 describes the CIM as an abstract object-oriented model representing all major objects in an electric utility enterprise. Used as the semantic backbone for grid-operational data exchanged with OMS and SCADA that feeds real-time outage information into the customer portal.

**IEC 62056 — Electricity Metering Data Exchange (DLMS/COSEM)**
- URL: https://www.dlms.com/core-specifications/ and https://en.wikipedia.org/wiki/IEC_62056
- The DLMS/COSEM suite specifies the transport and application layers for smart-meter communication. IEC 62056-21 covers direct local data exchange; IEC 62056-61 covers the OBIS object identification system for meter registers. Although primarily a meter-to-head-end standard, interval data flowing from smart meters through the HES and MDMS ultimately originates in DLMS/COSEM format. The customer portal must handle latency and data-quality characteristics arising from this pipeline.

**IEC 62325 — Energy Market Communications**
- URL: https://webstore.iec.ch/searchpage/searchresult.aspx?SearchText=62325
- Defines CIM extensions for electricity market business processes including forecasting, bidding, scheduling, and settlement. Relevant for utilities operating in deregulated markets where customer portal data must reflect wholesale price signals or time-of-use rate schedules derived from market prices.

---

### W3C & IETF Standards

**NAESB REQ.21 — Energy Services Provider Interface (ESPI) / Green Button**
- URL: https://www.naesb.org/espi_standards.asp · https://www.greenbuttonalliance.org/green-button-connect-my-data-cmd
- The North American Energy Standards Board (NAESB) REQ.21 ESPI standard (current version: 4.0, December 2023) is the formal basis for Green Button Connect My Data (CMD). It defines a REST API and data model for customer-authorised sharing of interval energy and billing data. Uses Atom Syndication Format for data serialisation and OAuth 2.0 for authorisation. All customer portals targeting US utilities should expose a Green Button CMD API. Minimum TLS version required is TLS 1.3 as of v4.0.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Defines the OAuth 2.0 authorisation framework underpinning Green Button CMD and all third-party data-access flows. The customer portal acts as both Resource Server (protecting customer data) and Client (accessing back-end APIs). OAuth 2.0 authorisation code flow with PKCE is mandatory for browser-based portal flows.

**RFC 6750 — Bearer Token Usage**
- URL: https://www.ietf.org/rfc/rfc6750.txt
- Specifies how OAuth 2.0 bearer tokens are transmitted in HTTP requests. Required companion to RFC 6749 for all protected API calls between portal and back-end services.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Builds on OAuth 2.0 to provide identity-layer authentication, enabling the portal to verify customer identity without storing credentials. Recommended for customer login flows alongside MFA to reduce account-takeover risk.

**W3C Payment Request API**
- URL: https://www.w3.org/TR/payment-request/
- W3C Candidate Recommendation defining a browser-native API for payment checkout. Enables streamlined card and digital-wallet payment flows in the bill-payment module. Reduces friction for customers paying on mobile devices.

**W3C Web Content Accessibility Guidelines (WCAG) 2.1 Level AA**
- URL: https://www.w3.org/WAI/standards-guidelines/wcag/
- The accessibility standard mandated by ADA Title II for public-facing utility web portals. Level AA covers perceivable, operable, understandable, and robust requirements. US utilities serving populations ≥50,000 must comply by April 2027 per the 2026 DOJ Interim Final Rule. The customer portal must pass third-party WCAG 2.1 AA audits covering screen reader support, keyboard navigation, colour contrast, and labelled form fields.

---

### Data Model & API Specifications

**Green Button Connect My Data (CMD) — REST API**
- URL: https://green-button.github.io/developers/ · https://greenbuttonalliance.github.io/OpenESPI-GreenButton-API-Documentation/API/
- The Green Button CMD REST API provides customer-authorised access to interval usage data (UsagePoint, MeterReading, IntervalBlock resources) in Atom/XML format following the NAESB REQ.21 ESPI schema. The Green Button Alliance operates a developer sandbox for testing CMD implementations. Open-source reference implementation: GreenButtonAlliance/OpenESPI-GreenButton-API-Documentation on GitHub.

**OpenAPI Specification (OAS) 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- De-facto standard for documenting REST APIs. The customer portal's public-facing REST APIs (usage data, account management, outage reporting) should be documented in OAS 3.1 to enable SDK generation and third-party integration.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/
- Used for validating request and response payloads in portal REST APIs. Ensures data-quality constraints on usage data, account records, and outage reports returned from back-end systems.

---

### Energy & Demand-Response Standards

**OpenADR 2.0b / 3.0 — Open Automated Demand Response**
- URL: https://www.openadr.org/specification · https://www.openadr.org/openadr-3-0
- OpenADR defines a communication standard for automated demand response (DR) signals between utilities/ISOs and customer devices. Version 2.0b (most widely deployed globally) uses OASIS Energy Interoperation 1.0 data models over HTTP/XMPP. Version 3.0 (2024) introduces a dedicated REST API for Virtual End Nodes (VENs), improved smart-charging integration, and DER control. The customer portal's demand-response enrolment and device management modules should implement OpenADR 3.0 VEN-side communications. Open-source Python implementation: OpenLEADR (https://openleadr.org/docs/).

**OCPP 2.1 — Open Charge Point Protocol**
- URL: https://openchargealliance.org/protocols/open-charge-point-protocol/
- Published January 2025, OCPP 2.1 defines the communication protocol between EV chargers and central management systems. Includes V2G (vehicle-to-grid) and DER control functional blocks. The customer portal's EV smart-charging scheduler integrates with OCPP 2.1 Central System APIs to retrieve charging session data, set charging schedules, and enrol chargers in DR events. Relevant for utilities offering EV rate plans and managed-charging programmes.

**OASIS Energy Interoperation 1.0**
- URL: https://docs.oasis-open.org/energyinterop/ei/v1.0/
- The foundational data model for demand-response event signals, underpinning OpenADR 2.0. Defines EiEvent, EiReports, and EiEnroll payloads. Referenced in OpenADR 2.0b profile specification.

---

### Security & Compliance Standards

**NERC CIP (Critical Infrastructure Protection) Standards**
- URL: https://www.nerc.com/standards/reliability-standards/cip
- Fourteen mandatory standards (CIP-002 through CIP-015) governing cybersecurity of the North American Bulk Electric System. Although NERC CIP primarily governs operational technology (SCADA, EMS), the customer portal must not expose pathways into CIP-classified assets. OMS integration in the portal must surface only customer-relevant outage data without exposing internal SCADA system access.

**PCI DSS 4.0.1 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/standards/
- Mandatory for any system storing, processing, or transmitting cardholder data. The portal's bill-payment module must comply with PCI DSS 4.0.1 (mandatory as of 2026), including continuous monitoring, third-party script controls, and third-party service provider oversight. Tokenisation and hosted payment fields are recommended to reduce PCI scope.

**NIST SP 800-63B — Digital Identity Guidelines (Authentication)**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- Defines assurance levels for digital authentication. The customer portal login flow should target NIST AAL2 (multi-factor authentication) to mitigate account-takeover fraud. Relevant for password-reset flows, MFA enrolment, and session-management policies.

**OWASP Top 10 (2021)**
- URL: https://owasp.org/www-project-top-ten/
- Industry-standard guidance on the most critical web application security risks. All portal components should be assessed against the OWASP Top 10, particularly injection, broken access control, and security misconfiguration given the sensitivity of customer billing and energy-usage data.

---

## Similar Products — Developer Documentation & APIs

### Oracle Utilities Customer Cloud Service
- **Description:** Enterprise cloud CIS with REST APIs for customer care, billing, service orders, meter data management, and digital self-service. Used by large investor-owned utilities globally.
- **API Documentation:** https://docs.oracle.com/en/industries/utilities/customer-cloud-service/restapi.html
- **Digital Self-Service REST API:** https://docs.oracle.com/en/industries/energy-water/digital-self-service/restapi/index.html
- **CC&B REST Endpoints:** https://docs.oracle.com/en/industries/energy-water/framework/4.5.0.0.0/restapi/api-v-model-service-points-ccb.html
- **Developer Guide:** https://docs.oracle.com/en/industries/utilities/customer-cloud-service/index.html
- **Standards:** REST/JSON, OpenAPI 3.x
- **Authentication:** Oracle Identity Cloud Service (OAuth 2.0 / OpenID Connect)

### KUBRA (EZ-PAY / MyHQ / Storm Center)
- **Description:** Customer experience management platform for utilities, covering bill payment, outage mapping, notifications, and customer communications. Deployed at 300+ North American utilities.
- **API Documentation:** https://developer.kubra.com/
- **Developer Center:** https://developer.kubra.com/ (API access for iMail, Notifi, Storm Center, and EZ-PAY products)
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0, API key

### UtilityAPI
- **Description:** Third-party data aggregator that provides standardised, customer-authorised access to billing and interval-usage data from US electric utilities via Green Button CMD and direct scraping. Enables third-party apps and energy efficiency programmes to access utility data.
- **API Documentation:** https://utilityapi.com/docs/api
- **Green Button Docs:** https://utilityapi.com/docs/greenbutton
- **Developer Guide:** https://utilityapi.com/docs
- **Standards:** REST/JSON, Green Button CMD (NAESB REQ.21 ESPI)
- **Authentication:** OAuth 2.0

### NREL / OpenEI Utility Rates API
- **Description:** US Department of Energy / National Renewable Energy Laboratory API providing access to the Utility Rate Database (URDB), covering rate structures for 3,700+ US utilities. Enables rate-comparison and cost-projection features in customer portals. Note: API domain migrating from developer.nrel.gov to developer.nlr.gov in 2026.
- **API Documentation:** https://developer.nlr.gov/docs/electricity/openei-utility-rates/
- **Rate Database:** https://openei.org/wiki/Utility_Rate_Database
- **Standards:** REST/JSON
- **Authentication:** API key (free registration)

### Bidgely UtilityAI (Disaggregation as a Service — DaaS)
- **Description:** AI-powered energy analytics platform offering appliance-level disaggregation from smart-meter interval data as an API service. Enables customer portals to display per-appliance usage breakdowns and personalised recommendations.
- **DaaS API Webinar / Resources:** https://www.bidgely.com/resources/data-driven-partnerships-daas-on-demand-webinar-resources/
- **GitHub Organisation:** https://github.com/bidgely
- **Standards:** REST/JSON; proprietary data model for disaggregation results
- **Authentication:** OAuth 2.0 / API key (by arrangement)

### Uplight Customer Portals
- **Description:** AI/ML-powered utility-customer engagement platform providing personalised energy insights, programme enrolment, home energy reports, and marketplace features. Deployed at 80+ US utilities.
- **Product Overview:** https://uplight.com/solutions/utility-energy-customer-portals/
- **Note:** Uplight APIs are enterprise integrations; no public developer documentation. Contact required for partnership.
- **Standards:** REST/JSON; widget embedding via JavaScript SDK
- **Authentication:** OAuth 2.0 (enterprise)

### Open International Smartflex
- **Description:** Unified CIS + billing + self-service portal platform for electric, gas, water, and multi-utility providers. Strong in time-of-use rate management and programme enrolment.
- **Product Page:** https://www.openintl.com/platform/
- **Note:** No public API documentation; contact Open International for integration guide.
- **Standards:** REST/JSON web services; SOAP for legacy CIS integrations
- **Authentication:** API key / enterprise SSO

### DataCapable Outage & Community Portal
- **Description:** AI-driven outage map, community portal, and key-accounts portal integrating data from SCADA, OMS, and weather services for customer-facing outage communication.
- **Product Overview:** https://datacapable.com/solutions/
- **Note:** Integration via proprietary API gateway; contact for developer documentation.
- **Standards:** REST/JSON
- **Authentication:** API key / OAuth 2.0

### OpenLEADR (Open-Source OpenADR)
- **Description:** Fully compliant open-source Python implementation of OpenADR 2.0b for both VTN (server) and VEN (client) roles. Enables demand-response programme integration in the customer portal without proprietary middleware.
- **Documentation:** https://openleadr.org/docs/
- **GitHub:** https://github.com/OpenLEADR/openleadr-python
- **Standards:** OpenADR 2.0b (OASIS Energy Interoperation 1.0)
- **Authentication:** TLS mutual authentication; XML signatures (optional)

### Green Button Alliance OpenESPI (Open-Source Reference Implementation)
- **Description:** Open-source Green Button Connect My Data server and client reference implementation. Enables utilities or portal developers to build a compliant CMD data custodian without proprietary SDKs.
- **Developer Guide:** https://green-button.github.io/developers/
- **GitHub:** https://github.com/GreenButtonAlliance/OpenESPI-GreenButton-API-Documentation
- **Standards:** NAESB REQ.21 ESPI v4.0, OAuth 2.0, Atom/XML
- **Authentication:** OAuth 2.0 (authorisation code flow)

---

## Notes

- **Green Button domain migration**: NREL is migrating its developer API domain from `developer.nrel.gov` to `developer.nlr.gov`; any portal integrating the OpenEI Utility Rates API should update to the new domain before April 2026 to avoid service interruption.
- **OpenADR 3.0 adoption**: OpenADR 3.0 is emerging but deployment at scale lags behind 2.0b. A portal targeting 2026–2027 should support 2.0b as the baseline and provide a migration path to 3.0.
- **IEC CIM licensing**: The CIM itself is maintained by the UCA CIM Users Group under Apache 2.0 licence, making it freely usable in open-source portal implementations.
- **NERC CIP boundary**: Customer portals are not directly subject to NERC CIP, but must not create attack vectors into CIP-protected operational systems. The OMS integration should be read-only and should present data through an intermediary API layer, not direct SCADA access.
- **Green Button certification**: The Green Button Alliance offers Green Button Certified CMD and DMD marks for implementations that pass its compliance test suite. Achieving certification improves utility buyer confidence and interoperability with third-party aggregators such as UtilityAPI.

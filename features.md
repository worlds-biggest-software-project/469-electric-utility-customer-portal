# Electric Utility Customer Portal — Feature & Functionality Survey

> Candidate #469 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Oracle Utilities Customer Cloud Service | Enterprise CIS + portal | Commercial SaaS | https://www.oracle.com/customer-hub/utilities/customer-cloud-service/ |
| KUBRA EZ-PAY / MyHQ | Customer engagement & payment portal | Commercial SaaS | https://www.kubra.com/industries/utilities |
| SilverBlaze Customer Portal | Utility self-service web portal | Commercial SaaS | https://www.silverblaze.com/ |
| Uplight Customer Portals | AI-powered energy engagement platform | Commercial SaaS | https://uplight.com/solutions/utility-energy-customer-portals/ |
| Bidgely UtilityAI | AI energy disaggregation & engagement | Commercial SaaS | https://www.bidgely.com/ |
| Open International Smartflex | Integrated CIS + portal + billing | Commercial SaaS | https://www.openintl.com/ |
| VertexOne VXcis | CIS with self-service customer portal | Commercial SaaS | https://www.vertexone.ai/ |
| DataCapable Community & Outage Portal | Outage mapping + community portal | Commercial SaaS | https://datacapable.com/ |
| Landis+Gyr Gridstream Connect | AMI data + home energy app | Commercial (hardware + software) | https://www.landisgyr.com/solution/gridstream-connect/ |
| SAP IS-U | Enterprise utility billing & CRM | Commercial (on-premise/cloud) | https://www.sap.com/industries/utilities.html |

---

## Feature Analysis by Solution

### Oracle Utilities Customer Cloud Service

**Core features**
- Account management: start, stop, transfer of service
- Bill presentment, payment processing, autopay enrolment
- Service order management with real-time CIS updates
- Smart-device enrolment management (thermostats, EV chargers, battery storage, solar inverters)
- Demand response event tracking and reservation management
- Integration with Oracle Field Service Cloud for appointment scheduling
- Integration with Network Management System for outage data
- Integration with Oracle Utilities Digital Assets Cloud Service (DER management)
- AI-based customer insights and next-best-action recommendations via agent console
- Programme management: enrolment processing across customers, programmes, and device locations

**Differentiating features**
- Context-aware agent console surfacing AI-driven insights for call-centre agents
- Full Customer Program Management module for controllable-device DR programmes
- Native integration with Oracle ERP Financial Cloud for GL/AP reconciliation
- Unified handling of scalar and interval-metered accounts

**UX patterns**
- Customer-facing self-service web portal for common tasks
- Real-time CIS updates on customer actions
- Configurable notification preferences (email, SMS)

**Integration points**
- Oracle Field Service Cloud, Network Management System, Oracle ERP Financial Cloud
- Payment processors (card and ACH)
- Oracle Utilities Digital Assets Cloud Service
- REST API for third-party integrations

**Known gaps**
- High total cost of ownership; primarily aimed at large investor-owned utilities
- Customisation requires Oracle Professional Services
- Limited out-of-box EV rate-plan self-enrolment compared to specialist platforms

**Licence / IP notes**
- Commercial SaaS; pricing by contract. Oracle proprietary IP.

---

### KUBRA EZ-PAY / MyHQ

**Core features**
- Online bill payment (card, ACH, text-to-pay, cash at retail kiosks)
- Paperless billing enrolment
- Payment history and scheduled payments
- Autopay setup and management
- Multi-factor authentication
- Outage map (Storm Center) with real-time outage visibility and restoration ETAs
- IncidentWatch: map-based tool for reporting broken streetlights and infrastructure issues
- Customisable email and SMS notification alerts
- Mobile-responsive portal accessible from any device

**Differentiating features**
- Cash payment at CVS and 7-Eleven via KUBRA EZ-Pay Cash
- Storm Center outage map with expandable cluster view
- IncidentWatch for non-power infrastructure reporting
- Transaction insights and customisable alert settings within a single hub (MyHQ)

**UX patterns**
- Single hub (MyHQ) for all customer engagement touchpoints
- Quick, secure, versatile payment flow
- Map-based outage reporting emphasising visual engagement

**Integration points**
- CIS systems via API (multiple vendor integrations)
- VertexOne partnership for billing/payment modernisation
- Retail cash payment network (CVS, 7-Eleven)

**Known gaps**
- Energy analytics and usage disaggregation not a core strength; relies on partner integrations
- Net metering / NEM tooling not featured prominently
- EV programme self-enrolment not highlighted

**Licence / IP notes**
- Commercial SaaS; KUBRA proprietary platform.

---

### SilverBlaze Customer Portal

**Core features**
- eBilling and online payment (24/7/365)
- Account activity and usage history display with graphs
- Real-time consumption data and bill comparison tools
- Comparison to neighbours (home energy benchmarking)
- Service start/stop/transfer
- Payment extension options and budget billing enrolment
- Outage reporting and integration with OMS
- Paperless billing and notification preferences
- Secure mobile app with near-real-time data
- AI-enhanced self-service (targeting 50% issue resolution without human intervention)

**Differentiating features**
- Regulatory-compliance focus: designed to help utilities meet data-access mandates
- Neighbour comparison benchmarking built in
- Self-service ROI calculator for utility business case
- Mobile app with expanded self-service options

**UX patterns**
- Login from any device; intuitive billing displays
- Progressive disclosure: customers see high-level bill first, drill into usage detail
- Proactive alerts for high usage or billing anomalies

**Integration points**
- CIS and billing system integration (multi-vendor)
- OMS integration for outage reporting
- Green Button data export

**Known gaps**
- AI analytics limited compared to dedicated platforms (Bidgely/Uplight)
- EV and DER management features not prominent
- Primarily targeted at small-to-mid-sized utilities

**Licence / IP notes**
- Commercial SaaS; SilverBlaze proprietary. No open-source components identified.

---

### Uplight Customer Portals

**Core features**
- AI/ML-powered personalised energy insights per household or business site
- Usage disaggregation (with or without AMI interval data)
- Bill forecasting and cost-change explanations
- Marketplace for energy efficiency products and rebates
- Home energy reports (digital and print)
- Customer alerts and notifications
- Programme enrolment (demand response, efficiency, electrification)
- Business-customer energy widgets embeddable in existing utility portals
- Tailored recommendations for clean energy transition (solar, EVs, electrification)

**Differentiating features**
- Purpose-built for utility-customer engagement via AI; formed by merging Tendril, EnergySavvy, Simple Energy, and Bidgee
- Modular architecture: portals, reports, marketplace, and alerts can be deployed independently
- Business customer widgets embeddable in third-party portals
- Behavioural science–informed recommendations to drive action

**UX patterns**
- Prioritised action recommendations surfaced prominently
- Disaggregated usage stats make abstract kWh consumption understandable
- Progressive engagement: bill insight leads to recommendation leads to programme enrolment

**Integration points**
- AMI/MDMS data feeds for interval data
- Utility CIS via API
- Third-party portal embedding (widgets)
- Marketplace integrations (product vendors, rebate processors)

**Known gaps**
- Billing and payment functionality not included; requires CIS partner integration
- Outage management not a feature
- Net metering tools not highlighted in public documentation

**Licence / IP notes**
- Commercial SaaS; Uplight proprietary.

---

### Bidgely UtilityAI

**Core features**
- AI/ML energy disaggregation at 15-minute interval resolution (12+ appliance categories)
- Per-appliance usage and cost breakdown for customers
- High Bill Analyzer for call-centre agents
- Next Best Interaction (NBI) engine combining 360° customer profiles
- Home Energy Reports and Digital Alerts
- GenAI conversational interface for customer self-service
- Co-Browse for call-centre agent-assisted navigation
- Appliance-level insights for major loads (HVAC, water heater, EV charger)
- Disaggregation-as-a-Service (DaaS) API for third-party developers

**Differentiating features**
- Patented disaggregation technology detecting 100+ customer attributes from meter data
- NBI engine combining appliance usage, weather, engagement history, and demographics
- DaaS API allowing third-party integration of disaggregation data
- GenAI conversational AI layer for natural-language customer interactions

**UX patterns**
- Appliance-level breakdown makes energy usage tangible and actionable
- High Bill Analyzer directly addresses top call-centre inquiry trigger
- Co-Browse reduces agent handle time and improves self-service adoption

**Integration points**
- MDMS/AMI data feed
- Utility CIS API
- DaaS REST API for third-party platforms
- Call-centre CRM integration

**Known gaps**
- Billing and payment not included
- Outage management not a feature
- Self-service account management (start/stop/transfer) not in scope

**Licence / IP notes**
- Commercial SaaS with patented disaggregation algorithms. DaaS API licensing available.

---

### Open International Smartflex

**Core features**
- CIS, billing, meter data management, service orders, and customer service in a single platform
- Self-service portal with bill history, itemised breakdown, and graphical usage comparisons
- Real-time usage and rate information
- Service start/stop, payment plans, autopay, paperless billing, budget billing
- Programme enrolment: rebates, pledges, donations, payment plans
- Flexible rate management: fixed, tiered, seasonal, and time-based (TOU) rates
- Online sales and digital onboarding
- Web services for digital ecosystem integration
- Multi-utility support (electricity, gas, water, bundled)

**Differentiating features**
- Truly unified CIS + portal in one product (not a portal layer over a third-party CIS)
- Emerging rate scheme support (TOU, dynamic pricing) without additional customisation
- Multi-utility / multi-commodity billing from a single system

**UX patterns**
- Insightful graphics for bill and usage comparison
- Self-service for most billing interactions including programme enrolment
- Real-time rate information displayed alongside usage

**Integration points**
- Web services API for external systems
- AMI/MDMS data integration
- Payment processors

**Known gaps**
- Energy disaggregation and AI analytics not a strength; typically integrates with specialist platforms
- EV programme management limited compared to Oracle CCS
- Customer-facing outage map not prominently featured

**Licence / IP notes**
- Commercial SaaS; Open International proprietary.

---

### VertexOne VXcis

**Core features**
- CIS with integrated self-service customer portal
- Account management: address updates, contact info, communication preferences
- Service start/stop/transfer
- Password reset and account security
- Real-time usage insights (current and historical)
- Outage and service request reporting
- PCI-compliant payments: text-to-pay, autopay, retail kiosks
- Programme self-enrolment: autopay, efficiency programmes, low-income assistance, rebates
- Customer support via text messaging and online chat

**Differentiating features**
- Text-to-pay and retail kiosk payment channels
- Low-income assistance programme enrolment in the self-service portal
- Integrated chat and text-message support channels
- Anomaly detection and consumption trend analysis via CIS-MDMS integration

**UX patterns**
- Easy-to-navigate knowledge base with FAQs
- Consistent multi-channel experience (web, text, kiosk)
- Proactive billing alerts

**Integration points**
- Smart metering systems (MDMS integration)
- Electronic Data Interchange (EDI) for data exchange
- Payment processors (PCI-compliant)
- Mobile Workforce Management (MWM) systems

**Known gaps**
- AI/ML energy analytics not a core offering
- EV and DER management features not highlighted
- Net metering tools limited

**Licence / IP notes**
- Commercial SaaS; VertexOne proprietary.

---

### DataCapable Community & Outage Portal

**Core features**
- Real-time outage map integrating SCADA/OMS/weather data
- Customer-facing outage case visualisation (location, cause, customer count, verified status)
- Estimated time of restoration (ETR) display
- Customer outage self-reporting for undetected outages
- Personalised outage alerts (email/SMS)
- Community Portal: two-way communication and interactive reporting tools
- Key accounts portal with dedicated outage updates
- Integration with SurvalentONE OMS for automatic data sharing
- Callout platform for utility workforce coordination (separate product)

**Differentiating features**
- AI-based threat detection for proactive outage identification
- Harmonised data streams from SCADA, OMS, and weather services via proprietary API
- Community portal for interactive, bi-directional customer communication
- Key-accounts focus for commercial/industrial customers

**UX patterns**
- Map-centric UI emphasising geographic outage context
- Cluster view at high zoom levels reduces map clutter
- Summary panel with regional outage statistics

**Integration points**
- SurvalentONE OMS integration
- SCADA system data feeds
- Weather service APIs
- Existing utility CIS and operational systems via API gateway

**Known gaps**
- Not a full-featured customer portal (no billing, usage analytics, or account management)
- Requires integration with CIS for customer account context
- Primarily sold as outage/incident management overlay

**Licence / IP notes**
- Commercial SaaS; DataCapable proprietary.

---

### Landis+Gyr Gridstream Connect

**Core features**
- AMI data collection and management from smart meters
- Meter Data Management System (MDMS): billing efficiency, anomaly detection, grid stability
- Consumer mobile app: real-time energy usage tracking, home analytics, customised alerts
- Appliance-level monitoring via Sense integration (Revelo platform)
- IoT platform for intelligent endpoint management
- DER integration and control
- Gridstream AIM for asset and infrastructure management (EMEA)

**Differentiating features**
- Vertically integrated: meters + communication network + MDMS + analytics
- Sense partnership for load disaggregation via home energy monitor hardware
- Open IoT platform enabling third-party application development

**UX patterns**
- Smartphone app as primary customer touchpoint for energy monitoring
- Real-time appliance tracking down to 15-minute intervals

**Integration points**
- Gridstream Connect open IoT platform (third-party app ecosystem)
- Utility CIS and billing systems
- DER/EV charger management
- Smart home device integrations

**Known gaps**
- Customer billing and payment not part of the platform
- Outage management not a core feature
- Requires separate CIS for account management

**Licence / IP notes**
- Commercial (hardware + software). Gridstream Connect IoT platform is open for third-party app integration.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Online bill payment (card, ACH, autopay) with PCI-compliant processing
- Billing history and electronic invoice delivery (paperless billing)
- Account management: contact details, communication preferences, start/stop/transfer of service
- Real-time or near-real-time usage display (daily and monthly charts)
- Outage self-reporting and status page with ETR
- Email and SMS notification alerts (billing, outage, high-usage)
- Multi-factor authentication and secure password reset
- Mobile-responsive design accessible from any device
- WCAG 2.1 Level AA accessibility compliance

### Differentiating Features
- AI/ML energy disaggregation providing appliance-level usage breakdowns
- Next Best Interaction (NBI) engine for hyper-personalised recommendations
- Neighbour/peer comparison benchmarking
- Programme enrolment portal (demand response, rebates, efficiency audits)
- Rate comparison tool projecting cost under alternative tariff plans
- EV integration: off-peak charging schedules, EV rate plan enrolment, charger management
- Net metering (NEM) account tools: generation vs. consumption, credit balance, true-up
- GenAI conversational interface for natural-language self-service
- Retail cash payment at third-party kiosks
- Key-accounts (C&I) portal with enhanced outage and usage data

### Underserved Areas / Opportunities
- Integrated NEM/solar settlement with transparent true-up calculations
- Time-of-use (TOU) guidance with actionable appliance-shift recommendations
- EV smart-charging scheduler integrating utility TOU pricing and grid signals
- AI-assisted high-bill investigation without requiring a call-centre agent
- Proactive financial-hardship detection and automated payment-plan offers
- Unified multi-commodity portal (electricity + gas + water in a single login)
- Accessible, plain-language bill explanations auto-generated from usage data
- AI-driven rate optimiser that continuously monitors and recommends tariff switches
- Open-source / open-standard portal that smaller co-ops and munis can deploy without enterprise licensing costs

### AI-Augmentation Candidates
- Energy disaggregation: AI already standard here but confined to expensive licensed platforms
- High-bill root-cause analysis: currently manual; AI can automate explanation generation
- Rate comparison: rule-based engines exist; AI can incorporate behavioural patterns and projections
- Outage ETA prediction: ML on historical outage + weather data to improve restoration estimates
- Demand response optimisation: personalised DR event scheduling based on appliance usage patterns
- Chatbot / virtual assistant: replacing FAQ navigation with natural-language query resolution
- Fraud and anomaly detection on login and payment flows
- Personalised energy-efficiency recommendation ranking based on home characteristics

---

## Legal & IP Summary

The major competing products (Oracle Utilities, KUBRA, SilverBlaze, Uplight, Bidgely, Smartflex, VertexOne) are all commercial SaaS platforms with proprietary codebases. Bidgely holds patents on energy disaggregation algorithms; building a competing open-source disaggregation engine would need to avoid those specific patented approaches or use alternative methods (deep learning models trained on openly licensed datasets). The Green Button / NAESB ESPI standard is freely implementable; the IEC CIM standard is open under Apache 2.0. OpenADR specifications are freely available from the OpenADR Alliance. No copyright or licence concerns were identified for using published API specifications or industry standards as the basis for a new portal. Open-source libraries for billing display, mapping (Leaflet/MapLibre), and authentication (Keycloak) are available under permissive licences.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Authenticated customer account portal (MFA, WCAG 2.1 AA)
- Bill presentment, payment (card/ACH), and autopay enrolment (PCI DSS compliant)
- Interval usage dashboard (daily/monthly charts from AMI/MDMS data feed)
- Outage self-reporting and real-time outage map (OMS integration)
- Email/SMS notification preferences and alert delivery
- Green Button Connect My Data (CMD) API for customer-authorised data sharing

**Should-have (v1.1)**
- Rate comparison tool using OpenEI Utility Rate Database
- Net metering (NEM) account view: generation, consumption, credit balance, true-up
- AI-generated plain-language bill explanation and high-bill root-cause summary
- Programme enrolment: demand response, rebates, efficiency audits
- EV rate plan self-enrolment and off-peak charging scheduler (OpenADR integration)
- Neighbour comparison benchmarking

**Nice-to-have (backlog)**
- AI energy disaggregation (appliance-level breakdown) via open DaaS API or on-device ML
- Conversational AI (GenAI chatbot) for natural-language self-service
- Key-accounts / C&I portal with interval demand data and power factor display
- Multi-commodity support (gas and water alongside electricity)
- Financial-hardship proactive detection and automated payment-plan offer
- Retail cash payment channel integration

# Electric Utility Customer Portal

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-enhanced self-service portal that gives electric utility customers transparent access to usage data, billing, outage status, net metering, and rate analysis -- without enterprise licensing costs.

Electric Utility Customer Portal is a web-based platform that connects residential and commercial electricity customers with the data and tools they need to understand their energy consumption, pay bills, report outages, manage net-metering accounts, and evaluate alternative rate plans. It targets electric utilities of all sizes -- from large investor-owned utilities to smaller cooperatives and municipal providers -- that need a modern, standards-compliant customer engagement layer on top of existing CIS, MDMS, and OMS back-office systems.

---

## Why Electric Utility Customer Portal?

- **Enterprise lock-in dominates the market.** Oracle Utilities, SAP IS-U, and other incumbents charge enterprise-scale licensing fees, putting modern customer portals out of reach for smaller co-ops and municipal utilities.
- **Fragmented solutions leave gaps.** No single incumbent covers billing, outage management, usage analytics, net metering, and EV integration in one product. KUBRA lacks energy analytics; Uplight and Bidgely lack billing and outage features; DataCapable only handles outage mapping.
- **AI analytics are siloed behind expensive platforms.** Energy disaggregation (Bidgely), personalised recommendations (Uplight), and high-bill analysis are available only as proprietary SaaS add-ons with patented algorithms and per-meter pricing.
- **Net metering and TOU guidance are underserved.** Integrated NEM settlement with transparent true-up calculations and actionable time-of-use appliance-shift recommendations are weak or absent across all surveyed solutions.
- **Open standards exist but are underutilised.** Green Button Connect My Data, OpenADR, IEC CIM (Apache 2.0), and the OpenEI Utility Rate Database provide a solid foundation for an open-source alternative, yet no open portal leverages them comprehensively.

---

## Key Features

### Usage Dashboard and Analytics

- Hourly, daily, and monthly electricity consumption displayed against billing-period targets
- Comparisons to prior periods and similar homes (neighbour benchmarking)
- Interval usage charts from AMI/MDMS smart-meter data feeds (15-minute resolution)
- AI-generated plain-language bill explanations and high-bill root-cause summaries

### Billing and Payments

- Electronic bill presentment with itemised breakdown
- Online payment by card or ACH, autopay enrolment, and payment history
- PCI DSS-compliant payment processing
- Budget billing and payment-plan enrolment

### Outage Reporting and Status

- Customer-submitted outage reports linked to service address
- Real-time outage map with affected areas, crew status, and estimated restoration times
- Automated SMS and email alerts for outage and restoration events
- OMS integration for live data without exposing internal SCADA systems

### Net Metering and Rate Analysis

- Solar generation vs. consumption display with NEM credit balance
- Monthly settlement statement and annual true-up calculation
- Rate comparison tool using OpenEI Utility Rate Database against actual usage profile
- Time-of-use guidance with appliance-shift recommendations and peak-price alerts

### Programme Enrolment and EV Integration

- Demand response programme enrolment and event tracking
- Rebate applications and energy efficiency audit scheduling
- EV rate plan self-enrolment and off-peak charging scheduler (OpenADR integration)
- Home charger management integration

### Account Management and Security

- Contact details, paperless billing, and notification preferences
- Service start, stop, and transfer of service
- Multi-factor authentication with anomaly detection on login
- WCAG 2.1 Level AA accessibility compliance
- Green Button Connect My Data (CMD) API for customer-authorised third-party data sharing

---

## AI-Native Advantage

An open-source portal can embed AI capabilities that incumbents sell as expensive add-ons. AI-generated plain-language bill explanations automate the most common call-centre inquiry without agent involvement. ML-based rate optimisation continuously monitors usage patterns and recommends tariff switches, going beyond the static rule-based engines in current products. Outage ETA prediction trained on historical outage and weather data improves restoration estimates. Energy disaggregation using open deep-learning models (avoiding patented approaches) can deliver appliance-level breakdowns without per-meter SaaS fees.

---

## Tech Stack and Deployment

The portal is designed for self-hosted, cloud, or hybrid deployment to accommodate utility IT policies. It builds on open standards: Green Button Connect My Data (NAESB ESPI) for customer data sharing, OpenADR for demand response signalling, IEC CIM (Apache 2.0 licensed) for utility data modelling, and the OpenEI Utility Rate Database for tariff analysis. The architecture assumes integration with existing CIS (billing), MDMS (meter data), and OMS (outage) systems via REST APIs. Key open-source libraries include Leaflet or MapLibre for outage mapping, Keycloak for identity and MFA, and standard PCI-compliant payment processor integrations. The portal must scale independently from back-office systems to handle tenfold traffic spikes during major storm events.

---

## Market Context

The electric utility customer engagement software market is driven by smart-meter proliferation (over 75% of US homes have smart meters in 2026), regulatory mandates for customer data access, and the growth of distributed energy resources. Incumbent platforms like Oracle Utilities Customer Cloud Service and SAP IS-U carry high total cost of ownership and typically require vendor professional services for customisation. Smaller utilities, cooperatives, and municipal providers represent an underserved segment that cannot justify enterprise licensing but still face the same customer expectations for digital self-service.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.

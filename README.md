<div align="center">

<img src="https://github.com/user-attachments/assets/2adf3ea6-8f54-47e9-b14e-25790be39173" width="120" alt="WhisprBill Logo" />



# WhisprBill

### AI-powered GST invoicing SaaS for Indian businesses

<p>
  <a href="https://whisprbill.com">Landing</a> •
  <a href="https://app.whisprbill.com">Live App</a>
</p>

</div>

---

## What is this?

WhisprBill is a production-grade AI invoicing SaaS that converts voice and text input into GST-compliant invoices.

It is built for real businesses to reduce manual effort and compliance friction.

This is a showcase repository. The production codebase is private.

---

## Key Highlights

- LLM-driven workflows that convert unstructured input into structured invoice data  
- Full-stack system built using React, Node.js, and MongoDB  
- Subscription handling and payments via Razorpay  
- Dynamic PDF generation using Puppeteer and Handlebars  
- Multi-tenant architecture with company-level data isolation  
- Deployed on AWS using Docker and CI/CD pipelines  

---

## Architecture

See full diagrams in [`/architecture`](./architecture/)

- Client: React SPA and Next.js landing page  
- API: Express.js with service-layer architecture  
- Database: MongoDB  
- AI: Groq, used for parsing and intent extraction  

---

## Tech Stack

- Frontend: React, Next.js, Tailwind CSS  
- Backend: Node.js, Express.js  
- Database: MongoDB  
- Infrastructure: AWS EC2, Docker, GitHub Actions  
- AI: Groq API  
- Payments: Razorpay  

---

## Engineering Decisions

Detailed writeups:

- [`AI as Parser`](./engineering-decisions/ai-as-parser-not-logic.md)  
- [`PDF Storage Strategy`](./engineering-decisions/pdf-storage.md)  
- [`Webhook Idempotency`](./engineering-decisions/payment-webhook-handling.md)  
- [`MongoDB Schema Design`](./engineering-decisions/mongodb-schema-design.md)  



---

## Screenshots

## Screenshots

### Invoice Creation

![Invoice Creation](https://github.com/user-attachments/assets/b0d30cc8-35ab-4304-ae77-3a575289ee5f)

---

### Downlaod and Share Instantly

![Generate and Share directly](https://github.com/user-attachments/assets/9713bef9-d95b-4b44-beb3-4631f1008155)

---

### Customer Management

![Customer Management](https://github.com/user-attachments/assets/598038d1-b1b4-493f-a51a-5dd555e81033)

---

### Inventory Management

![Inventory Management](https://github.com/user-attachments/assets/ec4d4447-676d-47b7-aad4-786d020d163e)




---

## Links

- https://whisprbill.com  
- https://app.whisprbill.com  

---

<div align="center">
<sub>Built to demonstrate system design, backend architecture, and product thinking.</sub>
</div>

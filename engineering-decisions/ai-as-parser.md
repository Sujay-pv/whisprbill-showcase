# AI as Parser, Not Business Logic

## Problem

LLMs are non-deterministic by nature. In a financial system, this creates risk.  
If the model is responsible for calculations, totals, or validation, even small inconsistencies can lead to incorrect invoices.

The goal was to use AI for flexibility without compromising correctness.

---

## Decision

The AI layer is used only for intent parsing and entity extraction.

User input (text or voice) is converted into structured JSON.  
All financial logic, including GST calculation, totals, validation, and persistence, is handled by deterministic backend code.

---

## Flow
      User Input (text / voice)
      │
      ▼
      Prompt Builder
      │
      ▼
      LLM (Groq)
      │
      ▼
      Structured JSON Output
      │
      ▼
      Backend Services
      (Taxes, totals, validation, DB write)

 
---

## Why this approach

- Keeps financial calculations deterministic and reliable  
- Prevents incorrect invoices due to model variance  
- Allows safe use of LLMs in production workflows  
- Makes the system easier to reason about and debug  

---

## Tradeoffs

- Requires an additional orchestration layer between input and business logic  
- Slight increase in implementation complexity  
- Extra validation needed to handle malformed model outputs  

---

## Notes

The AI layer can trigger CRUD operations through structured outputs, but it never executes business logic directly.  
This separation ensures that all critical operations remain predictable and testable.     

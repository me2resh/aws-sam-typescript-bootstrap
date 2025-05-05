# Chapter 4: Appointment Repository Interface

In the [previous chapter](03_appointment_service_.md), we saw how the [Appointment Service](03_appointment_service_.md) acts as a coordinator, taking requests from the Lambda Handler and being responsible for getting appointment data. We mentioned that the service delegates the actual data fetching task.

But how does the service *know* how to ask for data? What if we want to get data from a simple list today, but from a powerful online database tomorrow? Should the [Appointment Service](03_appointment_service_.md) need to be rewritten every time we change where the data comes from? That sounds messy!

## What's the Problem? Standardizing the Request

Imagine you have different appliances: a lamp, a toaster, a phone charger. They all need electricity, but they plug into the same standard wall outlet. The outlet defines *how* they get power (the shape of the plug, the voltage), but it doesn't care *what* the appliance does with that power.

Similarly, our [Appointment Service](03_appointment_service_.md) needs appointment data. It needs a standard way to *ask* for this data, regardless of whether the data is stored in:

*   A simple, temporary list in memory (like we'll use for testing).
*   A file on the server.
*   A professional database service in the cloud (like AWS DynamoDB).

The service shouldn't have to worry about the specific details of *how* the data is retrieved from each source. It just needs a consistent "plug" to connect to.

## The Solution: A Contract (Interface)

In programming, especially with TypeScript, we can create something called an **Interface**. Think of an interface as a **contract** or a **blueprint for behavior**.

It defines *what* methods must exist, what inputs they take, and what outputs they return, but it *doesn't* provide the actual code for *how* those methods work.

It's like the electrical outlet standard:

*   **The standard (Interface) says:** "You must accept a plug with two flat pins and one round pin. You must provide ~120 Volts."
*   **It does NOT say:** "The electricity must come from a coal power plant" or "The wires in the wall must be copper".

This contract allows the [Appointment Service](03_appointment_service_.md) to work with *any* data source, as long as that data source agrees to follow the rules defined in the contract.

## Our Contract: `AppointmentRepository`

Let's look at the contract we've defined for getting appointment data. You can find it in `src/infrastructure/appointment-repository.ts`.

```typescript
// File: src/infrastructure/appointment-repository.ts
import { Appointment } from '@/domain/appointment'; // Need the blueprint for data

// This is the contract!
export interface AppointmentRepository {
    // Any class implementing this MUST have this method:
    getAppointmentsByPatientId(patientId: string): Promise<Appointment[]>;
}
```

Let's break down this simple contract:

1.  **`import { Appointment } from '@/domain/appointment';`**: We need to refer to the [Appointment](02_domain_model__appointment__patient__.md) blueprint because our contract involves fetching appointments.
2.  **`export interface AppointmentRepository { ... }`**: This declares a contract named `AppointmentRepository`. `export` means other files can use it. `interface` signals it's a contract, not a class with implementations.
3.  **`getAppointmentsByPatientId(patientId: string): Promise<Appointment[]>;`**: This is the core rule of the contract. It states:
    *   Anyone following this contract *must* provide a function (method) named `getAppointmentsByPatientId`.
    *   This function must accept one piece of input: `patientId`, which must be text (`string`).
    *   This function must return a `Promise`. A `Promise` means the operation might take some time (like fetching from a database). Eventually, it will resolve to give back an array (`[]`) of `Appointment` objects (`Appointment[]`).

That's it! The contract only defines *what* is needed: a way to get appointments for a patient ID. It says nothing about *how* to actually find them.

## How the Service Uses the Contract

Remember how we created the [Appointment Service](03_appointment_service_.md)?

```typescript
// File: src/application/appointment-service.ts (Constructor part)
import { AppointmentRepository } from '@/infrastructure/appointment-repository';

export class AppointmentService {
    // It accepts ANY object that fulfills the AppointmentRepository contract
    constructor(private appointmentRepository: AppointmentRepository) {}
    // ... other methods ...
}
```

The key is `private appointmentRepository: AppointmentRepository`. The service doesn't ask for a *specific kind* of repository (like `MockAppointmentRepository` or `DatabaseRepository`). It asks for *any* object that fulfills the `AppointmentRepository` **interface (contract)**.

This makes the service incredibly flexible. We can plug in different implementations:

```mermaid
graph TD
    AS[Appointment Service] -- Uses --> ARI(Appointment Repository Interface);
    subgraph "Implementations (Plug into the Interface)"
      MOCK[Mock Appointment Repository]
      REAL[Real Database Repository (Future)]
    end
    ARI -- Implemented by --> MOCK;
    ARI -- Implemented by --> REAL;

    style ARI fill:#f9f,stroke:#333,stroke-width:2px
    style MOCK fill:#ccf,stroke:#333,stroke-width:1px
    style REAL fill:#ccf,stroke:#333,stroke-width:1px,stroke-dasharray: 5 5
```

*   When we run our tests or do initial development, we can give the service a [Mock Appointment Repository](05_mock_appointment_repository_.md) that just returns predefined sample data (like plugging in a simple test light).
*   When we deploy our application for real users, we could create a `DynamoDBAppointmentRepository` that fetches data from a real AWS database and give *that* to the service (like plugging in the actual toaster).

The [Appointment Service](03_appointment_service_.md) code doesn't change at all! It just calls `this.appointmentRepository.getAppointmentsByPatientId(...)`, trusting that whatever object it was given follows the contract.

## Benefits of Using an Interface

This approach gives us significant advantages:

1.  **Flexibility (Swappability):** We can easily switch out the data source (mock, database, file) without modifying the core application logic ([Appointment Service](03_appointment_service_.md), [Lambda Handler (Patient Appointments)](01_lambda_handler__patient_appointments__.md)). We just create a new class that implements the `AppointmentRepository` interface and "plug it in".
2.  **Decoupling:** The [Appointment Service](03_appointment_service_.md) is "decoupled" from the data storage details. It depends only on the *contract* (interface), not on a specific *implementation*.
3.  **Testability:** It's super easy to test the [Appointment Service](03_appointment_service_.md). We can create a simple mock repository specifically for testing that returns predictable data, allowing us to check if the service logic works correctly without needing a real database.

## Conclusion

We've learned that the **Appointment Repository Interface** (`AppointmentRepository`) is a vital **contract**. It defines *what* operations related to appointment data storage must be available (like `getAppointmentsByPatientId`) but not *how* they are performed.

This allows our [Appointment Service](03_appointment_service_.md) to remain flexible and independent of the specific data storage mechanism. It relies on the contract, enabling us to "plug in" different implementations like mock data stores or real databases.

Now that we have the contract, we need an actual implementation that follows its rules. Let's look at our first implementation: a simple, in-memory data source perfect for getting started and testing.

Next up: [Mock Appointment Repository](05_mock_appointment_repository_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)
---
title: "TrafficG: Defining the Reservation Flow and Project Scope"
description: "Early planning notes for an online concert reservation prototype, from user journeys to APIs and data modeling."
publishDate: 2025-05-04
updatedDate: 2026-10-19
tags: [trafficg, system-design, project-notes]
draft: false
---

> Translated from [the original Korean planning note](https://tyge-ottesen-brahe.tistory.com/29). This is an **initial plan**, not a list of completed functionality.

## 1. Define the features

The first step was to decide what a concert reservation platform should let users and administrators do.

### User-facing features

- Register an account and log in.
- Browse performances and open a performance detail page.
- View seat availability and select seats.
- Submit a reservation request, including appropriate handling for seats that another customer has already selected or reserved.
- Check the result of the reservation.

### Administrative features

- Register performances, including names, dates, and seating capacity.
- Configure seat layouts by rows and columns.
- View basic reservation status by performance.
- Set ticket sale opening times.

### Possible future extensions

- User-side reservation cancellation and a personal account page.
- Scheduled ticket-sale opening.
- Reopening seats after cancellations or payments that remain incomplete for a defined period (the original plan suggested 15 minutes).

These were feature ideas at the planning stage. Their inclusion here does not establish that they were all implemented.

## 2. Design the user journey

The main flow I wanted to model was:

**Browse a performance → join the waiting room → select available seats → request a reservation → confirm the outcome.**

This journey raises questions about capacity, concurrent selection, reservation expiration, and failure handling. A seat that appears available in the browser may no longer be available when the user submits a reservation. The authoritative decision must therefore occur on the server.

## 3. Design the data model

The initial ERD would cover performances, seats, users, reservations, and waiting-room entries. I wanted to identify one-to-many and many-to-many relationships and consider how the entities would map to JPA. The aim was to express a reservation workflow as a consistent data model before writing the business logic.

## 4. Specify APIs

The next task was to document request and response contracts for registration, login, reservation, and queue entry. This would help coordinate frontend and backend development and expose ambiguous behavior before implementation.

## 5. Initialize the backend project

The setup plan was to create a Spring Boot project using Gradle or Maven, initialize Git, connect the GitHub repository, and establish the initial domain and configuration packages. That would provide a starting point for implementing authentication and reservation logic.

## Reflection

What interested me about this project was not just displaying a ticketing interface. It was the question of how the system should behave when many people request limited seats at nearly the same time. The planning phase helped me identify data consistency and failure cases as design concerns, even before implementation.
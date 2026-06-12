# 🎟️ ShowTime - Ticket Booking System Architecture

A system design project to build a scalable ticket booking platform (BookMyShow-like) capable of handling **5 lakh concurrent users** with **zero double-booking guarantee** under a constrained **$2,000/month AWS budget**.

---

## 🚧 Problem Statement

We are designing the backend infrastructure for a high-demand ticket sale (e.g., Coldplay concert) with the following constraints:

* ⚡ **5 lakh users at peak (12:00 PM launch)**
* 🚫 **Zero tolerance for double-booking**
* 💰 **Strict budget: $2,000/month on AWS**
* 🧱 **No existing codebase — built from scratch**

---

## 🧠 Key Constraints & Assumptions

### 1. High Concurrency

* 5 lakh users may hit the system simultaneously
* Peak traffic can result in extremely high RPS
* Bottlenecks likely at:

  * Database connections
  * Locking mechanisms

---

### 2. Zero Double-Booking

* A double booking = same seat booked twice
* Must guarantee **strong consistency**
* Requires:

  * Row-level locking OR distributed locks
  * Atomic operations

---

### 3. Budget Constraint ($2,000/month)

Approx infra we can afford:

* EC2 instances (application servers)
* RDS PostgreSQL (primary DB)
* Redis (cache + locks)
* SQS (async processing)
* ALB (load balancing)

👉 Forces us to:

* Avoid over-engineering
* Prefer simpler, reliable solutions

---

## 🏗️ What This Project Contains

This repository includes **4 core design documents**:

### 📄 1. SCHEMA.md

* PostgreSQL schema design
* Tables: events, venues, seats, users, bookings, booking_seats
* Constraints, indexes, relationships

---

### 🔒 2. CONCURRENCY.md

* Strategy to prevent double-booking
* Tradeoff analysis:

  * PostgreSQL locking vs Redis locks
* Final chosen approach with justification

---

### ⚡ 3. CACHE.md

* Redis caching strategy
* What to cache:

  * Event details
  * Seat availability counts
  * Seat layouts
* TTL values + invalidation logic

---

### 📬 4. QUEUE.md

* Async order processing using SQS
* Payment worker flow:

  * Success path
  * Failure handling
* Retry + DLQ strategy

---

## 🧩 System Design Highlights

* ✅ Strong consistency for seat booking
* ⚡ Async processing for scalability
* 🧠 Smart caching for performance
* 🔁 Fault-tolerant queue-based architecture

---

## 🚀 Design Philosophy

This system is designed with:

* **Simplicity over complexity**
* **Correctness over speed (for bookings)**
* **Scalability within budget constraints**

---

## 🔮 Future Improvements

* Distributed locking via Redis (for higher scale)
* Read replicas for scaling reads
* Rate limiting & abuse prevention
* CDN for static content
* Auto-scaling infrastructure

---

## 📌 Summary

This project demonstrates how to design a **high-scale, fault-tolerant ticket booking system** under real-world constraints.

The focus is on:

* Preventing race conditions
* Handling peak traffic efficiently
* Making smart tradeoffs under budget limits

---

## 👨‍💻 Author

Nikunj Kohli

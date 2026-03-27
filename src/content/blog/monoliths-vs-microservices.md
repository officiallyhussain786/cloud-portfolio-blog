---
title: "Why I Chose a Monolith First (And When I'll Split It)"
description: "Learning Go + building a carpool app taught me something: microservices aren't a starting point. They're a destination. Here's why that matters for indie projects."
pubDate: 2025-03-27
category: Systems
---
 
## The Context
 
I'm building a carpool app in Go. Simple premise: users request rides, match with drivers, process payments.
 
When I started, I had a choice: build it as one monolith, or start with microservices architecture from day one.
 
Everyone online says: "Start with services. You'll thank yourself later."
 
I did the opposite. Here's why—and what I actually learned about this choice.
 
## What I Was Tempted To Do
 
The architecture porn was appealing:
 
- Auth service (stateless, scales independently)
- Ride service (real-time matching, Redis)
- Payment service (strict consistency, PostgreSQL)
- Notification service (async, decoupled)
 
It *looked* professional. It *felt* like I was doing it right.
 
But then I got honest with myself: **I've never actually run a microservices system in production.** I read about them. I studied them. I know the theory. But I haven't lived through the debugging, the operational pain, the coordination overhead.
 
So why would I start there?
 
## What I Learned From People Who Actually Did This
 
### Netflix's Story (What I Understood From It)
 
I've read about Netflix's journey. In 2008, they hit a database corruption that nearly killed them. That forced them to ask: "How do we build systems that don't fail catastrophically?"
 
The answer they landed on was microservices—but not because it's trendy. Because they had a *specific problem*: one database going down shouldn't take everything down.
 
The thing I noticed: They also hired hundreds of engineers to manage it. They built entire teams around observability, chaos engineering, monitoring. It wasn't magic. It was deliberate investment in the complexity.
 
The lesson I took: **Microservices are a response to specific pain. Not a destination.**
 
### Twitter's Consolidation (And What That Suggests)
 
I've watched people discuss Twitter's move from thousands of services back to consolidated systems. The argument: with smaller teams, the overhead of orchestrating between services outweighed the benefits.
 
This stuck with me because it suggests something important: **The best architecture isn't fixed. It moves based on constraints.**
 
If constraints change (team size, product focus, market pressure), the architecture can change too.
 
## Where I Landed: Start Simple
 
I decided to start my carpool app as one Go service. Not because I'm anti-microservices. Because:
 
**1. I can ship faster.**
 
One codebase. One deployment pipeline. One database. If I need to iterate on core logic, I change one place.
 
**2. I'll actually understand the coupling.**
 
If I split too early, I won't know *why* services fail or how they interact. With a monolith, the dependencies are explicit and visible. I can learn.
 
**3. The real constraints don't exist yet.**
 
Right now, I'm one person. I'm not waiting on other teams. I don't have 10,000 requests per second. I have zero users. The problems that microservices solve don't exist yet.
 
**4. I can refactor later if I need to.**
 
If I hit genuine pain—"this module is hard to change," "this piece scales differently," "this is genuinely separate logic"—I can split it out. But I'm not paying the operational cost for pain that might never come.
 
## The Costs I've Read About (And Why They Seemed Real)
 
I spent a lot of time studying how microservices fail. Here's what stood out:
 
### Debugging Across Services
 
When I read how people debug issues in distributed systems, it sounded... complex. A request touches 3 services. Something fails. Now you need:
 
- Logs from each service
- Distributed traces
- Network monitoring
- Correlation IDs across systems
 
With one service? One codebase. One set of logs. Grep. Done.
 
I don't have distributed systems experience yet, but the surface area for bugs seems bigger.
 
### Latency Adds Up
 
I've done the math on this: If every service call takes 50ms, and a user request touches 5 services, that's 250ms of latency just for inter-service communication.
 
In a monolith, same logic, one database roundtrip. Much faster.
 
When I profiled my carpool ride-matching logic locally, I realized: the fastest version is the one that doesn't jump between services. Simple truth.
 
### Coordination Overhead
 
This one hit me as I was planning: if I had split auth into its own service early, every time I needed to change the user-to-ride relationship, I'd need to:
 
- Update the schema in auth service
- Update the schema in ride service
- Coordinate deployments
- Test the integration
 
With one service, I change the schema once. Run tests. Deploy.
 
## So When Might I Split? (Honest Answer: I Don't Know Yet)
 
Here's what I've noticed from reading and studying:
 
**If my app grows and I have a team:**
- Someone focuses on real-time matching (different tech, different constraints)
- Someone focuses on payments (strict consistency, different SLA)
- Someone focuses on notifications
 
Then maybe services make sense. Maybe.
 
**If my ride-matching becomes genuinely slow:**
- I could pull it into its own service with Redis
- Scale it independently
- Keep the rest of the monolith simple
 
**If payments needs PCI compliance and audit logs:**
- Maybe that becomes its own service with stricter controls
- Segregated database
- Different deployment pipeline
 
But today? None of these are real problems. So I'm not pre-solving them.
 
## What I'm Actually Building
 
Right now: one Go service. One PostgreSQL database. Clear module boundaries in code:
 
```
/auth       (JWT, user login)
/rides      (ride logic)
/matching   (algorithm, matching)
/payments   (Stripe integration)
/handlers   (HTTP routes)
```
 
They're separated in the codebase, but deployed together. If one module gets big enough or complex enough that it needs its own service, I'll know by then.
 
And I'll have learned enough to do it right, instead of guessing.
 
## The Actual Lesson (For Me, And Maybe For You)
 
Here's the uncomfortable honest part: I don't have production microservices experience. I'm learning. I'm building. I'm reading about what other people have learned, and I'm trying to make smart choices early.
 
The choice I made: **Start simple. Learn the real constraints. Refactor when I understand them.**
 
Not because I'm anti-microservices. Because I'm honest about where I am in my learning.
 
## If You're Building Something Too
 
**Where I landed:**
 
- One service initially
- Clear code boundaries (modules, not services)
- Ship fast, learn what actually breaks
- Split services if/when I hit real pain
- Be willing to consolidate if I was wrong
 
**Why this matters:**
 
The coolest architecture isn't the one that *looks* impressive. It's the one that actually lets you ship, gather feedback, and iterate.
 
If you're building something—whether it's a side project, an MVP, or just learning—start where I started:
 
1. Simple monolith
2. Well-organized code
3. Real problems before real complexity
 
Then, if you hit genuine constraints (team size, scaling mismatch, different tech needs), you'll understand them deeply enough to solve them right.
 
## What I'm Learning
 
- How to structure Go code clearly (modules matter)
- How to make good deployment decisions
- When to optimize and when to ship
- How to read about architecture without getting caught in analysis paralysis
 
And maybe the most important: **Admitting when I don't know yet. And building in a way that lets me learn.**
 
That's actually the competitive advantage—not the architecture. The ability to make good decisions as constraints change.
 
Ship simple. Learn from real feedback. Iterate when you understand the problem.
 
That's the path forward.
 

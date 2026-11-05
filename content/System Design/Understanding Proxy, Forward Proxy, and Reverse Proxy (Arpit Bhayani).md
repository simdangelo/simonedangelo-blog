---
date: 2026-01-11
---

*This article is entirely based on a [YouTube video by Arpit Bhayani](https://www.youtube.com/watch?v=HrG0MHkSsCA&t=314s)*.

---

A **proxy** is a **machine or set of machines positioned between two systems**. These systems might be a user and a backend service, two backend systems, or any other pairing. The proxy serves as an intermediary, typically introduced to **abstract complexity**, manage untrusted environments, or enforce policies.

# Forward Proxy
A **forward proxy** is the type most users encounter in colleges and workplaces. It **abstracts and protects clients** by acting as a middleman. When a client makes a request, the call passes through the proxy to the internet or another service. The forward proxy handles forwarding the request, receiving the response, and shielding the client's identity from external systems.

**The external world sees only the proxy's IP address, not the client's.**

This provides several advantages, beginning with **security**. By masking client identities, external services cannot directly identify individual users behind the proxy. A concrete example illustrates this protection mechanism. During a hackathon in 2014, a team was scraping LinkedIn data to build a search engine. LinkedIn employed rate limiting, and when the scraping script made too many rapid requests, LinkedIn blocked the originating IP address. However, because the entire college network operated behind a forward proxy, LinkedIn saw only the proxy's IP address. The block therefore affected the entire institution—no one at the college could access LinkedIn, even though only one user had triggered the rate limit. This scenario is common in educational institutions and corporate environments, where forward proxies protect client identities and consolidate network access.

The second major advantage of forward proxies concerns **policy enforcement**. Organizations frequently need to restrict access to certain websites. Because all requests funnel through a single machine, administrators can apply comprehensive access policies at the proxy level. For example, torrent sites were blocked at the college level through proxy configuration. Similarly, India's national firewall blocks TikTok by configuring ISP proxies to reject any requests destined for that domain. In corporate environments like Walmart, certain websites may be blocked by default. When an employee needs access to a blocked site, they must submit a request for IT administrators to review the site and potentially whitelist it.

The third benefit is **caching** frequently accessed content on the proxy itself. Between 2008 and 2012, Java documentation was cached on the college proxy. Students coding in Java needed frequent access to these docs, so caching them locally meant they loaded quickly and remained accessible even when internet connectivity was unavailable. This caching strategy reduces bandwidth consumption and improves response times for commonly requested resources.

# Reverse Proxy
A **reverse proxy** operates in the opposite direction: it **abstracts the complexities of downstream systems rather than protecting clients**. Users connect to the reverse proxy, which then routes requests to the appropriate backend servers. The most familiar example of a reverse proxy is a **load balancer**.

When a user connects to a load balancer, the load balancer selects one of several backend servers based on its configured algorithm. This arrangement abstracts the complexity of the backend infrastructure—users need not know how many servers exist or how they're organized. **Load balancing is one of the most common reasons for deploying a reverse proxy.**

The second major use case is **routing**, exemplified by **API gateways**. An API gateway can route requests based on path prefixes: requests starting with `/auth` might go to the authentication service, while those starting with `/payments` route to the payment service. The routing logic resides in the reverse proxy, which directs traffic to the appropriate downstream service.

**Caching** represents the third advantage. Since all traffic passes through the reverse proxy, it can cache static responses and serve them directly without contacting the origin server. If a particular blog post becomes very popular, the content can be cached at the load balancer level. Subsequent requests for that post receive immediate responses from the cache, conserving API server bandwidth, CPU, and memory.

The fourth benefit is **infrastructure abstraction**. **The reverse proxy becomes a single point of entry, hiding whether five, ten, or fifteen servers operate behind it.** Clients always connect to the load balancer's domain name, and the load balancer handles server selection. This abstraction enables elastic infrastructure: servers can be added or removed without affecting users, who remain unaware of these changes. The reverse proxy masks downstream complexity, whereas the forward proxy masks client complexity.

Common reverse proxy implementations include load balancers like Nginx and HAProxy, API gateways like Kong, and database proxies. Database proxies deserve particular attention.

# Database Proxies
**ProxySQL** exemplifies a **database proxy**. It accepts SQL queries from clients and abstracts the underlying database topology. Behind the scenes, the database may be sharded or partitioned across multiple servers, but clients interact with a single endpoint. The routing logic resides in the database proxy. While ProxySQL is one concrete implementation, most database systems provide their own proxy solutions.

Database proxies offer three primary advantages. First, they cache common SQL queries. When a frequently executed query arrives, the proxy can serve results from its cache rather than forwarding the request to the database. Only the initial request reaches the database; subsequent identical queries receive cached responses. This significantly reduces database load.

Second, database proxies implement connection pooling. They accept numerous connections from clients but maintain a limited, optimized pool of connections to the actual database servers. This approach maximizes database connection utilization while accommodating many simultaneous clients.

Third, they abstract database topology. Clients don't need to know how data is distributed, which servers own which data, or how many servers exist. If the infrastructure scales from three servers to five, clients remain unaffected—they continue using the same single point of contact. This abstraction is fundamental to building scalable, maintainable systems.

**Reverse proxies are extremely prevalent in real-world system design, whether for APIs or databases.** Load balancers, API gateways like Kong, and database proxies like ProxySQL are essential components of modern distributed systems. Understanding their operation and capabilities is critical for designing robust, scalable architectures.
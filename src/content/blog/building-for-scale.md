---
title: 'Building for Scale'
description: 'Building for Scale is a series of posts focussed on my experience running FutureFund at scale.'
pubDate: '2024-08-25'
---

Building for Scale is a series of posts focussed on my experience running FutureFund at scale. This is written for two audiences: 1/ people looking to scale their systems with a similar tech stack. 2/ People curious about the challenges scaling a hosted product.

Before we start, it's important to understand our starting position:

1. We are not looking to do this as inexpensively as possible. There are cheaper ways to get things done. If you have zero budget, this is not for you.
2. Innovation is in our product, not our production environment. We want production to be boring and vanilla. Nothing special to see here. The value in these posts is how we stitch everything together.
3. We do not tolerate downtime. None.

From this, we created our DevOps principles:

1. **Make it Unique**
   - Focus our energy on what makes us unique
   - Always buy vs build
   - Use cloud hosted services instead of running services in our own instances

2. **Build it Big**
   - Over-provision services and use redundant services to reduce the chance of downtime
   - Ensure no SPOF (Single Point of Failures) in the architecture
   - Obsess over monitoring and alerting

3. **Keep it Clean**
   - Continually upgrade services and libraries
   - Maintain clean logs
   - Maintain clear and concise documentation

That's it for now. Next time we'll discuss our tech stack and why we chose it.

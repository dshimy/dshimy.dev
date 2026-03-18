---
title: 'Our Stack'
description: 'A high-level overview of our stack at FutureFund.'
pubDate: '2024-09-01'
---

Let's take a look at our core components of our tech stack. Where possible, I'll include the reasoning behind these decisions.

**Ruby on Rails -** I'm a developer at heart and no other framework has brought me as much joy as RoR. Additionally, I'm not a fan of SPA and shy away from JavaScript frameworks whenever I can. Rails + Turbo + Stimulus.js is perfect for me.

**MySQL -** I selected MySQL because I knew it better and you typically use what you know. There is only one feature I wish MySQL supported, UUIDs. Yeah, I know Postgres supports this natively, but I've been married to MySQL for too long to leave for someone else.

**Kubernetes -** Around eight years ago, I felt containers were the future. I attended Google Next, drank a ton of K8s Kool-Aid, and boom, Bob's your uncle. If I was starting today, I would take a serious look at serverless technologies, but for now, K8s continues to serve me very well.

**Google Cloud -** When I was switching to K8s, Google had much better support for K8s than AWS. Today, I'd say they are on par. The cloud services I use are supported in GCP and AWS, switching would not be hard. That being said, I've been in GCP for nine years without issue.

**GitHub -** This is another service that I have used in the past, know it well enough, and don't see any reason to look for something else.

It's important to note, we've strive to stick with "out of the box" Rails. Meaning, we use their recommended tooling. This includes importmaps, Stimulus.js, Turbo, minitest, etc. You won't find rspec, Vue.js, React, etc.

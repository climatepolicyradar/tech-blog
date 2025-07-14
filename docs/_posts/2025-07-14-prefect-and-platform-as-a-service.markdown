---
layout: post
title:  "How we’re using Prefect to develop our Platform-as-a-service"
date:   2025-07-14 12:00:00 +0000
categories: platform
---

How do you build a system that can take nearly two and a half million pages of climate law, policy, funding, litigation, and reporting and make it searchable and accessible? At Climate Policy Radar, this is what we do: we ingest lots of documents, we translate and extract the text, we embed the text in a search engine, and then we run thousands of topic classifiers on them to label mentions of policy instruments, targets, fossil fuels, and anything else policy experts need to find in that morass. As a result, "the pipeline" is one of the most common phrases you'll hear in our tech meetings. Our pipelines are fundamental assets, taking the raw PDFs and generating the structured, accessible, searchable dataset our hundreds of thousands of users around the world rely on.

When you're ingesting tens of thousands of documents in multiple languages, you have to ensure your pipelines are manageable, maintainable, performant, and don't overwhelm your infrastructure. All while still meeting the needs of our policy team, our data science team, and our users. We've dealt with these constraints every day as we've scaled Climate Policy Radar from a handful of documents to our current 30,000 -- averaging 70-80 pages each, and with some behemoths exceeding 3,000 pages. We plan to scale a lot further yet. To support this, our platform team has been building a self-service data platform, and a huge part of that is orchestrating the flows of data from PDF to knowledge graph.

## The Problem: Bottlenecks

We started, like most startups, with greenfield codebases, a small team, and early versions of our infrastructure running on AWS -- using step functions to manage pipeline tasks like PDF text extraction. This was decidedly non-self-service: our data scientists need to run processes that have outgrown their laptops? Ask the platform team. New classifier needing deployment? Ask the platform team. When we were [researching how to do responsible question-answering with LLMs](https://arxiv.org/abs/2410.23902v1), we needed to experiment with millions of synthetic questions and answers to characterise risks and failure modes. 

Our requirements get more complex every month. Scaling our technology is critical.

Our first task was to make the raw text in PDFs searchable, but even that is challenging, requiring us to download, translate, extract the text, and embed the text in our search engine.  Recently, we’ve started rolling out our knowledge graph, which identifies common policy topics in document passages. These capabilities add thousands more topics to process on each document. 

The challenge of this isn't just technical complexity, but human complexity. Before dedicating a team to building out our platform, our data scientists were frustrated waiting for infrastructure support, platform engineers spent more time handling requests than building the platform, and teams were building workarounds to avoid the bottlenecks.

## The Solution: Platform as a service

Our platform team's aim is to build a full platform-as-a-service capability, allowing engineers and data scientists to deploy and manage their own production processes without needing a platform engineer every time. This decouples teams, allowing us to work on distinct capabilities independently. A cornerstone of this transformation has been our orchestration tool, Prefect. Not because it's perfect (we have plenty of complaints about their licensing model), but because it provides huge benefits for our architectural needs.

The platform team also owns our ingest and inference pipelines, managing the huge complexity of compiling and generating our core dataset. The application and data science teams can then define Prefect deployments in their own repos, deploy via Github actions, and run their own production processes and experiments without needing to pull platform team away from their work.

This separation is crucial for a scaling startup like ours -- helping teams to focus on their core goals, iterate quickly, and get things into production without waiting.

## The technical bit

Architecturally, we have the platform layer, managed by the platform team. Pulumi infrastructure as code defines our base Prefect deployment, IAM roles, task definitions — including using Prefect’s Python SDK to define our our base Prefect deployment for things like work pools, work queues and job variables —- and our underlying AWS resources like IAM roles, task definitions and secret.. We've chosen to use Prefect's managed cloud service, but it's open source so you have the option to self-host. As a small team, we prefer to not add more maintenance and operational burden for ourselves!

We're almost solely in AWS, so ECS is an easy choice for our compute layer. We use AWS Secrets Manager over Prefect's secrets blocks so our secrets stay in one place and can be easily ring-fenced to each environment we operate. Similarly this allows the platform team to manage the permissions and role for ECS jobs (e.g. which assets they have access to) centrally. For CI/CD we use Github actions, following well established patterns within our engineering organisation.

Our two main pipelines underpin our entire application layer:

![Our ingestion pipeline](/tech-blog/assets/blog-images/prefect-pipeline-1.png)

**Ingestion Pipeline**: Downloads documents (typically PDFs, though we convert HTML to PDF to maintain one text extraction path) → translates if needed via Google Translate → extracts text with Microsoft DocumentAI (we tested many options; they handle weird PDFs best) → embeds and inserts into our Vespa search index.

![Our knowledge graph pipeline](/tech-blog/assets/blog-images/prefect-pipeline-2.png)

**Knowledge Graph Pipeline**: Runs classifier inference on document batches → indexes labels → derives aggregate concept counts. No point doing this online when batch processing works perfectly.

Some topics yield to straightforward lexical analysis (think of a search engine that simultaneously searches for every possible name something is called), while linguistically complex concepts like net-zero targets demand trained classifiers. We've prioritised simplicity first (building complexity incrementally is one of our architectural principles), reaching for more intense techniques only when the topic demanded it. This has meant we can run the entire pipeline -- except, currently, for a single classifier -- without GPU acceleration, saving significantly on cost.

After months of running production workloads, we've developed some rules of thumb for how we build these systems:

- **Unidirectional flows beat dynamic ones**. We manage state upstream where possible, and avoid reading from downstream services
- **Parent pipelines over triggers**. A control pipeline calling inference, indexing, and counting is clearer than complex trigger chains
- **Stage data after substantial changes**. This makes debugging easier and lets you rerun individual steps
- **Test on production data**. Don't believe a pipeline works until it has processed real documents at scale!
- **Run pipelines more often than needed**. This ensures we surface issues before they become critical
- **Batch operations wisely**. Prefect's ECS task setup can suffer from cold starts, so batch appropriately
- **Watch out for the prefect API limits**. Understanding the plan you’re on and how adding flow and task decorators to your functions will affect API usage is key.

We've resisted premature optimisation. A single slow pipeline that works reliably beats a complex system that fails mysteriously. Good names and schema designs pay dividends. And infrastructure as code is mandatory; unless you want to sacrifice your sanity while developing your platform.

## What's next?

Looking ahead, we're planning to integrate our Prefect infrastructure into our new OpenTelemetry observability stack, and lift and shift some legacy pipeline elements that have lingered on AWS step functions since the early days of our organisation. We've been slowly rolling out knowledge graph functionality on our apps, and that's just the beginning for us: on the roadmap, we've got support for biodiversity and energy domains, local and subnational data, and APIs for external users. All will require performant, maintainable infrastructure, and our platform is critical to that.

But our data scientists now PR code straight into Prefect pipelines; application engineers can write jobs to ship data across our estate; when we had to generate millions of synthetic examples for responsible RAG research we just...did it. Adopting Prefect has given us a huge amount of decoupling, observability, stability and standardisation in our stack. But more than the specific choice of technology, removing barriers and frictions that limit ideas and implementation has been key. Prefect's been invaluable in that journey, even if our budgeting sheets make us wince every time we add a new user.
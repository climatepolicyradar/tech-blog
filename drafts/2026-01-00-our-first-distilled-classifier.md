# Distilling our first live topic classifier from an LLM

*Kalyan Dutia*

We've just reached a milestone in data science that it's worth writing on the internet about – publishing our first topic classifier which makes use of the gains in NLP abilities recent LLMs have given us, without bearing the cost or environmental impact of running a state-of-the-art LLM in production.

If you're already familiar with our knowledge graph, feel free to [skip ahead to 'Finance Flow: a high impact but challenging topic'](#finance-flow-a-high-impact-but-challenging-topic).

If you just want to hear our tips and tricks from distilling a BERT model from a state-of-the-art LLM, jump to [Distilling our first BERT classifier from an LLM](#distilling-our-first-bert-classifier-from-an-llm).

## Our knowledge graph and topic classifiers

We maintain a knowledge graph of climate-related topics that are valuable for our users to be able to find in documents. For each topic live in our tools, we build a classifier: a way of finding mentions of a topic in a passage of text. You see the results of these classifiers when you select the topic filters on our tools.

<screenshot of topic filters goes here>

We currently have <n> topics in the knowledge graph, each powered by its own classifier. All but one of these classifiers so far work without any machine learning – they compile all of the labels from a concept in our concept store into a regex pattern, and we use that regex pattern as the 'classifier'. The one exception to this is our targets classifier. As the task of identifying whether a passage mentions a target is too complex to be handled by a regex pattern, we built this by finetuning a BERT model (TODO footnote ICLR paper). This took us months and required thousands of expert-labelled passages and iteration on our methodology (TODO: footnote link to methodology) – not a rate of development our team could keep up!

## Finance flow: a high impact but challenging topic

Identifying the transfer of finance is a high impact topic for us, especially as we host [a platform containing all of the project documents and policies from the Multilateral Climate Funds](https://climateprojectexplorer.org/).

Having a look in our concept store (TODO footnote to concept store and article), the definition illustrates this:

> A finance flow is an economic flow that reflects the creation, transformation, exchange, transfer, or extinction of economic value and involves changes in ownership of goods and/or financial assets, the provision of services, or the provision of labor and capital. Ideally, a financial flow describes four elements: the source (who is sending the financial asset, such as a bank or organisation); the financial instrument or mechanism (how it is being sent, such as a grant, loan or a subsidy); the use or destination (the purpose for which the asset will be used, which is often expressed as the recipient organisation or their sectoral categorisation); and the value (which can, but does not need to be, expressed in monetary terms directly).
> *[finance flow (Q1829) – Climate Policy Radar concept store](https://climatepolicyradar.wikibase.cloud/wiki/Item:Q1829)*

This is too challenging for a set of regex rules, or pretrained NER models. A classifier trained to detect monetary values would also include stocks – amounts held, rather than transferred. A regex statement like `grant|loan|subsidy` falls into the trap of financial metaphors in the English language – *"I grant you permission..."*.

## Distilling our first BERT classifier from an LLM

Internally, we've been working on ways to adapt [active learning](todo wikipedia) to make the most of LLMs in our context. We'll write more about that in a future post; our first learning was that state of the art models (GPT5 era) are performant enough zero-shot classifiers for a concept as complex as *finance flow*.

TODO:

- the process we roughly followed, and the results
- what we learnt
  - GPT5 gives a marked jump in performance for classification tasks
  - putting a prompt in the hands of policy experts gives a really nice separation between teams. policy produce the best LLM classifier (with data science help) and vibe check that; data science can then be responsible for producing the best BERT model
  - modernBERT shows no real improvement over climateBERT on these tasks
- next steps
  - we're currently working on distributive justice - this is harder
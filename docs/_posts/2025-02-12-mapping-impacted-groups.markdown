---
layout: post
title:  "Mapping the language of impacted groups"
date:   2025-02-12 09:07:33 +0000
categories: data-science
---

Our [main blog](https://www.climatepolicyradar.org/latest) features Anne Sietsma's exploration of [how we can analyse and visualise language patterns in climate policy](https://www.climatepolicyradar.org/latest/mapping-language-part-one) documents across different regions and contexts.

> Making climate policy easy to find and use is especially important for communities that are most vulnerable to climate change. Finding climate policies relevant to these communities is not a simple task. The Climate Policy Radar app has already collected the policies and extracted their text, so selecting text based on keywords is an obvious first step but language is often very context-dependent.
 
> To help us learn more about how climate terminology changes from country to country, we recently developed a little mapping tool.

The analysis used our public [climate policy dataset](https://huggingface.co/datasets/ClimatePolicyRadar/all-document-text-data) hosted on Hugging Face, loaded into DuckDB for efficient text search across millions of paragraphs. A streamlit app 
using GeoPandas was used to visualise the data, which you can [explore here](https://maps.labs.climatepolicyradar.org/). 

Check out the full post to learn more about the analysis, including handling data normalisation across different countries, technical considerations in mapping sparse
datasets, and balancing technical efficiency with inclusivity requirements. 

If you're interested in the technical implementation, all our code is open source and [available on Github](https://github.com/climatepolicyradar/open-data/tree/main/src/streamlit_apps). We welcome contributions and feedback!

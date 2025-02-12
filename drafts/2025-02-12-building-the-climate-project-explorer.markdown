---
layout: post
title:  "Building the Climate Project Explorer"
date:   2025-02-12 09:07:33 +0000
categories: application
---

The [Climate Project Explorer](https://climateprojectexplorer.org/) is a platform for finding publicly available information about projects funded by the Multilateral Climate Funds (MCF). It [launched during COP29](https://www.greenclimate.fund/news/multilateral-climate-funds-launch-climate-project-explorer-cop29), at a joint event between the Head of the MCFs and the COP29 Presidency, and we have been delighted by the response to it. 

In this post, we'll look at some of the challenges we faced: harmonising data from different sources, getting data in as quickly as possible, and managing multiple custom front-ends.

### Different funds, different data formats 

The MCF includes four funds ([Green Climate Fund](https://greenclimate.fund/), [Adaptation Fund](https://www.adaptation-fund.org/), [Climate Investment Funds](https://www.climateinvestmentfunds.org/), and [Global Environment Facility](https://www.gef.org/)). As such, we knew each fund would have different ways of working, different ways of organising and different ways of funding their own projects. 

From our perspective (when building the Climate Project Explorer), that meant the funds were also likely to have different ways of managing and organising their data. 

We needed to find a solution for handling data that would be sent in different formats. That could be in a spreadsheet (where there would not be the same columns and fields used to structure information across the four funds) or in a widely accepted digital format like JSON[^json]. 

[^json]: A JSON file is a text-based file that stores data in a format that's both human-readable and machine-parsable. It's a common format used in data analysis, software engineering and the like.

We asked the funds to send us data in one of these two formats, to make it easier to parse and transform. For the parsing and transforming, we created ‘datamappers’. These tools parse, extract and store the information we need from a data set and send that to another tool that writes this information to our database, matching how we organise data. 

We set up common properties we wanted to extract from the data sets. These included properties like: when a project started, the regions a project covered, which agencies implemented the project, and more. This allowed us to render this information in our frontend tools, and use the data in a host of different ways. 

Instead of building a single datamapper, we decided to create one for each fund. This helped us with a lot of things, one of those being flexibility. We knew every vendor would have different data requirements. Having individual datamappers allowed us to tailor the transformation process without worrying about compatibility issues. We found it also allowed us to quickly identify issues with the dataset in our test runs. That meant we could easily return to the individual fund for data correction or improvement, which allowed us to keep the data quality high even while we were working at pace. 

Separate datamappers also allowed us to work in parallel. That means that multiple engineers could work on separate datamappers at the same time. Any conversations we had with a fund about their dataset did not hold up the mapping process as whole (which it would have been if we had used one datamapper for all of the funds). 

One of the main challenges we found when mapping the data was normalisation -- organising it to be easier to use, for example by removing duplication and standardising data formats. This meant a lot of back and forth to ensure that the data fit into our expected data format. It made developing slightly more difficult, as we were introducing more safeguards to ensure the data would not fail the import or ingest steps (further down the line).  

### Getting data in, faster 

We first needed to incorporate the data on MCF projects into our database. Doing that means that everyone can browse or search for anything on the Climate Project Explorer app. 

We built a CLI (Command Line Interface) tool which connects to a dedicated new endpoint in our existing admin service. That lets us reuse the majority of the existing logic for saving and validating data. Since data integrity is crucial to our product, the whole process had to be idempotent. (This means that, no matter how many times you do something, it does not change.) We achieved this by running the entire import inside a single database transaction, and only attempting to save data that didn’t already exist. 

We soon discovered that, with datasets in the 1000s, each bulk import would take almost an hour. Unfortunately, this meant that the HTTP calls to the endpoint would timeout before the import process was actually completed. In turn, this forced us to make it an asynchronous task triggered by the CLI tool, which then triggered Slack alerts when it completed. This worked fine during the development of the Climate Project Explorer because the amount of data we brought together from the MCFs was relatively small. 

If we want to import larger datasets in the future, though, it is not scalable. It ties up the database that both our main application and an admin web app use for a significant amount of time. Therefore, larger datasets will be more challenging to import quickly without affecting our users. We have a couple of ideas for how we might do this, and it is an interesting challenge as we look to add more data in the future.

### Evolving our front-end 

Over the last year or so, we’ve been evolving our front-end repository. That has meant taking it from a repository we could use to build a single application, into what we are calling a “theme factory”. We can use this collection of scripts to build a unique, “themed” version of the front-end. 

For each theme, we can customise a number of aspects of the application, like display options (including colours and typography) as well as content pages. We did this by a combination of environment variables and making the most of the Next.js build options.

To better be able to customise our themes, we needed to evolve the theme factory further. Before working on the Climate Project Explorer, there were two areas we had not tackled: data and content. Rather like how a factory might be given a blueprint, or configuration of what to build, our factory can now be provided with a configuration file that indicates how the application is to be built. The “config” file can control key aspects of the app, such as the available filters, data types and even labels used within search.

This results in us being able to quickly and easily create bespoke experiences of search between each application. Our applications can query only the information they are concerned with, without losing out on any of the underlying functionality.

The other reason for taking this approach is to ensure our application is future-proofed. Currently we define the configuration files in code, and (hopefully) in the future we will be calling an API to gather the application’s configuration from a CMS. The idea is that this can all be coordinated at the infrastructure layer.

### Managing multiple custom apps 

Climate Policy Radar manages three websites: our own, Climate Change Laws of the World (CCLW), and the new Climate Project Explorer (CPE). While we want to display all aggregated climate laws and policies on our website, CCLW and CPE users require different data subsets. For instance, CPE users are interested in project and guidance data for four multi-lateral climate funds, but not in climate laws. Conversely, CCLW users focus on climate laws and policies, excluding multi-lateral climate fund data.

To address this, we created a JSON Web Token (JWT) for each website, containing an encrypted version of the relevant data we wanted to show on each website. Our rationale includes:

1. Security: JWTs are signed, ensuring the authenticity of the contained data and preventing tampering. Only authorised users can generate a token.  
2. Decentralisation: Each website can independently validate the JWT without querying a central server for permissions, reducing server load and enabling faster responses.  
3. Flexibility: The JWT payload can include any data, such as allowed corpus IDs, allowing each website to specify the exact data needed without hardcoding permissions in the backend.  
4. Scalability: As the number of websites grows or access requirements change, we can issue new tokens with updated permissions without altering the core application logic.  
5. Expiration and Revocation: JWTs can have expiration times, limiting access over time. If a website no longer needs certain data, we can exclude it from the JWT, ensuring it won’t appear on the site. This is also useful for curating datasets that aren’t ready for publication.

| \`\`\`json {   "allowed\_corpora\_ids": \[     "CCLW.corpus.i00000001.n0000",     "CPR.corpus.i00000001.n0000",     "CPR.corpus.i00000589.n0000",     "CPR.corpus.i00000591.n0000",     "CPR.corpus.i00000592.n0000",     "MCF.corpus.AF.Guidance",     "MCF.corpus.AF.n0000",     "MCF.corpus.CIF.Guidance",     "MCF.corpus.CIF.n0000",     "MCF.corpus.GCF.Guidance",     "MCF.corpus.GCF.n0000",     "MCF.corpus.GEF.Guidance",     "MCF.corpus.GEF.n0000",     "OEP.corpus.i00000001.n0000",     "UNFCCC.corpus.i00000001.n0000"   \],   "exp": 2053007034,   "iat": 1737474234,   "iss": "Climate Policy Radar",   "sub": "CPR",   "aud": "app.climatepolicyradar.org" } \`\`\` | \`\`\`json {   "allowed\_corpora\_ids": \[     "CCLW.corpus.i00000001.n0000",     "CPR.corpus.i00000001.n0000",     "CPR.corpus.i00000591.n0000",     "CPR.corpus.i00000592.n0000",     "UNFCCC.corpus.i00000001.n0000"   \],   "exp": 2050054479,   "iat": 1734521679,   "iss": "Climate Policy Radar",   "sub": "CCLW",   "aud": "climate-laws.org" } \`\`\` | \`\`\`json {   "allowed\_corpora\_ids": \[     "MCF.corpus.AF.Guidance",     "MCF.corpus.AF.n0000",     "MCF.corpus.CIF.Guidance",     "MCF.corpus.CIF.n0000",     "MCF.corpus.GCF.Guidance",     "MCF.corpus.GCF.n0000",     "MCF.corpus.GEF.Guidance",     "MCF.corpus.GEF.n0000"   \],   "exp": 2052915353,   "iat": 1737382553,   "iss": "Climate Policy Radar",   "sub": "CPR",   "aud": "app.climatepolicyradar.org" } \`\`\` |
| :---- | :---- | :---- |
| Climate Policy Radar | Climate Change Laws of the World | Climate Project Explorer |

Caption: Decoded JWT for each website

### Thanks everyone\! 

This post was written by Anna, Katy, Patrick and Osneil (and edited by Ella), but the Climate Project Explorer is built and maintained by a much bigger team. Thank you to everyone at the MCFs and CPR who worked on this project\! 
---
layout: post
title:  "Data engineering principles according to Gatis Seja"
author: mikulskibartosz
tags: [ 'Data engineering', 'Principles' ]
description: "Lessons learnt from Gatis Seja's presentation about data engineering principles"
featured: false
hidden: false
excerpt: "Lessons learnt from Gatis Seja's presentation about data engineering principles"
---

Recently, I have watched an impressive presentation about the principles which work very well in data engineering teams: "Data Engineering Principles - Build frameworks, not pipelines" by Gatis Seja.

<iframe width="560" height="315" src="https://www.youtube.com/embed/pzfgbSfzhXg" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

In this talk, he recommends thinking about data engineering as creating a data product. It means that the outcome produced by the team must be "ready to use."

The users should not need to investigate the pipeline to figure out how the data was generated. They should not worry whether the data is correct. The data product must be trustworthy. If the data consumers get anything from us, it means it is correct.

# Understand the data consumers

The most important principle is to understand the needs of data consumers. We should know how the data is going to be used, so we can write validation rules to ensure that the outcome is useful for the consumers. We should have a service level agreement which not only tells the consumers how often we update the data but also what happens when the validation fails.

{% include info.html %}

# Validation

Speaking of validation. Gatis suggests two ways of dealing with validation failures. The first one is to stop the processing of the whole batch when validation fails and wait until the team fixes the problem. 
The consumers will need to wait for their data, but they can be sure that everything you give them is correct.

Gatis thinks this approach is better than the one described below. I totally agree with him. It is better to wait until you have correct data than deal with reprocessing and updating incorrectly calculated aggregates.

The second approach is to generate whatever you can and skip the invalid data points. In this case, you always provide the consumers with something correct, but it may have missing values.

When you fix the problem, the consumers must be ready to retrieve the missing values. They may need to repeat some computation or explain to the final user why they see uncomplete reports.

# Keep the raw data and intermediate steps data

We should be able to repeat computation when we need to, so we must always make a copy of the input data. After all, the original data source may change or be unavailable in the future. 

It is essential because when we spot a bug in the data pipeline, we want to reprocess the pipeline for the affected days. For that, we need the exact data which was available on those days.

We may also want to store the transformed data from the intermediate steps of the transformation because it makes it easier to debug the pipeline when something weird is happening. We look at the outputs of every step to locate the one which produces incorrect output.

# Validate input and output

That one should be a no-brainer. When we extract data from the input data source, we must not only store the extracted raw data but also make sure that we have useful quality data, to begin with. After all, "garbage-in, garbage-out" still applies. 

When the validation of input data fails, we should store the invalid raw input into a separate collection (call it "extracted_failed" or in other way indicate that those values are invalid), so we can take a look at them and notify the team responsible for the data source about the problem.

Similarly, we must validate the output before we load it into the production data warehouse. Gatis suggests creating a staging data warehouse, load the output there, and run validation rules to make sure that the consumers of the data warehouse can use the data.

When we make sure that the data is valid, we copy the data from the staging warehouse into the production warehouse.

During the output validation, we should check the schema (primarily if the schema is enforced on-read) and the business rules.

# Monitoring is part of the product

Monitoring is not only nice-to-have. It is crucial. Not only for the data team but also for the consumers. If you implement a data product, the consumers must be able to check at any time the performance of the data processing pipeline and the validation results.

Obviously, it is also the most useful thing for the data team. It allows us to check which parts of the data pipeline are slow or getting slower. For me, it is also vital to add the number of processed data points to the monitoring dashboard. A sudden, unexpected change of that value is usually a sign of a severe problem.

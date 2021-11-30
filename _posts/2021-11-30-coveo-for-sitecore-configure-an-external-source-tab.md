---
layout: post
title: 'Coveo for Sitecore: Configure an External Source Tab without Sitecore Items'
description: Coveo for Sitecore comes with an out-of-the-box tab component that can be useful for switching between different types of Sitecore content. I will go over how to supress Sitecore content to only reveal External content.
image: assets/images/pic09.jpg
---

Coveo for Sitecore (Coveo Hive) comes with an out-of-the-box Coveo Tab component that can be useful for switching between different types of Sitecore content, but lacks a direct way within Sitecore to *only show External Content without Sitecore content*.

If you're team is lucky enough to have the Enterprise version, then you may be able to get away with adding a rule within the Coveo Tab Datasource field `Filter expression rules`, but the rule "where <u>specific</u> field <u>compares to</u> <u>specific value</u>" doesn't allow you to make a blanket statement rule to not inlude a document that contains a field, which is enough to keep Sitecore documents out of the mix.

A solution I recommend (for both Pro and Enterprise) is to use a Pipeline Advanced Filter. The Datasource for Coveo Tab contains the field `Names of external content sources` that will append external coveo sources from your organization to the Coveo API call. It also contains the field `Pipeline` for determining which Query Pipeline you would like the request to filter through within the Coveo Admin Portal. The following steps should be taken:

1. Add all of your external source names from the Coveo Portal to the field `Names of external content sources`:
<img src="{% link assets/images/2021-11-30_field-names-of-external-sources.png %}" alt="coveo names of external content sources" />

2. Create a new Pipeline in Coveo Portal that your external sources will flow through and paste that Pipeline name into the `Pipeline` field:
<img src="{% link assets/images/2021-11-30_coveo-portal-pipeline-community.png %}" alt="new coveo pipeline for external sources" />
<img src="{% link assets/images/2021-11-30_coveo-pipeline-field.png %}" alt="new coveo pipeline for external sources" />

3. Within the Coveo Portal, double-click on the created external source Pipeline and navigated to the `Advanced` tab and stay on the right-rail Filters section. Click on the `Add rule` button and add a new cq (constant query) parameter with the expression `NOT @alltemplates`. The condition section can be left empty since we never want to include Sitecore documents. The `@alltemplates` field is specific to Sitecore documents, but feel free to use any other field if your external source meta data has created this field:
<img src="{% link assets/images/2021-11-30_coveo-externalsource-pipeline-filter-expression-configure.png %}" alt="coveo pipeline filter expression to remove sitecore documents" />
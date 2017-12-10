---
layout: single
title: Global Azure Bootcamp - Aftermath
date: 2015-05-19 18:23:58.000000000 +02:00
type: post
categories:
- ASP.NET MVC
- General
tags: []
---

It's been almost a month now since the last edition of [Global Azure Bootcamp](http://global.azurebootcamp.net/), and I wanted to share with you some of the details of my involvement in this years edition. This year, we had two labs in which you could participate: a racing and a [science lab](http://global.azurebootcamp.net/global-azure-bootcamp-science-lab-2015/). And I helped out with some aspects behind the workings of the science lab: the [Elasticsearch](https://www.elastic.co/products/elasticsearch) cluster and the dashboard website.

For this lab, all you had to do was to deploy a cloud service to Azure and scale it up to as many instances as you could spare. These instances would then crunch a lot of data concerning breast cancer research. After the results from your cloud instances were uploaded, they would eventually end up in the Elasticsearch cluster.


# Elasticsearch
This cluster was running on three D11 Azure Virtual Machines (2 cores, 14 GB RAM), so we had one master node and two children, each of them maintaining a full set of the data shards and copies for faster querying and redundancy. At the time, we had no real clue about the amount of data that would end up in this cluster, so we decided to store the data on a separate 200 GB disk (one for each virtual machine). We could easily attach more storage later on if that would've been necessary.

# Lab Dashboard
The dashboard site was created using [ASP.NET MVC](http://www.asp.net/mvc), a tiny bit of [Knockout](http://knockoutjs.com/) and the charts were made using the awesome [Chart.js](http://www.chartjs.org/) library. Initially I tried using [D3](http://d3js.org/), but that would've cost me a lot more time to get used to. Chart.js really was a lifesaver here considering the time frame in which the dashboard site has been created.

# Statistics
We all love statistics, don't we :-)

First of all, the cluster: running on three D11-sized virtual machines. In the end, the Elasticsearch nodes each held **5.378.542** documents, that's over 5 million updates that each of your cloud instances have sent to us when they finished working on a piece of research! And that data ended up taking *only* **959 MB** of disk space. Meanwhile, the cluster nodes were only using about **4 GB** of their available RAM, and no swap at all (because they were running on Ubuntu, but you didn't get that information from me ;-) ).
The dashboard was running as a Web App on two **Standard S2** instances (2 cores and 3.5 GB of RAM per instance) and didn't have any issues considering load whatsoever [^1].
In the end, these services were running for three to four days and ended up costing only 41,99 € for the Elasticsearch cluster and 27,70 € for the web app, including all extra costs like storage and bandwidth.
Azure sure is an awesome platform :-)

[^1]: There was an issue though with data caching, which I didn't manage to pin down: some charts/numbers appeared to be difficult to load, unless you refreshed the page several times, giving the dashboard the chance to fetch previously loaded data from its cache. This was especially the case on the main page, which displayed the top 10 charts and the metrics page, which showed some totals (number of participating countries, locations, attendees, etc.).

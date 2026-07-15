---
name: wikipedia_api_curl
description: How to fetch Wikipedia article data using curl via the REST API and MediaWiki
  Action API, including endpoints, parameters, and response formats.
tags:
- wikipedia
- api
- curl
- mediawiki
- rest-api
- json
- cli
created_at: '2026-07-15T10:32:03.551460+00:00'
---

Wikipedia provides two main APIs for fetching data with curl:

1. Wikipedia REST API (https://en.wikipedia.org/api/rest_v1)
   - Page summary: curl -s "https://en.wikipedia.org/api/rest_v1/page/summary/Albert_Einstein"
   - Full page HTML: curl -s "https://en.wikipedia.org/api/rest_v1/page/html/Albert_Einstein"
   - Returns JSON with fields: title, extract, description, thumbnail, content_urls.

2. MediaWiki Action API (https://en.wikipedia.org/w/api.php)
   - Get plain-text intro extract:
     curl -s "https://en.wikipedia.org/w/api.php?action=query&titles=Albert_Einstein&prop=extracts&exintro&explaintext&format=json" -H "User-Agent: MyApp/1.0 (contact@example.com)"
   - Search pages:
     curl -s "https://en.wikipedia.org/w/api.php?action=query&list=search&srsearch=Albert%20Einstein&format=json" -H "User-Agent: MyApp/1.0 (contact@example.com)"

Important parameters:
- format=json (also xml, php)
- action=query for reads; list=search for searching
- prop=extracts for article text; exintro for lead only; explaintext for plain text
- titles=Page_Name (pipe-separated for multiple)
- User-Agent header is required by Wikimedia; include app name and contact

Response formats:
- REST API: flat JSON with title, extract, etc.
- MediaWiki API: JSON with pages keyed by numeric pageid under query.pages

Guideline: Use REST API for quick summaries; use Action API for search or batch metadata.

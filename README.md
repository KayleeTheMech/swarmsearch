# Swarmsearch
Decentralized Search, Crawling, Filtering and Tagging

Local Index, federated with other Indices.

Shared Bloom Filters for avoiding SEO/AI Slop and search improvements.

## Requirements Plan:
- Backend components written in Rust.
- Frontend components minimize Javascript? Best mostly HTML + CSS


Decentralized and federated:
- the search index is decentralized and federated
- Decentralized and federated subscription service for bloom filters to mark for e.g. bias in URLs (like shinigami eyes) and enable filtering of search results based on custom user filters
- there are several ways these bloom filters should be able to be used:
  	- they add a label to a source
	- the filter can prevent adding this source to the index: for example SEO or AI generated results
	- the filter can disfavor this result (e.g. low quality source, propaganda channel, transphobia)
	- the filter can favor a result (e.g. credible source: for example wikipedia, trusted research institution)
	- plain informative (government source)    

### Backend Components:
- local search index service
- local web-crawler service that feeds the local search index on a pre-determined bandwidth limit
- a search engine that works on the local index with subscribed filters + is able to send out decentralized queries to other indices 

### Frontend Components:
- a search interface that connects to the local search engine
    - regular old search query page
    - it displays results
	- it displays subscribed filters next to results
	- enables fast tagging of webpages as belonging into a filter OR do not belong into a filter
    - enables configuring whether this search should disfavor or favor a certain result
- a firefox plugin that enables quick additions to filters (like shinigami eyes)

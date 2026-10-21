Search engine plan:

- Backend components written in Rust.
- Frontend components minimize Javascript? Best mostly HTML + CSS

Backend Components:
- local search index service
- local web-crawler service that feeds the local search index on a pre-determined bandwidth limit
- a search engine that works on the local index with subscribed filters + is able to send out decentralized queries to other indices 

Decentralized and federated:
- the search index is decentralized and federated
- Decentralized and federated subscription service for bloom filters to mark for e.g. bias in URLs (like shinigami eyes) and enable filtering of search results based on custom user filters
- there are several ways these bloom filters should be able to be used:
	- prevent adding this source to the index: for example SEO or AI generated results
	- disfavor this result (e.g. low quality source, propaganda channel, transphobia)
	- favor this result (e.g. credible source: for example wikipedia, trusted research institution)
	- plain informative (government source)    


 

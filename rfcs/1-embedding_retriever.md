- Start Date: (2022-10-04)  
  RFC PR: (fill after opening the PR)  
  Github Issue: (if available, link the issue containing the original request for this change)  
  # Summary  
    
- EmbeddingRetriever doesn't allow Haystack users to provide new embedding methods and is 
  currently constricted to farm, transformers, sentence transformers, and OpenAI based 
  embedding approaches. Any new encoding methods need to be explicitly added to Haystack 
  and registered with the EmbeddingRetriever.  
 
- We should provide pluggable approach to improve EmbeddingRetriever's ability to use 
  completely new embedding methods. For example, a Haystack user should be able to 
  add Co:here embeddings without having to commit additional code to Haystack repository.

  # Basic example  
    EmbeddingRetriever is instantiated with:  

  ``` python
	  retriever = EmbeddingRetriever(
	      document_store=document_store,
	      embedding_model="sentence-transformers/multi-qa-mpnet-base-dot-v1",
	      model_format="sentence_transformers",
	  )
  ```
- The current approach doesn't provide a pluggable abstraction point of injection but 
  rather attempts to satisfy various embedding methodologies by having a lot of 
  parameters which keep ever expanding.

- A new approach allows creation of the underlying embedding mechanism (EmbeddingEncoder) 
  which is then in turn plugged into EmbeddingRetriever.  For example:  
 
  ``` python
	  encoder = OpenAIEmbeddingEncoder.create(api_key="asdfklklja", 
	                                          query_model="text-search-babbage-query-001",
	                                          doc_model="text-search-babbage-doc-001")
  ```

- EmbeddingEncoder is then used for the creation of EmbeddingRetriever. EmbeddingRetriever 
  init method doesn't get polluted with additional parameters as all of the peculiarities 
  of a particular encoder methodology are contained on in its abstraction layer.  

  ``` python
	  retriever = EmbeddingRetriever(
	      document_store=document_store,
	      encoder=encoder
	  )
  ```
  
  # Motivation

- Why are we doing this? What use cases does it support? What is the expected  
  outcome?  
    
  We could certainly keep the current solution as is; it does implement a decent level 
  of composition/decoration to lower coupling between EmbeddingRetriever and the underlying 
  mechanism of embedding (sentence transformers, OpenAI, etc). However, the current mechanism 
  in place basically hard-codes available embedding implementations and prevents our users from 
  adding new embedding mechanism by themselves outside of Haystack repository. We also might 
  want to have a non-public dC embedding mechanism in the future. In the current design a non-public 
  dC embedding mechanism would be impractical. In addition, the more underlying implementations we 
  add we'll continue to "pollute" EmbeddingRetriever init method with more and more parameters. 
  This is certainly less than ideal long term.  

  # Detailed design  

- Our new EmbeddingRetriever would still wrap the underlying encoding mechanism in the form of 
  _BaseEmbeddingEncoder. _BaseEmbeddingEncoder still needs to implement methods:  
	- embed_queries  
	- embed_documents 
- The new design approach differs is in the creation of EmbeddingRetriever - rather than hiding the underlying encoding 
  mechanism one could simply create the EmbeddingRetriever with a specific encoder directly. For example:  

  ```
	  retriever = EmbeddingRetriever(
	      document_store=document_store,
	      encoder=OpenAIEmbeddingEncoder(api_key="asdfklklja", model="ada"),
	      #additional EmbeddingRetriever-abstraction-level parameters
	  )
  ```
 
- In some unforeseeable case where the "two-step approach" of EmbeddingRetriever initialization is no longer preferred 
  we might simply add the EmbeddingRetriever subclass for every supported encoding approach. For example we could have 
  OpenAIEmbeddingRetriever subclass of EmbeddingRetriever and an init method containing only OpenAI embeddings related 
  primitive parameters - if that is a requirement or a preferred approach for embedding retrievers.  
 
- The proposed approach for EmeddingRetriever will likely be an issue for the current schema generation and 
  loading/saving via YAML pipelines. The main reason is using non-primitive parameters in the class' **init()** constructor. 
  To overcome this issue, we might create a thin layer of EmbeddingRetriever classes that have init constructors with 
  primitive parameters only.  

- Therefore, we'll likely have OpenAIEmbeddingRetriever, CohereEmbeddingRetriever, SentenceTransformerEmbeddingRetriever 
  and so on. Each of these retrievers will delegate the bulk of the work to an existing EmbeddingRetriever with a 
  per-class-specific EmbeddingEncoder set in the class constructor (for that custom encoding part). We'll get the best 
  of both worlds. Each <Specific>EmeddingRetriever will have only the relevant primitives parameters for the **init()** 
  constructor; the underlying EmbeddingRetriever attribute in <Specific>EmeddingRetriever will handle most of the business 
  logic of retrieving, yet each retriever will use an appropriate per-class-specific EmbeddingEncoder for the custom 
  encoding part.  
 
  # Drawbacks  
- The main shortcoming are:  
	- The "two-step approach" in EmbeddingRetriever initialization  
	- Likely be an issue for the current schema generation and loading/saving via YAML pipelines (see solution above)  
	- It is a API breaking change so it'll require code update for all EmbeddingRetriever usage both in our codebase and for Haystack users  
	- Can only be done in major release along with other breaking changes  

  # Alternatives  
    
  We could certainly keep everything as is :-)

  # Adoption strategy  
- As it is a breaking change, we should implement it for the next major release.  
 
  # How we teach this  
- This change would require only a minor change in documentation.  
- The concept of embedding retriever remains, just the mechanics are slightly changed  
- All docs and tutorials need to be updated  
- Haystack users are informed about a possibility to create and use their own encoders for embedding retriever.  
- # Unresolved questions  
    
  Optional, but suggested for first drafts. What parts of the design are still  
  TBD?  
 
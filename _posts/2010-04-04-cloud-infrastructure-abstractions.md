---
layout: post
title: Cloud Infrastructure Abstractions
---

# {{page.title}}

<span class="meta">April 4 2010</span>

The essence of programming is abstraction. We begin with hardware and machine code primitives and build a series of increasingly powerful software abstractions on top of them. We can then use these abstractions to write sophisticated user-facing applications that would be impossible to define directly in terms of the underlying primitives.

Programmers clearly understand this abstraction story at the level of a single machine. Less appreciated is the corresponding story evolving at the cloud scale.

As in the single machine case, the cloud offers a handful of primitives: ephemeral compute and attached block storage for example. One can build reasonably complex applications directly on top of them by applying existing single-machine abstractions to each node in an application cluster. But there is a need for more powerful abstractions in this new cloud ecosystem. In particular, cloud applications can be developed not only in terms of single-machine libraries but also cross-machine services. These services will form a network of increasingly powerful abstractions over the raw compute and storage primitives of the cloud.

When such services are available, application developers will spend less time on infrastructure concerns orthogonal to their business goals. For example, they will no longer be concerned with installing and maintaining their own datastores; they will be accessed as services. Web application servers will not be orchestrated by developers directly on top of raw, ephemeral compute clouds; they will be managed with simple API calls. Algorithms will also be abstracted; everything from image processing to numerical optimization routines will be accessible externally.

By defining applications in terms of these infrastructure abstractions, developers can focus more on providing real value to their users. The ultimate effect of cloud infrastructure abstractions will therefore be better user-facing software.

To be concrete, I've included below a sample of cloud infrastructure abstractions that are or should be available to application developers. I've also started adding links to existing providers. If you know of good provider that I should add, say so in the comments.

* App servers (e.g. for Ruby, Python, PHP, JVM, .NET, JavaScript) [Heroku](http://heroku.com), [Stax](http://stax.net)
* Data stores (e.g. Postgres, Drizzle, Redis, CouchDB, Voldermort, Riak, Cassandra, HBase, Neo4j) [SimpleDB](http://aws.amazon.com/simpledb/), [Amazon RDS](http://aws.amazon.com/rds/), [FathomDB](http://fathomdb.com), [MongoHQ](http://mongohq.com), [Cloudant](https://cloudant.com)
* Metadata/consensus stores (e.g. ZooKeeper)
* Cache stores (e.g. Memcache)
* Queues (e.g. FIFO, priority, and messaging) [Amazon SQS](http://aws.amazon.com/sqs/) 
* Message buses (e.g. coalescing, splitting, merging, filtering, fanning)
* Poll/event converters
* Abstracted distributed computing (e.g. map reduce, data flow, graph iteration, message passing) [Amazon EMR](http://aws.amazon.com/elasticmapreduce/)
* Remote language-specific computing [PiCloud](http://picloud.com)
* Load balancing and HA proxying [Amazon ELB](http://aws.amazon.com/elasticloadbalancing/)
* Web fetching
* Web crawling [80legs](http://80legs.com/)
* Non-API site scraping and eventing 
* Blog, twitter, and IRC event pumping
* Email, text, phone, and xmpp sending and receiving [SendGrid](http://sendgrid.com), [Twilio](http://www.twilio.com)
* Payment accepting, processing, and management [Amazon FPS](http://aws.amazon.com/fps/), [Chargify](http://chargify.com/)
* Exception collection and tracking [hoptoad](http://hoptoadapp.com), [Exceptional](http://www.getexceptional.com)
* Test execution and continuous integration
* Log collection, analysis, and search
* Pageview, usage pattern, and A/B analytics
* Text-from-markup extraction
* RSS parsing [SuperFeedr](http://superfeedr.com)
* Image processing
* Audio and video encoding [ZenCoder](http://zencoder.com)
* Optical character recognition
* Audio transcribing
* Natural language date and relative-time parsing [TimeAPI](http://www.timeapi.org)
* Media streaming and buffering
* Source code compilation
* Markup rendering
* Figure, chart, and graph rendering [Google Charts](http://code.google.com/apis/charttools)
* 3D rendering
* Syntax highlighting 
* Web and user corpus full text search [Yahoo BOSS](http://developer.yahoo.com/search/boss), [websolr](http://websolr.com)
* GIS storage and indexing [SimpleGeo](http://simplegeo.com)
* Graph storage and indexing
* Document format conversion
* Spell correction
* Text and audio translation
* JavaScript compression and obfuscation
* Natural language processing
* Machine learning
* Collaborative filtering [DirectedEdge](http://www.directededge.com)
* Trend analysis
* Graph analysis
* Statistical analysis
* Satisfiability determination
* Logistics and graph planning
* Real-time auction facilitation
* Real time multi-user document editing, text chat, and video conferencing

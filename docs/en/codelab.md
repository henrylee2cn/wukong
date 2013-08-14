Wukong Engine Codelab
====

**don't read. translation in progress**

At the end of this codelab, you will be able to write a simple full-text search website using Wukong engine.

If you do not know Go yet, [here](http://tour.golang.org/#1) is a tutorial.

## Engine basics

Engine handles user requests, segmentation, indexing and sorting in their own goroutines.

1. main goroutines, are responsible for sending and receiving user requests
2. segmenter goroutines
3. indexer goroutines, for building inverted index and lookup
4. ranker goroutines, for scoring and ranking documents

![](https://raw.github.com/huichen/wukong/master/docs/en/wukong_en.png)

**Indexing pipeline**

When a document is added, the main goroutine sends the doc through a channel to a segmenter goroutine, which segments the text and passes it to a indexer. Indexer build doc index from search keyword. Inverted index table is stored in memory for quick lookups.

**Search pipeline**

Main goroutine receives a user request, segments the query and passes it to an indexer goroutine. The indexer looks up corresponding documents for each search keyword and applies logic operation (list intersection) to get a set of docs that have all keywords. The list is then passed to a ranker to be scored, filtered and ranked. Finally the ranked docs are passed to the main goroutine through callback channel and returned to user .

In order to improve concurrency at search time, documents are sharded based on content and docid (number of shards can be specified by user) -- indexing and search requests are sent to all shards in parallel, and ranked docs are merged in the main goroutine before returning to user.

Above gives you basic idea on how Wukong works. A search system often consists of four parts, **documents crawling**, **indexing**, **search** and **rendering**. I will explain them in following sections.

## Doc crawling

The technology of doc crawling deserve a full article. Fortunately for Weibo fetching, Sina provides simple APIs to get Weibo data and there's an easy-to-use [Go language SDK](http://github.com/huichen/gobo) which allows fetching with high concurrency.

I've downloaded about one hundred thousand weibo posts and stored them in testdata/weibo_data.txt, so you do not need to do it yourself. One line of the file has following format:

    <Weibo id>||||<timestamp>||||<User id>||||<user name>||||<share counts>||||<comment counts>||||<fav counts>||||<thumbnail URL>||||<large image URL>||||<body>

The Weibos are saved in following struct. Only saved data useful for this codelab.

```Go
type Weibo struct {
    Id           uint64
    Timestamp    uint64
    UserName     string
    RepostsCount uint64
    Text         string
}
```

However, if you are interested how the data is crawled, see this script [testdata/crawl_weibo_data.go](/testdata/crawl_weibo_data.go).

## Indexing

To use Wukong engine you need to import two packages

```Go
import (
    "Github.com/huichen/wukong/engine"
    "Github.com/huichen/wukong/types"
)
```
The first one implements the engine, and the second package defines common structs. The engine has to be initialized before use, for example,

```Go
var search engine.Engine
searcher.Init(types.EngineInitOptions{
    SegmenterDictionaries: "../../data/dictionary.txt",
    StopTokenFile: "../../data/stop_tokens.txt",
    IndexerInitOptions: & types.IndexerInitOptions {
        IndexType: types.LocationsIndex,
    },
})
```

[Types.EngineInitOptions](/types/engine_init_options.go) defines the initialization engines need to set parameters, such as where the loaded word from the dictionary file, stop word lists, indexes, type, BM25 parameters, and the default rating rules (see "Search" a) and output pagination options. Please read the details of the code structure of the Notes.

What must be emphasized is that please choose carefully IndexerInitOptions.IndexType type, there are three different types of index table:

1. DocIdsIndex, provide the most basic index records the search button appears only documents docid.
2. FrequenciesIndex, in addition to recording docid, but also save the search button in the frequency of occurrence of each document, so if you need BM25 FrequenciesIndex what you need.
3. LocationsIndex, this includes not only the index on the two kinds of content, but also additional storage a keyword specific location in a document, which is used [close distance calculation] (/ docs / token_proximity.md).

These three indexes from top to bottom at the same time provide more computing power also consumes more memory, especially LocationsIndex, when the document is very long memory intensive. According to the need to balance choice.

After a good initialization can add an index, the following example will add a microblogging engine

```Go
searcher.IndexDocument(docId, types.DocumentIndexData{
    Content: weibo.Text, // Weibo struct is defined above. Content must be in UTF-8.
    Fields: WeiboScoringFields {
        Timestamp: weibo.Timestamp,
        RepostsCount: weibo.RepostsCount,
    },
})
```

DocId document must be unique, for it can be directly used microblogging microblogging ID. Wukong engine allows you to join three kinds of index data:

1. body of the document (content), will be sub-word as a keyword (tokens) added to the index.
2. Documentation of keywords (tokens). When the body is empty, it allows the user to bypass the built-in word Wukong directly input document keywords, which makes the engine outside the document segmentation possible.
3. Document Properties tab (labels), such as micro-blog author, category, etc. Tag does not appear in the text.
4. custom score field (scoring fields), which allows you to add documents of any type ** ** ** ** arbitrary data structure used for sorting. "Search" will further introduce a custom score field usage.

Special attention is ** **, keyword (tokens) and labels (labels) formed the indexer in the search key (keywords), documentation and code will be repeated three concepts, please do not be confused. Search for text in the search key is a logical query, such as a body of the document appears in the "bicycle" the key words there are "fitness" This category labels, but the "fitness" of the word does not directly appear in the text, When the query "bicycle" + "fitness" This search key combination, this article will be queried. Design label is intended to facilitate the dimension from the non-literal meaning quickly narrow scope of the query.

Engine uses the index of non-synchronous mode, that is when IndexDocument returns the index may not yet be added to the index table, which facilitate you cycle concurrently added to the index. If you need to wait before you start adding up the index operation, please call the following function

```Go
searcher.FlushIndex ()
```

## Search

Search process in two steps, the first step is to look in the index table containing the search key documents, which has been introduced in the last one before. The second step is an index of all documents to be sorted.

Sort the core of the document score. Wukong engine allows you to customize any of the scoring rules (scoring criteria). Search the microblogging example, we define scoring rules are as follows:

1. first sort by keywords close distance, for example search for "cycling", the phrase will be cut into two words, "bicycle" and "movement", there are two words next to the article should be at two key separate article in front of the word.
2. and then follow the microblogging Published roughly sort, and every three days as a team, later articles echelon top surface.
3. score was finally given microblogging BM25 * (1 + forwarding number / 10000)

Such rules need to save some of the score for each document data, such as microblogging Published, microblogging forwarding number and so on. The data is stored in the following structure body
```Go
type WeiboScoringFields struct {
    Timestamp uint64
    RepostsCount uint64
}
```
You may have noticed, this is the last one when the document is added to the index passed to the function call IndexDocument parameter type (in fact, that argument is the interface {} type, so you can pass any type of structure). The data stored in the memory sequencer waits for the call.

With these data, we can score, the code is as follows:
```Go
type WeiboScoringCriteria struct {
}

func (criteria WeiboScoringCriteria) Score (
        doc types.IndexedDocument, fields interface {}) [] float32 {
        if reflect.TypeOf (fields)! = reflect.TypeOf (WeiboScoringFields {}) {
                return [] float32 {}
        }
        wsf: = fields. (WeiboScoringFields)
        output: = make ([] float32, 3)
        if doc.TokenProximity> MaxTokenProximity {/ / Step
                output [0] = 1.0 / float32 (doc.TokenProximity)
        } Else {
                output [0] = 1.0
        }
        output [1] = float32 (wsf.Timestamp / (SecondsInADay * 3)) / / Step
        output [2] = float32 (doc.BM25 * (1 + float32 (wsf.RepostsCount) / 10000)) / / The third step
        return output
}
```
WeiboScoringCriteria actually inherited types.ScoringCriteria interface that implements Score function. This function takes two parameters:

1. Types.IndexedDocument indexer parameters passed from the data obtained, for example, word frequency, word specific location, BM25 value, close to the degrees and other information, see specific [types / index.go] (/ types / index.go) of comments.
(2) The second parameter is the type of interface {}, you can put this type of understanding into the C language void pointer, it can point to any data type. In our example, the point is WeiboScoringFields structure, and through reflection mechanism checks the correct type.

With custom scoring data and custom scoring rules, we will be able to search, and see the code below

```Go
response: = searcher.Search (types.SearchRequest {
Text: "cycling"
RankOptions: & types.RankOptions {
ScoringCriteria: & WeiboScoringCriteria {},
OutputOffset: 0,
MaxOutputs: 100,
},
})
```

Which, Text is entered search phrase (must be UTF-8 format), will be sub-word as a keyword. And the same index, Wukong engine allows to bypass the built-in word documents directly enter keywords and labels, see types.SearchRequest structure annotation. RankOptions defines the sorting options. WeiboScoringCriteria is our scoring rules defined above. In addition, you can also OutputOffset and MaxOutputs parameters control the paging output. Search results in response variable, the specific content [types / search_response.go] (/ types / search_response.go) SearchResponse defined in the file structure, such as the structure returned keywords appear in the document location, you can used to generate the document summary.

## Rendering

The final step in completing the user search is the search results to the user. The usual practice is to make a background service search engine, and then let the front end of the JSON-RPC way to call it. Front engine itself does not belong to Goku is not so much inked.

## Summary

Read here, you should use Wukong microblogging search engine have a basic understanding, I suggest you do it yourself to complete it. If you do not have patience, you can see the code has been completed, see [examples / codelab / search_server.go] (/ examples / codelab / search_server.go), total of less than 200 lines. Run this example is very simple, enter the examples / codelab directory input

    go run search_server.go

Waiting terminal in a "indexes, xxx microblogging" after the output in the browser to open [http://localhost:8080] (http://localhost:8080) to enter the search page, which implements a simplified version
http://soooweibo.com

If you want to learn more about Wukong engine, I suggest you read the code directly. Code directory structure is as follows:

    /core       core components, including the index and sorter
    /data       dictionary files and stop word file
    /docs       documentation
    /engine     engine, including the main coroutine, word coroutine, indexers, coroutines, and sorter implementation of coroutines
    /examples   examples and performance testing procedures
    /testdata   test data
    /types      commonly used structure
    /utils      commonly used functions
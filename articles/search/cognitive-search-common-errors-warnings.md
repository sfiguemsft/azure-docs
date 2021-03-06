---
title: Indexer errors and warnings 
titleSuffix: Azure Cognitive Search
description: This article provides information and solutions to common errors and warnings you might encounter during AI enrichment in Azure Cognitive Search.

manager: nitinme
author: amotley
ms.author: abmotley
ms.service: cognitive-search
ms.topic: conceptual
ms.date: 11/04/2019
---

# Troubleshooting common indexer errors and warnings in Azure Cognitive Search

This article provides information and solutions to common errors and warnings you might encounter during indexing and AI enrichment in Azure Cognitive Search.

Indexing stops when the error count exceeds ['maxFailedItems'](cognitive-search-concept-troubleshooting.md#tip-3-see-what-works-even-if-there-are-some-failures). 

If you want indexers to ignore these errors (and skip over "failed documents"), consider updating the `maxFailedItems` and `maxFailedItemsPerBatch` as described [here](https://docs.microsoft.com/rest/api/searchservice/create-indexer#general-parameters-for-all-indexers).

> [!NOTE]
> Each failed document along with its document key (when available) will show up as an error in the indexer execution status. You can utilize the [index api](https://docs.microsoft.com/rest/api/searchservice/addupdate-or-delete-documents) to manually upload the documents at a later point if you have set the indexer to tolerate failures.

The error information in this article can help you resolve errors, allowing indexing to continue.

Warnings do not stop indexing, but they do indicate conditions that could result in unexpected outcomes. Whether you take action or not depends on the data and your scenario.

Beginning with API version `2019-05-06`, item-level Indexer errors and warnings are structured to provide increased clarity around causes and next steps. They contain the following properties:

| Property | Description | Example |
| --- | --- | --- |
| key | The document ID of the document impacted by the error or warning. | https:\//coromsearch.blob.core.windows.net/jfk-1k/docid-32112954.pdf |
| name | The operation name describing where the error or warning occurred. This is generated by the following structure: [category].[subcategory].[resourceType].[resourceName] | DocumentExtraction.azureblob.myBlobContainerName Enrichment.WebApiSkill.mySkillName Projection.SearchIndex.OutputFieldMapping.myOutputFieldName Projection.SearchIndex.MergeOrUpload.myIndexName Projection.KnowledgeStore.Table.myTableName |
| message | A high-level description of the error or warning. | Could not execute skill because the Web Api request failed. |
| details | Any additional details which may be helpful to diagnose the issue, such as the WebApi response if executing a custom skill failed. | `link-cryptonyms-list - Error processing the request record : System.ArgumentNullException: Value cannot be null. Parameter name: source at System.Linq.Enumerable.All[TSource](IEnumerable`1 source, Func`2 predicate) at Microsoft.CognitiveSearch.WebApiSkills.JfkWebApiSkills.` ...rest of stack trace... |
| documentationLink | A link to relevant documentation with detailed information to debug and resolve the issue. This link will often point to one of the below sections on this page. | https://go.microsoft.com/fwlink/?linkid=2106475 |

<a name="could-not-read-document"/>

## Error: Could not read document

Indexer was unable to read the document from the data source. This can happen due to:

| Reason | Details/Example | Resolution |
| --- | --- | --- |
| inconsistent field types across different documents | Type of value has a mismatch with column type. Couldn't store `'{47.6,-122.1}'` in authors column.  Expected type is JArray. | Ensure that the type of each field is the same across different documents. For example, if the first document `'startTime'` field is a DateTime, and in the second document it's a string, this error will be hit. |
| errors from the data source's underlying service | (from Cosmos DB) `{"Errors":["Request rate is large"]}` | Check your storage instance to ensure it's healthy. You may need to adjust your scaling/partitioning. |
| transient issues | A transport-level error has occurred when receiving results from the server. (provider: TCP Provider, error: 0 - An existing connection was forcibly closed by the remote host | Occasionally there are unexpected connectivity issues. Try running the document through your indexer again later. |

<a name="could-not-extract-document-content"/>

## Error: Could not extract content or metadata from your document
Indexer with a Blob data source was unable to extract the content or metadata from the document (for example, a PDF file). This can happen due to:

| Reason | Details/Example | Resolution |
| --- | --- | --- |
| blob is over the size limit | Document is `'150441598'` bytes, which exceeds the maximum size `'134217728'` bytes for document extraction for your current service tier. | [blob indexing errors](search-howto-indexing-azure-blob-storage.md#dealing-with-errors) |
| blob has unsupported content type | Document has unsupported content type `'image/png'` | [blob indexing errors](search-howto-indexing-azure-blob-storage.md#dealing-with-errors) |
| blob is encrypted | Document could not be processed - it may be encrypted or password protected. | You can skip the blob with [blob settings](search-howto-indexing-azure-blob-storage.md#controlling-which-parts-of-the-blob-are-indexed). |
| transient issues | "Error processing blob: The request was aborted: The request was canceled." "Document timed out during processing." | Occasionally there are unexpected connectivity issues. Try running the document through your indexer again later. |

<a name="could-not-parse-document"/>

## Error: Could not parse document
Indexer read the document from the data source, but there was an issue converting the document content into the specified field mapping schema. This can happen due to:

| Reason | Details/Example | Resolution |
| --- | --- | --- |
| The document key is missing | Document key cannot be missing or empty | Ensure all documents have valid document keys |
| The document key is invalid | Document key cannot be longer than 1024 characters | Modify the document key to meet the validation requirements. |
| Could not apply field mapping to a field | Could not apply mapping function `'functionName'` to field `'fieldName'`. Array cannot be null. Parameter name: bytes | Double check the [field mappings](search-indexer-field-mappings.md) defined on the indexer, and compare with the data of the specified field of the failed document. It may be necessary to modify the field mappings or the document data. |
| Could not read field value | Could not read the value of column `'fieldName'` at index `'fieldIndex'`. A transport-level error has occurred when receiving results from the server. (provider: TCP Provider, error: 0 - An existing connection was forcibly closed by the remote host.) | These errors are typically due to unexpected connectivity issues with the data source's underlying service. Try running the document through your indexer again later. |

<a name="could-not-execute-skill"/>

## Error: Could not execute skill
Indexer was not able to run a skill in the skillset.

| Reason | Details/Example | Resolution |
| --- | --- | --- |
| Transient connectivity issues | A transient error occurred. Please try again later. | Occasionally there are unexpected connectivity issues. Try running the document through your indexer again later. |
| Potential product bug | An unexpected error occurred. | This indicates an unknown class of failure and may mean there is a product bug. Please file a [support ticket](https://ms.portal.azure.com/#create/Microsoft.Support) to get help. |
| A skill has encountered an error during execution | (From Merge Skill) One or more offset values were invalid and could not be parsed. Items were inserted at the end of the text | Use the information in the error message to fix the issue. This kind of failure will require action to resolve. |

<a name="could-not-execute-skill-because-the-web-api-request-failed"/>

## Error: Could not execute skill because the Web API request failed
Skill execution failed because the call to the Web API failed. Typically, this class of failure occurs when custom skills are used, in which case you will need to debug your custom code to resolve the issue. If instead the failure is from a built-in skill, refer to the error message for help in fixing the issue.

<a name="could-not-execute-skill-because-web-api-skill-response-is-invalid"/>

## Error: Could not execute skill because Web API skill response is invalid
Skill execution failed because the call to the Web API returned an invalid response. Typically, this class of failure occurs when custom skills are used, in which case you will need to debug your custom code to resolve the issue. If instead the failure is from a built-in skill, please file a [support ticket](https://ms.portal.azure.com/#create/Microsoft.Support) to get assistance.

<a name="skill-did-not-execute-within-the-time-limit"/>

## Error: Skill did not execute within the time limit
There are two cases under which you may encounter this error message, each of which should be treated differently. Please follow the instructions below depending on what skill returned this error for you.

### Built-in Cognitive Service skills
Many of the built-in cognitive skills, such as language detection, entity recognition, or OCR, are backed by a Cognitive Service API endpoint. Sometimes there are transient issues with these endpoints and a request will time out. For transient issues, there is no remedy except to wait and try again. As a mitigation, consider setting your indexer to [run on a schedule](search-howto-schedule-indexers.md). Scheduled indexing picks up where it left off. Assuming transient issues are resolved, indexing and cognitive skill processing should be able to continue on the next scheduled run.

If you continue to see this error on the same document for a built-in cognitive skill, please file a [support ticket](https://ms.portal.azure.com/#create/Microsoft.Support) to get assistance, as this is not expected.

### Custom skills
If you encounter a timeout error with a custom skill you have created, there are a couple of things you can try. First, review your custom skill and ensure that it is not getting stuck in an infinite loop and that it is returning a result consistently. Once you have confirmed that is the case, determine what the execution time of your skill is. If you didn't explicitly set a `timeout` value on your custom skill definition, then the default `timeout` is 30 seconds. If 30 seconds is not long enough for your skill to execute, you may specify a higher `timeout` value on your custom skill definition. Here is an example of a custom skill definition where the timeout is set to 90 seconds:

```json
  {
        "@odata.type": "#Microsoft.Skills.Custom.WebApiSkill",
        "uri": "<your custom skill uri>",
        "batchSize": 1,
        "timeout": "PT90S",
        "context": "/document",
        "inputs": [
          {
            "name": "input",
            "source": "/document/content"
          }
        ],
        "outputs": [
          {
            "name": "output",
            "targetName": "output"
          }
        ]
      }
```

The maximum value that you can set for the `timeout` parameter is 230 seconds.  If your custom skill is unable to execute consistently within 230 seconds, you may consider reducing the `batchSize` of your custom skill so that it will have fewer documents to process within a single execution.  If you have already set your `batchSize` to 1, you will need to rewrite the skill to be able to execute in under 230 seconds or otherwise split it into multiple custom skills so that the execution time for any single custom skill is a maximum of 230 seconds. Review the [custom skill documentation](cognitive-search-custom-skill-web-api.md) for more information.

<a name="could-not-mergeorupload--delete-document-to-the-search-index"/>

## Error: Could not '`MergeOrUpload`' | '`Delete`' document to the search index

The document was read and processed, but the indexer could not add it to the search index. This can happen due to:

| Reason | Details/Example | Resolution |
| --- | --- | --- |
| A field contains a term that is too large | A term in your document is larger than the [32 KB limit](search-limits-quotas-capacity.md#api-request-limits) | You can avoid this restriction by ensuring the field is not configured as filterable, facetable, or sortable.
| Document is too large to be indexed | A document is larger than the [maximum api request size](search-limits-quotas-capacity.md#api-request-limits) | [How to index large data sets](search-howto-large-index.md)
| Document contains too many objects in collection | A collection in your document exceeds the [maximum elements across all complex collections limit](search-limits-quotas-capacity.md#index-limits) | We recommend reducing the size of the complex collection in the document to below the limit and avoid high storage utilization.
| Trouble connecting to the target index (that persists after retries) because the service is under other load, such as querying or indexing. | Failed to establish connection to update index. Search service is under heavy load. | [Scale up your search service](search-capacity-planning.md)
| Search service is being patched for service update, or is in the middle of a topology reconfiguration. | Failed to establish connection to update index. Search service is currently down/Search service is undergoing a transition. | Configure service with at least 3 replicas for 99.9% availability per [SLA documentation](https://azure.microsoft.com/support/legal/sla/search/v1_0/)
| Failure in the underlying compute/networking resource (rare) | Failed to establish connection to update index. An unknown failure occurred. | Configure indexers to [run on a schedule](search-howto-schedule-indexers.md) to pick up from a failed state.
| An indexing request made to the target index was not acknowledged within a timeout period due to network issues. | Could not establish connection to the search index in a timely manner. | Configure indexers to [run on a schedule](search-howto-schedule-indexers.md) to pick up from a failed state. Additionally, try lowering the indexer [batch size](https://docs.microsoft.com/rest/api/searchservice/create-indexer#parameters) if this error condition persists.

<a name="could-not-index-document-because-the-indexer-data-to-index-was-invalid"/>

## Error: Could not index document because the indexer data to index was invalid

The document was read and processed, but due to a mismatch in the configuration of the index fields and the nature of the data extracted by the indexer, it could not be added to the search index. This can happen due to:

| Reason | Details/Example
| --- | ---
| Data type of the field(s) extracted by the indexer is incompatible with the data model of the corresponding target index field. | The data field '_data_' in the document with key '888' has an invalid value 'of type 'Edm.String''. The expected type was 'Collection(Edm.String)'. |
| Failed to extract any JSON entity from a string value. | Could not parse value 'of type 'Edm.String'' of field '_data_' as a JSON object. Error:'After parsing a value an unexpected character was encountered: ''. Path '_path_', line 1, position 3162.' |
| Failed to extract a collection of JSON entities from a string value.  | Could not parse value 'of type 'Edm.String'' of field '_data_' as a JSON array. Error:'After parsing a value an unexpected character was encountered: ''. Path '[0]', line 1, position 27.' |
| An unknown type was discovered in the source document. | Unknown type '_unknown_' cannot be indexed |
| An incompatible notation for geography points was used in the source document. | WKT POINT string literals are not supported. Please use GeoJson point literals instead |

In all these cases, refer to [Supported Data types](https://docs.microsoft.com/rest/api/searchservice/supported-data-types) and [Data type map for indexers](https://docs.microsoft.com/rest/api/searchservice/data-type-map-for-indexers-in-azure-search) to make sure that you build the index schema correctly and have set up appropriate [indexer field mappings](search-indexer-field-mappings.md). The error message will include details that can help track down the source of the mismatch.

<a name="could-not-process-document-within-indexer-max-run-time"/>

## Error: Could not process document within indexer max run time

This error occurs when the indexer is unable to finish processing a single document from the data source within the allowed execution time. [Maximum running time](search-limits-quotas-capacity.md#indexer-limits) is shorter when skillsets are used. When this error occurs, if you have maxFailedItems set to a value other than 0, the indexer bypasses the document on future runs so that indexing can progress. If you cannot afford to skip any document, or if you are seeing this error consistently, consider breaking documents into smaller documents so that partial progress can be made within a single indexer execution.

<a name="could-not-execute-skill-because-a-skill-input-was-invalid"/>

## Warning: Could not execute skill because a skill input was invalid
Indexer was not able to run a skill in the skillset because an input to the skill was missing, the wrong type, or otherwise invalid.

Cognitive skills have required inputs and optional inputs. For example the [Key phrase extraction skill](cognitive-search-skill-keyphrases.md) has two required inputs `text`, `languageCode`, and no optional inputs. If any required inputs are invalid, the skill gets skipped and generates a warning. Skipped skills do not generate any outputs, so if other skills use outputs of the skipped skill they may generate additional warnings.

If you want to provide a default value in case of missing input, you can use the [Conditional skill](cognitive-search-skill-conditional.md) to generate a default value and then use the output of the [Conditional skill](cognitive-search-skill-conditional.md) as the skill input.


```json
{
    "@odata.type": "#Microsoft.Skills.Util.ConditionalSkill",
    "context": "/document",
    "inputs": [
        { "name": "condition", "source": "= $(/document/language) == null" },
        { "name": "whenTrue", "source": "= 'en'" },
        { "name": "whenFalse", "source": "= $(/document/language)" }
    ],
    "outputs": [ { "name": "output", "targetName": "languageWithDefault" } ]
}
```

| Reason | Details/Example | Resolution |
| --- | --- | --- |
| Skill input is the wrong type | "Required skill input was not of the expected type `String`. Name: `text`, Source: `/document/merged_content`."  "Required skill input was not of the expected format. Name: `text`, Source: `/document/merged_content`."  "Cannot iterate over non-array `/document/normalized_images/0/imageCelebrities/0/detail/celebrities`."  "Unable to select `0` in non-array `/document/normalized_images/0/imageCelebrities/0/detail/celebrities`" | Certain skills expect inputs of particular types, for example [Sentiment skill](cognitive-search-skill-sentiment.md) expects `text` to be a string. If the input specifies a non-string value, then the skill doesn't execute and generates no outputs. Ensure your data set has input values uniform in type, or use a [Custom Web API skill](cognitive-search-custom-skill-web-api.md) to preprocess the input. If you're iterating the skill over an array, check the skill context and input have `*` in the correct positions. Usually both the context and input source should end with `*` for arrays. |
| Skill input is missing | "Required skill input is missing. Name: `text`, Source: `/document/merged_content`"  "Missing value `/document/normalized_images/0/imageTags`."  "Unable to select `0` in array `/document/pages` of length `0`." | If all your documents get this warning, most likely there is a typo in the input paths and you should double check property name casing, extra or missing `*` in the path, and make sure that the documents from the data source provide the required inputs. |
| Skill language code input is invalid | Skill input `languageCode` has the following language codes `X,Y,Z`, at least one of which is invalid. | See more details [below](cognitive-search-common-errors-warnings.md#skill-input-languagecode-has-the-following-language-codes-xyz-at-least-one-of-which-is-invalid) |

<a name="skill-input-languagecode-has-the-following-language-codes-xyz-at-least-one-of-which-is-invalid"/>

## Warning:  Skill input 'languageCode' has the following language codes 'X,Y,Z', at least one of which is invalid.
One or more of the values passed into the optional `languageCode` input of a downstream skill is not supported. This can occur if you are passing the output of the [LanguageDetectionSkill](cognitive-search-skill-language-detection.md) to subsequent skills, and the output consists of more languages than are supported in those downstream skills.

If you know that your data set is all in one language, you should remove the [LanguageDetectionSkill](cognitive-search-skill-language-detection.md) and the `languageCode` skill input and use the `defaultLanguageCode` skill parameter for that skill instead, assuming the language is supported for that skill.

If you know that your data set contains multiple languages and thus you need the [LanguageDetectionSkill](cognitive-search-skill-language-detection.md) and `languageCode` input, consider adding a [ConditionalSkill](cognitive-search-skill-conditional.md) to filter out the text with languages that are not supported before passing in the text to the downstream skill.  Here is an example of what this might look like for the EntityRecognitionSkill:

```json
{
    "@odata.type": "#Microsoft.Skills.Util.ConditionalSkill",
    "context": "/document",
    "inputs": [
        { "name": "condition", "source": "= $(/document/language) == 'de' || $(/document/language) == 'en' || $(/document/language) == 'es' || $(/document/language) == 'fr' || $(/document/language) == 'it'" },
        { "name": "whenTrue", "source": "/document/content" },
        { "name": "whenFalse", "source": "= null" }
    ],
    "outputs": [ { "name": "output", "targetName": "supportedByEntityRecognitionSkill" } ]
}
```

Here are some references for the currently supported languages for each of the skills that may produce this error message:
* [Text Analytics Supported Languages](https://docs.microsoft.com/azure/cognitive-services/text-analytics/text-analytics-supported-languages) (for the [KeyPhraseExtractionSkill](cognitive-search-skill-keyphrases.md), [EntityRecognitionSkill](cognitive-search-skill-entity-recognition.md), and [SentimentSkill](cognitive-search-skill-sentiment.md))
* [Translator Supported Languages](https://docs.microsoft.com/azure/cognitive-services/translator/language-support) (for the [Text TranslationSkill](cognitive-search-skill-text-translation.md))
* [Text SplitSkill](cognitive-search-skill-textsplit.md) Supported Languages: `da, de, en, es, fi, fr, it, ko, pt`

<a name="skill-input-was-truncated"/>

## Warning: Skill input was truncated
Cognitive skills have limits to the length of text that can be analyzed at once. If the text input of these skills are over that limit, we will truncate the text to meet the limit, and then perform the enrichment on that truncated text. This means that the skill is executed, but not over all of your data.

In the example LanguageDetectionSkill below, the `'text'` input field may trigger this warning if it is over the character limit. You can find the skill input limits in the [skills documentation](cognitive-search-predefined-skills.md).

```json
 {
    "@odata.type": "#Microsoft.Skills.Text.LanguageDetectionSkill",
    "inputs": [
      {
        "name": "text",
        "source": "/document/text"
      }
    ],
    "outputs": [...]
  }
```

If you want to ensure that all text is analyzed, consider using the [Split skill](cognitive-search-skill-textsplit.md).

<a name="web-api-skill-response-contains-warnings"/>

## Warning: Web API skill response contains warnings
Indexer was able to run a skill in the skillset, but the response from the Web API request indicated there were warnings during execution. Review the warnings to understand how your data is impacted and whether or not action is required.

<a name="the-current-indexer-configuration-does-not-support-incremental-progress"/>

## Warning: The current indexer configuration does not support incremental progress

This warning only occurs for Cosmos DB data sources.

Incremental progress during indexing ensures that if indexer execution is interrupted by transient failures or execution time limit, the indexer can pick up where it left off next time it runs, instead of having to re-index the entire collection from scratch. This is especially important when indexing large collections.

The ability to resume an unfinished indexing job is predicated on having documents ordered by the `_ts` column. The indexer uses the timestamp to determine which document to pick up next. If the `_ts` column is missing or if the indexer can't determine if a custom query is ordered by it, the indexer starts at beginning and you'll see this warning.

It is possible to override this behavior, enabling incremental progress and suppressing this warning by using the `assumeOrderByHighWatermarkColumn` configuration property.

For more information, see [Incremental progress and custom queries](search-howto-index-cosmosdb.md#IncrementalProgress).

<a name="some-data-was-lost-during projection-row-x-in-table-y-has-string-property-z-which-was-too-long"/>

## Warning: Some data was lost during projection. Row 'X' in table 'Y' has string property 'Z' which was too long.

The [Table Storage service](https://azure.microsoft.com/services/storage/tables) has limits on how large [entity properties](https://docs.microsoft.com/rest/api/storageservices/understanding-the-table-service-data-model#property-types) can be. Strings can have 32,000 characters or less. If a row with a string property longer than 32,000 characters is being projected, only the first 32,000 characters are preserved. To work around this issue, avoid projecting rows with string properties longer than 32,000 characters.

<a name="truncated-extracted-text-to-x-characters"/>

## Warning: Truncated extracted text to X characters
Indexers limit how much text can be extracted from any one document. This limit depends on the pricing tier: 32,000 characters for Free tier, 64,000 for Basic, and 4 million for Standard, Standard S2 and Standard S3 tiers. Text that was truncated will not be indexed. To avoid this warning, try breaking apart documents with large amounts of text into multiple, smaller documents. 

For more information, see [Indexer limits](search-limits-quotas-capacity.md#indexer-limits).

<a name="could-not-map-output-field-x-to-search-index"/>

## Warning: Could not map output field 'X' to search index
Output field mappings that reference non-existent/null data will produce warnings for each document and result in an empty index field. To workaround this issue, double-check your output field-mapping source paths for possible typos, or set a default value using the [Conditional skill](cognitive-search-skill-conditional.md#sample-skill-definition-2-set-a-default-value-for-a-value-that-doesnt-exist).

<a name="the-data-change-detection-policy-is-configured-to-use-key-column-x"/>

## Warning: The data change detection policy is configured to use key column 'X'
[Data change detection policies](https://docs.microsoft.com/rest/api/searchservice/create-data-source#data-change-detection-policies) have specific requirements for the columns they use to detect change. One of these requirements is that this column is updated every time the source item is changed. Another requirement is that the new value for this column is greater than the previous value. Key columns don't fulfill this requirement because they don't change on every update. To work around this issue, select a different column for the change detection policy.

<a name="document-text-appears-to-be-utf-16-encoded-but-is-missing-a-byte-order-mark"/>

## Warning: Document text appears to be UTF-16 encoded, but is missing a byte order mark

The [indexer parsing modes](https://docs.microsoft.com/rest/api/searchservice/create-indexer#blob-configuration-parameters) need to know how text is encoded before parsing it. The two most common ways of encoding text are UTF-16 and UTF-8. UTF-8 is a variable-length encoding where each character is between 1 byte and 4 bytes long. UTF-16 is a fixed-length encoding where each character is 2 bytes long. UTF-16 has two different variants, "big endian" and "little endian". Text encoding is determined by a "byte order mark", a series of bytes before the text.

| Encoding | Byte Order Mark |
| --- | --- |
| UTF-16 Big Endian | 0xFE 0xFF |
| UTF-16 Little Endian | 0xFF 0xFE |
| UTF-8 | 0xEF 0xBB 0xBF |

If no byte order mark is present, the text is assumed to be encoded as UTF-8.

To work around this warning, determine what the text encoding for this blob is and add the appropriate byte order mark.

<a name="cosmos-db-collection-has-a-lazy-indexing-policy"/>

## Warning: Cosmos DB collection 'X' has a Lazy indexing policy. Some data may be lost

Collections with [Lazy](https://docs.microsoft.com/azure/cosmos-db/index-policy#indexing-mode) indexing policies can't be queried consistently, resulting in your indexer missing data. To work around this warning, change your indexing policy to Consistent.

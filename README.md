# custom_read_api
---
Description:
- Form Recognizer API evolves quickly to increase transcription accuracy, add new supported languages and/or simply onboard new features. custom_read_api is an Azure Cognitive Search skill to integrate Azure Form Recognizer Read API within an Azure Cognitive Search skillset. By default the in built OCR skill uses the latest GA version of Read API (which does not included latest and greatest developments) so this skill enables fresh previews to be leveraged.

Languages:
- ![python](https://img.shields.io/badge/language-python-orange)

Products:
- Azure Cognitive Search
- Azure Cognitive Services (Form Recognizer Read API https://docs.microsoft.com/en-us/azure/applied-ai-services/form-recognizer/concept-read)
- Azure Functions
---

# Steps    

1. Create or reuse a Form Recognizer resource. Creation can be done from the Azure portal or programatically
2. Create a Python Function in Azure, for example this is a good [starting point](https://docs.microsoft.com/azure/azure-functions/create-first-function-vs-code-python)
3. Clone this repository
4. Open the folder in VS Code and deploy the function, find here a [tutorial](https://docs.microsoft.com/azure/search/cognitive-search-custom-skill-python)
5. Fill your Functions appsettings with the details from your deployment ('FR_ENDPOINT', 'FR_ENDPOINT_KEY' with the info you got in Azure Portal) (ie FR_ENDPOINT can be something like https://westeurope.api.cognitive.microsoft.com/formrecognizer/documentModels/prebuilt-read:analyze?api-version=2022-06-30-preview) 
6. Add a field in your index where you will dump the enriched entities, more info [here](#sample-index-field-definition)
7. Add the skill to your skillset as [described below](#sample-skillset-integration)
8. Add the output field mapping in your indexer as [seen in the sample](#sample-indexer-output-field-mapping)
9. Run the indexer 

## Sample Input:

You can find a sample input for the skill [here](../main/latestocr/sample.dat)

```json
{
    "values": [
      {
        "recordId": "0",
        "data":{
            "Url": "yourbase64encoded", 
            "SasToken": "?sp=rac_restofyoursastokenhere"
            }
      }
     
    ]
}
```

## Sample Output:

```json
{
  "values": [
    {
      "recordId": "0",
      "data": {
        "text": {
            "your merged text after OCR transcription provides line by line within each of the pages"        
        }
      }
    }
  ]
}
```

## Sample Skillset Integration

In order to use this skill in a cognitive search pipeline, you'll need to add a skill definition to your skillset.
Here's a sample skill definition for this example (inputs and outputs should be updated to reflect your particular scenario and skillset environment):

```json
    {
      "@odata.type": "#Microsoft.Skills.Text.KeyPhraseExtractionSkill",
      "name": "#1",
      "description": "Takes the merged text and processes it to filter only the key phrases",
      "context": "/document/merged_content",
      "defaultLanguageCode": "en",
      "maxKeyPhraseCount": null,
      "modelVersion": null,
      "inputs": [
        {
          "name": "text",
          "source": "/document/merged_content"
        }
      ],
      "outputs": [
        {
          "name": "keyPhrases",
          "targetName": "keyphrases"
        }
      ]
    },
    {
      "@odata.type": "#Microsoft.Skills.Custom.WebApiSkill",
      "name": "#2",
      "description": null,
      "context": "/document",
      "uri": "https://yourappfuncname.azurewebsites.net/api/function_name?code=unique_code_for_auth_here",
      "httpMethod": "POST",
      "timeout": "PT2M60S",
      "batchSize": 2,
      "degreeOfParallelism": null,
      "inputs": [
        {
          "name": "Url",
          "source": "/document/metadata_storage_path"
        },
        {
          "name": "SasToken",
          "source": "/document/metadata_storage_sas_token"
        }
      ],
      "outputs": [
        {
          "name": "text",
          "targetName": "merged_content"
        }
      ],
      "httpHeaders": {}
    }
```

## Sample Index Field Definition

This skill internally calls a given endpoint for FR, navigates the output, trims the response to only fetch the text (leaving aside layout and confidence levels) and concats the text into one single string.

```json
{
      "name": "keyphrases",
      "type": "Collection(Edm.String)",
      "facetable": false,
      "filterable": false,
      "retrievable": true,
      "searchable": true,
      "analyzer": "standard.lucene",
      "indexAnalyzer": null,
      "searchAnalyzer": null,
      "synonymMaps": [],
      "fields": []
    },
    {
      "name": "merged_content",
      "type": "Edm.String",
      "facetable": false,
      "filterable": false,
      "key": false,
      "retrievable": true,
      "searchable": true,
      "sortable": false,
      "analyzer": "standard.lucene",
      "indexAnalyzer": null,
      "searchAnalyzer": null,
      "synonymMaps": [],
      "fields": []
    },
```

## Sample Indexer Output Field Mapping

The output enrichment of your skill can be directly mapped to one of your fields described above. By default it will be automatically mapped if there is a field named as the inner AI enriched targets (as in this case for merged_content and keyphrases, so no need to map it manually). This can be done with the indexer setting:
```
  "outputFieldMappings": []
```

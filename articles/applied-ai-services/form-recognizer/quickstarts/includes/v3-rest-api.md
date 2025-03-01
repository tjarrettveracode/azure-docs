---
title: "Quickstart: Form Recognizer REST API v3.0 | v3.0"
titleSuffix: Azure Applied AI Services
description: Form and document processing, data extraction, and analysis using Form Recognizer REST API v3.0
author: laujan
manager: nitinme
ms.service: applied-ai-services
ms.subservice: forms-recognizer
ms.topic: include
ms.date: 08/22/2022
ms.author: lajanuar
---

| [Form Recognizer REST API](https://westus.dev.cognitive.microsoft.com/docs/services/form-recognizer-api-2022-08-31/operations/AnalyzeDocument) | [Azure SDKS](https://azure.github.io/azure-sdk/releases/latest/index.html) | [Supported SDKs](../../sdk-overview.md)

In this quickstart you'll, use the Form Recognizer REST API to analyze and extract data and values from forms and documents:

## Prerequisites

* Azure subscription - [Create one for free](https://azure.microsoft.com/free/cognitive-services)

* curl command line tool installed.

  * [Windows](https://curl.haxx.se/windows/)
  * [Mac or Linux](https://learn2torials.com/thread/how-to-install-curl-on-mac-or-linux-(ubuntu)-or-windows)

* **PowerShell version 7.*+** (or a similar command-line application.):
  * [Windows](/powershell/scripting/install/installing-powershell-on-windows?view=powershell-7.2&preserve-view=true)
  * [macOS](/powershell/scripting/install/installing-powershell-on-macos?view=powershell-7.2&preserve-view=true)
  * [Linux](/powershell/scripting/install/installing-powershell-on-linux?view=powershell-7.2&preserve-view=true)

* To check your PowerShell version, type the following:
  * Windows: `Get-Host | Select-Object Version`
  * macOS or Linux: `$PSVersionTable`

* A Form Recognizer (single-service) or Cognitive Services (multi-service) resource. Once you have your Azure subscription, create a [single-service](https://portal.azure.com/#create/Microsoft.CognitiveServicesFormRecognizer) or [multi-service](https://portal.azure.com/#create/Microsoft.CognitiveServicesAllInOne) Form Recognizer resource, in the Azure portal, to get your key and endpoint. You can use the free pricing tier (`F0`) to try the service, and upgrade later to a paid tier for production.

> [!TIP]
> Create a Cognitive Services resource if you plan to access multiple cognitive services under a single endpoint/key. For Form Recognizer access only, create a Form Recognizer resource. Please note that you'll  need a single-service resource if you intend to use [Azure Active Directory authentication](../../../../active-directory/authentication/overview-authentication.md).

* After your resource deploys, select **Go to resource**. You need the key and endpoint from the resource you create to connect your application to the Form Recognizer API. You'll paste your key and endpoint into the code below later in the quickstart:

  :::image type="content" source="../../media/containers/keys-and-endpoint.png" alt-text="Screenshot: keys and endpoint location in the Azure portal.":::

## Analyze documents and get results

 A POST request is used to analyze documents with a prebuilt or custom model. A GET request is used to retrieve the result of a document analysis call. The `modelId` is used with POST and `resultId` with GET operations.

### Analyze document (POST Request)

Before you run the cURL command, make the following changes:

1. Replace `{endpoint}` with the endpoint value from your Azure portal Form Recognizer instance.

1. Replace `{key}` with the key value from your Azure portal Form Recognizer instance.

1. Using the table below as a reference, replace `{modelID}` and `{your-document-url}` with your desired values.

1. You'll need a document file at a URL. For this quickstart, you can use the sample forms provided in the table below for each feature:

    **Sample documents**

    | **Feature**   | **{modelID}**   | **{your-document-url}** |
    | --- | --- |--|
    | **General Document** | prebuilt-document | `https://raw.githubusercontent.com/Azure-Samples/cognitive-services-REST-api-samples/master/curl/form-recognizer/sample-layout.pdf` |
    | **Read** | prebuilt-read | `https://raw.githubusercontent.com/Azure-Samples/cognitive-services-REST-api-samples/master/curl/form-recognizer/rest-api/read.png` |
    | **Layout** | prebuilt-layout | `https://raw.githubusercontent.com/Azure-Samples/cognitive-services-REST-api-samples/master/curl/form-recognizer/rest-api/layout.png` |
    | **W-2**  | prebuilt-tax.us.w2 | `https://raw.githubusercontent.com/Azure-Samples/cognitive-services-REST-api-samples/master/curl/form-recognizer/rest-api/w2.png` |
    | **Invoices**  | prebuilt-invoice | `https://github.com/Azure-Samples/cognitive-services-REST-api-samples/raw/master/curl/form-recognizer/rest-api/invoice.pdf` |
    | **Receipts**  | prebuilt-receipt | `https://raw.githubusercontent.com/Azure-Samples/cognitive-services-REST-api-samples/master/curl/form-recognizer/rest-api/receipt.png` |
    | **ID Documents**  | prebuilt-idDocument | `https://raw.githubusercontent.com/Azure-Samples/cognitive-services-REST-api-samples/master/curl/form-recognizer/rest-api/identity_documents.png` |
    | **Business Cards**  | prebuilt-businessCard | `https://raw.githubusercontent.com/Azure-Samples/cognitive-services-REST-api-samples/de5e0d8982ab754823c54de47a47e8e499351523/curl/form-recognizer/rest-api/business_card.jpg` |

> [!IMPORTANT]
> Remember to remove the key from your code when you're done, and never post it publicly. For production, use a secure way of storing and accessing your credentials like [Azure Key Vault](../../../../key-vault/general/overview.md). For more information, *see* Cognitive Services [security](../../../../cognitive-services/security-features.md).

#### POST request

```bash
curl -v -i POST "{endpoint}/formrecognizer/documentModels/{modelID}:analyze?api-version=2022-08-31" -H "Content-Type: application/json" -H "Ocp-Apim-Subscription-Key: {key}" --data-ascii "{'urlSource': '{your-document-url}'}"
```

#### POST response

You'll receive a `202 (Success)` response that includes an **Operation-location** header. The value of this header contains a `resultID` that can be queried to get the status of the asynchronous operation:

:::image type="content" source="../../media/quickstarts/operation-location-result-id.png" alt-text="{alt-text}":::

### Get analyze results (GET Request)

After you've called the [**Analyze document**](https://westus.dev.cognitive.microsoft.com/docs/services/form-recognizer-api-2022-08-31/operations/AnalyzeDocument) API, call the [**Get analyze result**](https://westus.dev.cognitive.microsoft.com/docs/services/form-recognizer-api-2022-08-31/operations/GetAnalyzeDocumentResult) API to get the status of the operation and the extracted data. Before you run the command, make these changes:

1. Replace `{POST response}` Operation-location header from the [POST response](#post-response).

1. Replace `{key}` with the key value from your Form Recognizer instance in the Azure portal.

<!-- markdownlint-disable MD024 -->

#### GET request

```bash
curl -v -X GET "{POST response}" -H "Ocp-Apim-Subscription-Key: {key}"
```

#### Examine the response

You'll receive a `200 (Success)` response with JSON output. The first field, `"status"`, indicates the status of the operation. If the operation isn't complete, the value of `"status"` will be `"running"` or `"notStarted"`, and you should call the API again, either manually or through a script. We recommend an interval of one second or more between calls.

#### Sample response for prebuilt-invoice

```json
{
    "status": "succeeded",
    "createdDateTime": "2022-03-25T19:31:37Z",
    "lastUpdatedDateTime": "2022-03-25T19:31:43Z",
    "analyzeResult": {
        "apiVersion": "2022-08-31",
        "modelId": "prebuilt-invoice",
        "stringIndexType": "textElements"...
    ..."pages": [
            {
                "pageNumber": 1,
                "angle": 0,
                "width": 8.5,
                "height": 11,
                "unit": "inch",
                "words": [
                    {
                        "content": "CONTOSO",
                        "boundingBox": [
                            0.5911,
                            0.6857,
                            1.7451,
                            0.6857,
                            1.7451,
                            0.8664,
                            0.5911,
                            0.8664
                        ],
                        "confidence": 1,
                        "span": {
                            "offset": 0,
                            "length": 7
                        }
                    },
}
```

#### Supported document fields

The prebuilt models extract pre-defined sets of document fields. See [Model data extraction](../../concept-model-overview.md#model-data-extraction) for extracted field names, types, descriptions, and examples.

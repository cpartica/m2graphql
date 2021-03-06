# Magento 2 GraphQL API [![Get on Slack](https://img.shields.io/badge/slack-join-orange.svg)](https://www.surveymonkey.com/r/27NGBTB)

Magento 2 GraphQL API is a fully-featured **GraphQL** [(watch video)](https://www.youtube.com/watch?v=wPPFhcqGcvk) server on top of the [Magento 2 Service Contracts](http://devdocs.magento.com/guides/v2.2/extension-dev-guide/service-contracts/design-patterns.html). It allows to easily consume Magento APIs through the GraphQL clients, such as [Apollo](http://dev.apollodata.com/). Also, it allows creating GraphQL APIs in 3-d party Magento extensions, in order to expose custom extension data to the GraphQL clients. The primary use-case for such APIs is javascript clients which implement re-usable UI building blocks (Web Components) and use Magento in a headless way.

To get started with the M2 GraphQL go to Magento 2 Graph<i>i</i>QL demo page [**Magento 2 Graph<i>i</i>QL Playground**](http://m2graphql.com/graphiql/) which works on top of the [store withs the sample data](http://m2graphql.com/).

Magento 2 GraphQL server endpoint can be used by any GraphQL client. It:

1. Exposes Magento 2 Web REST APIs with the GraphQL interface
2. Installed as an extension on Magento 2 and does not require core customizations
3. Uses webonyx/graphql-php library which implements all server-side features of the GraphQL language
4. Allows traffic optimization for the client applications transferring just the data needed in particular use-case. Allows combining multiple API requests in a single request.
5. Supports Magento 2 API authentication model and exposes just the data client authorized to see
6. Supports flexible deprecation model informing clients on usage of deprecated fields

### Purpose and Intent of this Repo

Investigate GraphQL API and come up with the best design for the M2 GraphQL schema. Use **playground**, experiment and share what you liked! Feedback and ides are welcome. Please join #pwa slack channel for the most efficient discussions.

### Related Work

[Alan Kent](https://github.com/alankent) is doing [exploration project](https://github.com/alankent/module-graphql) with slightly different approach: instead of using current Magento APIs as a base for GraphQL schema, the schema for GraphQL is deisgned from the scratch, getting it directly to the desired end state of the GraphQL API, and Magento API will be changed (or re-implemented) accordingly. At the end, we will take the best of both approaches. Current repository allows to play with all the Magento API data immediatedly, while Alan's is focused on how to make the requests most convenient.

Read more in [Dev Blog](https://community.magento.com/t5/Magento-DevBlog/Why-GraphQL-Is-Interesting-for-Magento-2/ba-p/71539)

## Installation 

#### 1. Add repository to the composer.json
```
"repositories": [
      {
        "type": "vcs",
        "url": "https://github.com/vrann/m2graphql"
      }
    ],
```

Set minimum-stability to dev:
```json
"minimum-stability": "dev"
```

#### 2. Download and Install Package

- `composer require vrann/m2graphql:dev-master`
- `bin/magento module:enable Magento_GraphQL`
- `bin/magento setup:upgrade`

#### 3. Verify that it works

- in the browser, go to the `http://magento-base-url/graphql` and you expected to see properly formed JSON document

## Usage

The easiest way to try GraphQL API is to install Graph<i>i</i>QL user interface either as [app](https://github.com/skevy/graphiql-app) or [Chrome ChromeiQL extension](https://chrome.google.com/webstore/detail/chromeiql/fkkiamalmpiidkljmicmjfbieiclmeij) to the browser. The [Appolo Client DevTools](https://github.com/apollographql/apollo-client-devtools) allows to install developer tools in the Chrome console.

Production application which uses GraphQL can be written on [any supported language](http://graphql.org/code/). In this document all, examples will be using javascript and the Apollo client.

### Writing JS Client

Following is an excerpt of Apollo documentation:

```js
import ApolloClient, { createNetworkInterface } from 'apollo-client';
import gql from 'graphql-tag';

const client = new ApolloClient({
  networkInterface: createNetworkInterface({
    uri: 'https://m2instance.dev/graphql',
  }),
});
```

Replace `https://m2instance.dev` with the base path of the Magento instance here package is installed.

To execute a query with your client you may now call the `client.query` method like this:

```js
client.query({
  query: gql`
    query MagentoCatalogProduct(sku: "CannondaleCaad1032014"){
    id, name, price, media_gallery_entries {
      file
    }, custom_attributes {
      attribute_code,
      value
    }
  }
  `,
})
  .then(data => console.log(data))
  .catch(error => console.error(error));
```

## Magento APIs

If you worked with the Magento Service layer, you should already be familiar with the concept of authorization, Service Interfaces, Data Interfaces, REST routing. Otherwise, please read following materials:
1. [DevDocs on API authentication](http://devdocs.magento.com/guides/v2.2/get-started/authentication/gs-authentication.html)
2. [DevDocs on Service Contracts](http://devdocs.magento.com/guides/v2.2/extension-dev-guide/service-contracts/design-patterns.html)
3. [Fooman on Extension Attributes](https://store.fooman.co.nz/blog/an-introduction-to-extension-attributes.html)

M2GraphQL extension works on top of the Magento Webapi framework and re-uses the same infrastructure, that's why most of the concepts will work in the same way.

### Mapping of Service Contracts to GraphQL Schema

At the same time, GraphQL schema design conventions have certain assumptions which do not directly correspond to the Service contracts. One example -- fields of the Query root object are expected to be Objects, and they can accept filters as parameters of the query to the data retrieval layer. While root objects of the Service contracts are operations, which can accept the input Object of operation. In order to make the transition of the Service Contracts to GraphQL paradigm, following rules were used:

1. All GET requests are exposed as fields of the Query object
2. For particular GET request, the type of the corresponding field is constructed using the name of the result object (declared in @return annotation) of the related service interface method.
3. The parameters of the field are taken from the parameters of the service interface method corresponding to GET request
4. Input and output parameters are constructed from the input object replacing \\ with _
5. Input parameters has _Input suffix

#### Example of the Query root object:

*webapi.xml*
```xml
 <route url="/V1/products/:sku" method="GET">
        <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="get"/>
        <resources>
            <resource ref="Magento_Catalog::products" />
        </resources>
    </route>
```

*RepositoryInterface*
```php
/**
     * Get info about product by product SKU
     *
     * @param string $sku
     * @param bool $editMode
     * @param int|null $storeId
     * @param bool $forceReload
     * @return \Magento\Catalog\Api\Data\ProductInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function get($sku, $editMode = false, $storeId = null, $forceReload = false);
```

*Resulting GraphQL schema*
```
MagentoCatalogProduct(
    sku: String
    editMode: Boolean
    storeId: Int
    forceReload: Boolean
): Magento_Catalog_Api_Data_ProductInterface
```

#### Example of the SearchCriteria:
```
MagentoCmsBlockSearchResults(
      searchCriteria: Magento_Framework_Api_SearchCriteriaInterface_Input
): Magento_Cms_Api_Data_BlockSearchResultsInterface
```

### Authorization

This is Work In Progress

### Requesting Object

This is an example of simple request to retrieve 3 fields from a Store object:

```
MagentoStoreStores{id, name, code}
```

### Requesting Objects With Parameters

This is an example of filter parameters passed to the root object:

```
MagentoCatalogProduct(sku: "CannondaleCaad1032014"){
    id, name, price, media_gallery_entries {
      file
    }, custom_attributes {
      attribute_code,
      value
    }
  }
```

### SearchCriteria Request example

Example of the getList query with the SearchCriteria:
```
MagentoCmsBlockSearchResults(searchCriteria: {
    sort_orders: {
    	field: "block_id",
      direction: "ASC"
    }
    filter_groups: {
      filters: {
        field: "block_id",
        value: "1",
        condition_type: "eq"
      }
    }
  }){
    items{id, title, content}
  }
```

### Requesting Custom Attributes

Custom attributes is an array of Attributes in the data interfaces corresponding to EAV entities which represent values of EAV attributes. You can find examples of custom attributes on the ProductInetrafce, CustomerInterface, etc

Custom attributes are not filterable and represented as an array of all attributes available for the entity. In order to make retrieval of custom attributes more efficient, filtration syntax is added.

Now, in order to request specific attribute under EAV entity interface, specify filter request: `filter:["attribute_code"]`
 
Example:
```
MagentoCatalogProductSearchResults(searchCriteria:{
    filter_groups: {
       filters: {
        field: "sku",
        value: "MH12",
        condition_type: "in"
      }
    },
    page_size:10,
    current_page: 0
}){
  items {
    id, 
    name, 
    price, 
    custom_attributes(
        filter:["thumbnail"]
    ) {
        attribute_code, 
        value
    }
  }
}
``` 


## React Sample App

The examples directory [examples/react-product-webcomponent](https://github.com/vrann/m2graphql/tree/master/examples/react-product-webcomponent) contains sample React application which loads Magento Product from the GraphQL API and renders as a javascript widget

See demo here: [http://m2graphql.com/sampleapp/index.html](http://m2graphql.com/sampleapp/index.html)

### Run the Example

1. Clone this repository
2. `cd examples/react-product-webcomponent`
3. open index.html in browser

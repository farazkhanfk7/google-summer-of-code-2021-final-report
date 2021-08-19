
![Logo](https://miro.medium.com/max/1400/1*2qx3yIp2EsbfPsAHaEHJ5g.jpeg)

# Google Summer of Code 2021 Final Report

This summer, I was pleased to get selected for Google Summer of Code'21 under the [Hydra Ecosystem](https://www.hydraecosystem.org/) organization. Hydra Ecosystem aims to build a set of tools to automate the process of building REST APIs and Next-Gen smart clients which follow the principle of Semantic Web, Linked Data, and JSON-LD. Hydra is a vocabulary for APIs which machines can understand and it is currently a draft that is created and maintained by [Hydra-CG](https://www.hydra-cg.com/). This is a summary of my work done this summer in GSoC 2021. 

**Project:** [General Improvements in Hydrus](https://summerofcode.withgoogle.com/projects/#6336003247177728)

  
## Abstract

The project idea was about improvements in existing hydrus ( flagship server ). This included support for POST operation on `hydra:Collection`. New functionalities were added to effectively update, get or delete particular members from a Collection without actually sending the whole collection in the request payload. A lot of improvements were made in CRUD operations of hydrus, including setting up constraints and handling errors on the server-side depending upon how the data is stored. The project also aims to enhance the functionality of hydrus, optimize existing codes and keep it synced with upgrades and development in hydra-python-core and Hydra specifications. Some major changes were made in the test suite and even more tests were added for CRUD operations. New features were added to support different datatypes in hydrus and provide technical support for OpenRisk's Banking API project.


# Coding Period

> List of deliverables and major tasks completed

## Changed resource URI format in hydra-python-core
In the last week of Community Bonding Period I was asked to review a PR in the hydra-python-core library and realized that it needs some more changes. The issue was about changing the format of URIs. It took me some days to understand the issue and debug the core library to find the required changes.

I submitted the following PRs to solve the issue:
* https://github.com/HTTP-APIs/hydra-python-core/pull/81
* https://github.com/HTTP-APIs/hydra-python-core/pull/84
* https://github.com/HTTP-APIs/hydrus/pull/578




## Support POST operation on hydra:Collection
> Added support to get/update/delete members of a collection

Updation of hydra:Resource is allowed in hydrus (our flagship server ) and hydra:Collection is also a subclass of hydra:Resource, however making a POST request to a Collection wasn’t allowed. The reason behind this was that we were still trying to find an effective method to update/delete few hydra:members from a Collection without actually sending the whole collection in the request body which was definitely not the right way to go with as it will increase the server payload. The idea was about creating new endpoints in hydrus which will take both a collection id and it’s member id as parameters.


```
GET: /<api_name>/<collection_name>/<collection_id>/<member_id>
DELETE: /<collection_name>/<collection_id>/delete/<member_id>
```

This was done in the following : 
* Related Issue : https://github.com/HTTP-APIs/hydrus/issues/494
* PR : https://github.com/HTTP-APIs/hydrus/pull/554


## Deletion of multiple members from the Collection
> The next step was allowing the same for deletion of multiple members from the Collection. I created something similar for Collections as well. The endpoint I created would take name of the collection and a list of object ids as a parameter. 

```
/<api-name>/<resource_name>/<collection-id>/delete/<ids_list>
```
Here’s the issue and PR for the same :
* https://github.com/HTTP-APIs/hydrus/issues/579
* https://github.com/HTTP-APIs/hydrus/pull/580
* https://github.com/HTTP-APIs/hydrus/pull/591

## Change URI format for GET/POST/PUT/DELETE for hydra:Resource

After discussion with mentors, we decided that it would be better to pass the ids separated after a comma as a query parameter instead of using keywords like delete or add . Also, this would make the code for Resource much cleaner as we won’t have to create separate API views for deletion of single instances or multiple. I had to make this change for all existing routes and request methods, and this was the final result.

```
/<api-name>/<resource_name>/<collection-id>?instances=<ids_list>
```

Also modified existing endpoints for creation of multiple objects in a single request. 

**PUT** ( Response ):
```
{
 "@context":"https://www.w3.org/ns/hydra/core",
 "@type": "Status",
 "description": "Objects with ID ['id1','id2'] successfully added",
 "iri": ["id1","id2"],
 "statusCode": 201,
 "title": "Objects successfully added"
}
```

**DELETE**:

URI : `/serverapi/Movie?instances=id1,id2,id3`

Response:
```
{
 "@context":"https://www.w3.org/ns/hydra/core",
 "@type": "Status",
 "description":"Objects with ID ['id1','id2'] successfully deleted",
 "statusCode": 200,
 "title": "Objects successfully deleted"}
}
```
* PR : https://github.com/HTTP-APIs/hydrus/pull/591


##  Foreign-key relationship between a hydra:Class and hydra:Collection

>  This issue was about creating a foreign-key relationship between a hydra:Class and hydra:Collection

Previously, if an object was being deleted from a Class it would still be there in the collection table in database( if it was previously inserted in a Collection that directly manages that Class). In such a case, a GET response from Collection endpoint will show the member available in the database but will throw an InstanceNotFound error when a GET request will be made to view details of the object because it has already been deleted from the Class. To fix this, I used `manages` keyword to check if a `hydra:Collection` actually manages that Class.

Here’s the issue and PR for the same :
* Issue: https://github.com/HTTP-APIs/hydrus/issues/592
* PR : https://github.com/HTTP-APIs/hydrus/pull/593

## Support for different dataypes columns in database

> Previously, all database columns were set to String as there was no way of specifying the type of variable for a hydra:supportedProperty.
I discussed the implementation in a weekly meeting with mentors and came up with the final approach which was using range keyword to specify the type of a supported property. `range` keyword can be used to specify the datatype of a property according to Hydra specifications. We already have hydra:range (which is also inherited from rdfs:range) in our API documentation’s context.

```
"range": {
     "@id": "rdfs:range",
     "@type": "@id"
}
```

The idea was that a user can add range as an keyword argument in HydraClassProp while creating an apidoc like range="xsd:float" which will be expanded by doc_maker and then interpreted as https://www.w3.org/TR/xmlschema-2/#float. Now, hydrus will look into apidoc and then create attributes in database table according to their respective column types.

Related Issues and Pull Requests:
* https://github.com/HTTP-APIs/hydrus/issues/581
* https://github.com/HTTP-APIs/hydrus/pull/594
* https://github.com/HTTP-APIs/hydra-python-core/issues/88
* https://github.com/HTTP-APIs/hydra-python-core/pull/91

## Add support for datetime column in hydrus
>  At this point, only integer, float, string were supported datatypes in database.

The idea was that user should be able to specify datatype like "xsd:dateTime" while creating API doc. Database columns should be created accordingly. SQLAlchemy Datetime Column only takes a datetime object. But a user cannot send a datetime object in request body ( not json serializable ). This is why I created helper functions to convert datetime string to a python datetime object and then it'll be inserted to database. The helper function (get_modified_object) checks if there is any supported property in object (request body) which is actually a datetime field. And then returns a modified object accordingly.

Related Issues and Pull Requests:

* https://github.com/HTTP-APIs/hydrus/issues/598
* https://github.com/HTTP-APIs/hydrus/pull/599

## Changes in existing test cases in hydrus.
Added some more tests and made modifications in existing test cases. Regex was being used to get the id of the inserted object from response description. Instead of that I used `response.location` to get IDs of created objects throughout the tests.

PR : https://github.com/HTTP-APIs/hydrus/pull/601

## Dockerization for POC

The next task assigned to me was to containerize [creditrisk-poc](https://github.com/HTTP-APIs/creditrisk-poc) and use docker-compose to run services like hydrus server and Postgres database inside a container. I made some changes in existing repository and created Dockerfile and a docker-compose file for the same. Previously, we were using ConfigParser to get required environment variable from config.ini file, instead we used `os` module to env variables from docker-compose file and used `uwsgi.ini` to run the server using nginx. 

Pull Requests:
* https://github.com/HTTP-APIs/creditrisk-poc/pull/25
* https://github.com/HTTP-APIs/creditrisk-poc/pull/26

## Improvement in test coverage in hydra-python-core using pytest
Only three tests were present for doc_writer and only for testing the context. More tests are added in hydra_python_core for different components of doc_writer like HydraClass, HydraCollection, HydraEntryPoint and many others. Apart from this, we were previously using unittest’s mock to create mock classes and objects. Along with that, I used Pytest’s fixtures to test different components and improve the readability and increase test coverage.

PR : https://github.com/HTTP-APIs/hydra-python-core/pull/93

Apart from these deliverables that were completed during the Coding Period. I also worked on some bugs and issues that were found during this period.
I'll be adding a list of all Pull Requests made in the contribution section.



## 🚀 Contributions ( During Coding Period )

### Pull requests

| PR Link                                                                                                                                                              | Description                                              | Status    |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | --------- |
| [#554](https://github.com/HTTP-APIs/hydrus/pull/554)                                                                                    | Added new endpoint to Get/Delete member of a collection.             | Merged ✅ |
| [#578](https://github.com/HTTP-APIs/hydrus/pull/578)                                                                                    | Update docs and fixed fragments test.              | Merged ✅ |
| [#580](https://github.com/HTTP-APIs/hydrus/pull/580)                                                                                    | Added endpoint to delete multiple members from a Collection  | Merged ✅ |
| [#590](https://github.com/HTTP-APIs/hydrus/pull/590)                                                                                    | Changed endpoint format for GET,PUT, DELETE for multiple objects of Class and Collections       | Merged ✅ |
| [#591](https://github.com/HTTP-APIs/hydrus/pull/591)                                                                                    | Added functional tests for PUT/DELETE multiple class objects             | Merged ✅ |
| [#593](https://github.com/HTTP-APIs/hydrus/pull/593)                                                                                    | Added foreign key relationship between a Collection and managed Class                   | Merged ✅ |
| [#594](https://github.com/HTTP-APIs/hydrus/pull/594)                                                                                    | Added support for datatype for columns in database tablesn | Merged ✅ |
| [#597](https://github.com/HTTP-APIs/hydrus/pull/597)                                                                                    | workflow using GitHub Actions to publish release on PyPi   | Merged ✅ |
| [#599](https://github.com/HTTP-APIs/hydrus/pull/599)                                                                                    | Added support for datetime column in datebase        | Merged ✅ |
| [#601](https://github.com/HTTP-APIs/hydrus/pull/601)                                                                                    | Changes in tests in hydrus : removed regex             | Merged ✅ |
| [#603](https://github.com/HTTP-APIs/hydrus/pull/603)                                                                                    | Fix get_host_domain function for deployment             | Merged ✅ |
| [#81](https://github.com/HTTP-APIs/hydra-python-core/pull/81)                                                                                    | Changed resource URI format in hydra-python-core             | **Open** |
| [#84](https://github.com/HTTP-APIs/hydra-python-core/pull/84)                                                                                    | Modified doc_maker and updated sample docs             | Merged ✅ |
| [#91](https://github.com/HTTP-APIs/hydra-python-core/pull/91)                                                                                    | Added support for datatype(range) in supported properties             | Merged ✅ |
| [#93](https://github.com/HTTP-APIs/hydra-python-core/pull/93)                                                                                    | Added tests in hydra python core             | **Open** |
| [#25](https://github.com/HTTP-APIs/creditrisk-poc/pull/25)                                                                                    | Dockerize creditrisk_pocon             | Merged ✅ |
| [#26](https://github.com/HTTP-APIs/creditrisk-poc/pull/26)                                                                                    | remove apidoc path environment variable from docker compose             | Merged ✅ |


## Acknowledgements

I would like to thank my mentors for their guidance and suppport through the project and GSoC journey.
I thoroughly believe it had a positive impact on my style of writing codes and tests. I am highly enthusiastic to be an active contributor and member of this organization even after completion of my project and will try to give my very best to keep this community active and healthy.

- [**Lorenzo Moriondo**](https://github.com/Mec-iS)
- [**Phillipos Papadopoulas**](https://github.com/open-risk)
- [**Chris Andrew**](https://github.com/chrizandr)
- [**Priyanshu Nayan**](https://github.com/priyanshunayan)
- [**Samesh Lakhotia**](https://github.com/sameshl)


I would also like to thank my GSoC colleague [**Purvansh Singh**](https://github.com/Purvanshsingh) for collaboration and discussions during the program.

## 🔗 Contact

[![GitHub](https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white)](https://github.com/farazkhanfk7)
[![linkedin](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/farazkhanfk7/)
[![Medium](https://img.shields.io/badge/Medium-%23000000.svg?style=for-the-badge&logo=Medium&logoColor=white)](https://batcypher.medium.com/)
[![Gmail](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:farazkhan138@gmail.com)


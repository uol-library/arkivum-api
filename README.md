Arkivum v6 API documentation
============================

The API documentation is available in the [GitHub Pages site of this repository](https://uol-library.github.io/arkivum-api/) using [Swagger UI](https://github.com/swagger-api/swagger-ui) and is made possible using a [template repository created by Peter Evans](https://peterevans.dev/posts/how-to-host-swagger-docs-with-github-pages/)

The Ingest API endpoint
-----------------------

Ingests can be long lived operations and therefore they operate asynchronously with respect to the http request that initiates them. Therefore, the typical pattern of operation will be to call an ingest endpoint and then monitor it for completion using the ingest status api.

All ingest initiated via the API will be visible on the UI with the 'Ingest Report' screen.

[Ingest API request workflow](assets/images/ingest-api-request.png)

Ingest data from a ingest location
----------------------------------

The ingested files go through a number of stages of processing to check their integrity, extract technical metadata and unpack any contained archives before they are placed into the archive locations. If a metadata file is defined it will determine the overall structure of the collections, objects and files and once the ingest is complete the structure will be visible via the UI.

### Request

`POST /api/arksys/ingest`

**Query Parameters**

<dl>
<dt>ingestPath</dt>
<dd><code>[String]</code> (required) The path within the ingest location to files (or file) to be ingested, should be URL encoded</dd>
<dt>metadataPath</dt>
<dd><code>[String]</code> The path within the ingest location to a metadata file, should be URL encoded</dd>
<dt>datapool</dt>
<dd><code>[String]</code> The name of the datapool into which files are to be ingested, should be URL encoded. Defaults to the default datapool</dd>
<dt>folderPath</dt>
<dd><code>[String]</code> The folder path used to prefix files paths being ingested, should be URL encoded</dd>
<dt>jobTag</dt>
<dd><code>[String]</code> A tag to be used to link related ingests, should be URL encoded</dd>
<dt>unpack</dt>
<dd><code>[Boolean]</code> Whether archives detected in the ingest location are unpacked or ingested without unpacking. Defaults to false</dd>
<dt>isArchive</dt>
<dd><code>[Boolean]</code> Whether archives detected in the ingest location are unpacked before the ingest begins or ingested without unpacking. Defaults to false</dd>
<dt>splitterChildren</dt>
<dd><code>[Integer]</code> This only applies when the ingest is into a datapool which has preservation enabled. The maximum number of files under a collection or object which can be preserved in one job. Above the threshold files are allocated to separate preservation jobs. Defaults to 10,000</dd>
<dt>locationId</dt>
<dd><code>[String]</code> The identifier of an ingest location. Defaults to the first ingest location.</dd>
</dl>

The `locationId` is visible on the UI:

[locationId variable in dashboard](assets/images/location-variable-in-dashboard.png)

### Responses

#### Status 202 - Accepted

The Request has been accepted by the system, the ingest can be determined using the `ingestId` status API.

**Example response**

```json
{
  "ingestId": "60143b7ee5310a740459c599",
  "datapool": "Default",
  "ingestPath": "dataset1",
  "metadataPath": "dataset1/ark-manifest.json",
  "folderPath": "myfolder",
  "status": "IN_PROGRESS",
  "errorMessage": null,
  "collectionId": "myCollection"
}
```
#### Status 4xx - Client errors

Typical values will be 404 ‘Resource Not Found’ or 400 ‘Bad Request’, but any status code in the 400 → 499 range will have the following form:

<dl>
<dt>errorMessage</dt>
<dd><code>[String]</code> An error message given the reason for failure</dd>
<dt>errorDetails</dt>
<dd><code>[array[String]]</code> A list of specific error conditions which have contributed to the overall failure.</dd>
</dl>

**Example Response**

```json
{
  "errorMessage":"Something has gone wrong",
  "errorDetails": [
    "This thing went wrong",
    "This other thing also went wrong"
  ]
}
```

Track the status of an ingest
-----------------------------

### Request

`GET /api/arksys/ingest/{ingestId}`

**Query Parameters**

<dl>
<dt>ingestId</dt>
<dd><code>[String]</code> (required) The identifier of the ingest job</dd>
</dl>

**Example Request**

```bash
curl -X GET \
  -H "Accept: */*" \
  -H "Authorization: Bearer + ACCESS_TOKEN" \
  "https://host.example.com/api/arksys/ingest/{ingestId}"
```

**Responses**

Responses are in the same format as those for ingestion.

The Ingestion Process
---------------------

### Implicit Preservation

If the ingest is into a datapool which has preservation enabled, once the ingest is mostly complete an automatic process is triggered which might result in one or more preservation jobs. Each preservation job is tracked under it’s own identifier, and associated with the original ingest.

### Usage of `metadataPath`

The `metadataPath` allows the user to specify a metadata file which performs two main functions: 

1. Create a [PCDM structure](https://pcdm.org/2016/04/18/models) for the files being ingested. This allows the ingest to be structured in a hierarchy in terms of collection, objects and files, where files are the files within the `ingestPath`.
2. Define the metadata associated with each collection, object or file. Metadata has to adhere to formats preconfigured in the system against the datapool. These can be pre-defined formats e.g. Dublin Core, ISAD(G), or created by the user.

There are two types of metadata file supported:

1. `ark-file-meta.csv` which is an Arkivum proprietary CSV format
2. `ark-manifest.json` which is an Arkivum proprietary JSON format

The PCDM Collections, Objects and Files can be viewed using the File Explorer. The ‘tree view’ icon can be used to show the hierarchical structure of the dataset.

[File Explorer interface](assets/images/file-explorer.png)

The ellipsis next to a Collection, Object or File can be used to pull up a menu that allows actions such as viewing metadata.

[File Explorer interface expanded](assets/images/file-explorer-action.png)

### Location of the metadata file

Whilst it is typical to have the metadata file located next to the files within the ingest and hence be contained within the `ingestPath`, this is not required - it is possible to locate the metadata file anywhere within the ingest location. This can be particularly useful for archives which have come from some other system already packed (see `isArchive` below) or anywhere where it is desirable not to disturb or pollute the files in the ingest location with a proprietary Arkivum format.

### Identifiers, `folderPath` and Datapool path

Within a metadata file identifiers are used to relate the collection, objects and files and for `fieldBindings` to extract metadata from other files.

* for files the identifiers are defined by the relative location of the file to the `ingestPath`. Once ingested the file identifiers will be prefixed by the datapool path and `folderPath`. This is to allow files to be unique within the system and to group them into the datapool in which they reside.
* for collections and objects the identifiers are defined in the metadata file and are unchanged by the datapool path or `folderPath`.

For example, if the path to a dataset in the ingest location is:

`test_data/ARK/planets`

And within this path there are files and subfolders like this:

```
test_data/ARK/planets/
└── objects
└── milky_way
└── solar_system
└── planets
├── earth
├── jupiter
│ ├── data
│ │ └── planetary_data.xls
│ ├── docs
│ │ ├── 11_Jupiter_FC.pdf
│ │ ├── 62211main_Jupiter_Lithograph.pdf
│ │ ├── Jupiter.docx
│ │ ├── Jupiter.msg
```

Then, when an ingest is triggered with `ingestPath = test_data/ARK/planets/` into the default datapool which always has path `/`, the files ingested will have IDs that look like this:

```
/objects/milky_way/solar_system/planets/Jupiter/data/planetary_data.xls
/objects/milky_way/solar_system/planets/Jupiter/docs/11_Jupiter_FC.pdf
```

Then, when an ingest is triggered with `ingestPath = test_data/ARK/planets/` into a datapool with path `/dp1`, then the files ingested will have IDs that look like this:

```
/dp1/objects/milky_way/solar_system/planets/Jupiter/data/planetary_data.xls
/dp1/objects/milky_way/solar_system/planets/Jupiter/docs/11_Jupiter_FC.pdf
```

When an ingest is triggered with `ingestPath = test_data/ARK/planets/` into a datapool with path `/dp1` and `folderPath = folder1/folder2` then the files ingested will have IDs that look like this:

```
/dp1/folder1/folder2/objects/milky_way/solar_system/planets/Jupiter/data/planetary_data.xls
/dp1/folder1/folder2/objects/milky_way/solar_system/planets/Jupiter/docs/11_Jupiter_FC.pdf
```

If an attempt is made to ingest a dataset that overlaps with data already within the system then you have these options:

1. Delete the dataset from the system before ingesting it again.
2. Change the subfolder and filenames within the `ingestPath` so they don’t clash with previous ingests.
3. Use the `folderPath` parameter to change the path of the files within the system.

### Bagit Ingests

The [Bagit format](https://datatracker.ietf.org/doc/html/rfc8493) is a set of hierarchical file layout conventions designed to support storage and transfer of arbitrary digital content.

If the `ingestPath` is structured as a Bagit format the system will automatically detect it based on whether the `ingestPath` contains a `bagit.txt` file and a data directory. If so the system will assume that the ingest is a bagit bag. It will then look for a checksum manifest and will validate the contents of the data directory.

Bagit bags are used purely as a ‘transport wrapper’ for supplying checksums that can be used to validate the payload of a bag. None of the other features of bagit are supported, e.g. tag manifests or bag-level metadata. As such the following should be noted:

* Only the content of the payload directory will be ingested after a bag has been checksum validated.
* None of the top-level bagit files (`bagit.txt`, `bag-info.txt`, `manifest-sha256.txt` etc.) will be ingested.
* No metadata will be extracted/processed from `bag-info.txt` or any tag files. Tag files will not be ingested.
* The system supports the mandatory parts of the bagit 1.0 spec, e.g. SHA256 and SHA512 checksum manifests, in addition, Adler32 checksum manifests are also supported. The system does not support optional parts of the spec, e.g. MD5 or SHA1 manifests.
* Bagit bag ingest using uncompressed bags, i.e. a bag in folder form. Zipped bags are only supported by usage of the `isArchive` parameter.
* Bagit fetch files are not currently supported.
* When referencing a metadata file using the `metadataPath`, the metadata file should be put inside the payload (`data` directory) of the bag. **Do not** put the metadata file at the top level of the bag, i.e. next to the manifest or `bagit.txt` files, because this will invalidate the bag and the ingest will fail.
* Given the above, indentifiers of files are therefore relative to the data directory and not to the the `ingestPath`.

**Example Bagit Format:**

Suppose the contents of the blogs folder looks like this:

```
test_data/ARK/bagit_datasets/blogs
├── bag-info.txt
├── bagit.txt
├── data
│ ├── Blogs
│ │ ├── Arkivum_Press_Release_FINAL.docx
│ │ ├── Arkivum_J_Press_Release_v1.docx
│ │ └── eTMF_blog_for_website.docx
│ └── ark-file-meta.csv
├── manifest-sha256.txt
├── manifest-sha512.txt
├── tagmanifest-sha256.txt
└── tagmanifest-sha512.txt
```

To ingest the above the above following parameters would be used

```
ingestPath = test_data/ARK/bagit_datasets/blogs/
metadataPath = test_data/ARK/bagit_datasets/blogs/data/ark-file-meta.csv
```

If the files were ingested into a datapool with path `/dp1` and `folderPath = folder1/folder2` then the files will have IDs that look like this:

```
/dp1/folder1/folder2/Blogs/Arkivum_Press_Release_FINAL.docx
/dp1/folder1/folder2/Blogs/Arkivum_J_Press_Release_v1.docx
/dp1/folder1/folder2/Blogs/eTMF_blog_for_website.docx
```

**Bagit Validation**

Bagit validation can be checked using the Audit Trail. For example, use the filter option (small funnel icon) and filter for events that contain ‘BAGIT’.

[Bagit validation](assets/images/bagit-validation.png)

Please note that the following files in the root of the bagit bag are mandatory:

**Bag Declaration: `bagit.txt`**

The `bagit.txt` file MUST consist of exactly two lines in this order:

```
BagIt-Version: M.N
Tag-File-Character-Encoding: ENCODING
```

`M.N` identifies the BagIt major (`M`) and minor (`N`) version numbers. `ENCODING` identifies the character set encoding used by the remaining tag files. `ENCODING` should be `UTF-8`.

**Payload Manifest: `manifest-[algorithm].txt`**

A payload manifest file lists each payload file along with a corresponding checksum to permit data integrity checking. Supported checksum algorithms are MD5, SHA256, SHA512 and Adler32. This supported manifest file names are:

```
manifest-alder32.txt
manifest-md5.txt
manifest-sha256.txt
manifest-sha512.txt
```

Each line of a payload manifest file MUST be of the form:

```
Checksum filepath
```

One or more manifests can be provided. It is not mandatory to use multiple manifests.

### Usage of `isArchive`

When the files to be ingested have come from some other system as an archive which is already packaged, but there is no desire to represent the archive itself with the system then it is possible to request that the archive is unpacked as part of the ingest. The following points are noteworthy:

* Extraction of the archive file is done asynchronously as part of the ingest.
* If the archive is in Bagit format the system will extract the archive, validate the bag, and ingest the contents of the payload (`data/`) directory. Everything above the payload directory will be discarded. It is only the contents of the payload that will be ingested.
* After the contents of the archive file (zip, tar etc.) have been extracted, the archive file will be discarded and not ingested. Only the contents of the archive file are ingested, not the archive file itself. The archive file is considered as a container for data during transit and not something that should be retained.
* If the archive file contains a single folder at the root level of the archive, then the contents of this folder will be ingested. The folder itself will not be included in the filepaths of the files that are ingested.
* If a metadata file is supplied by using the `metadataPath` parameter, then the metadata file cannot be inside the archive file, since `metadataPath` is always a path within the ingest location.

### Usage of `unpack`

When a `ingestPath` contains archives and those archives also contain archives then the option exists to request the system to unpack them as part of the ingest. The unpacking is fully recursive so all archive formats encountered will be unpacked. The following points are noteworthy:

* If an archive is encountered which is corrupted or unsupported or has a password it will be ingested as is.
* The format of archives (e.g. `tar`, `zip`, `7z`) is detected by looking at the file contents and possibly the file extension, therefore detection is best effort.
* If the archive is in root of the `ingestPath`, it will be ingested it addition to also unpacking it. This is to ensure there is no data loss should the system incorrectly unpack due to different feature sets between what was used to create the archive and what is used to unpack it.

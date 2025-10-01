# Support for ZIP and other container formats

- Start Date: 2025-10-01
- Authors: Mirek Simek
- RFC PR: TODO [#95](https://github.com/inveniosoftware/rfcs/pull/95)
- State: DRAFT

## Summary

This RFC proposes to add support for ZIP files and other container file formats. It includes the ability to show the contents of these files via the API and UI as well as to extract the files contained within them.

## Motivation

ZIP files and other container formats (e.g., tar, rar) are commonly used to bundle multiple files together for easier distribution and storage. However, currently there is no support for extracting and previewing the contents of these files in the Invenio framework. Adding this functionality would enhance the user experience by allowing users to view and access the contents of ZIP files directly within the Invenio platform.

This is a feature that is commonly requested by our users.

### User stories

#### Border Stones Dataset

A user submitted a dataset of 1000s images of border stones in a single ZIP file. The file has a hierarchy of folders that are named after the regions where the stones were found. The user would like to be able to preview the images in the UI and also download individual images or folders of images.

#### Multidimensional Data

A user submitted a multidimensional dataset packed as NetCDF file. The user would like to list the parts of the file and preview some of them as plots/maps.

## User interface

### File listing

Ideally, we would like to show the contents of the ZIP file in a hierarchical tree structure. The user should be able to expand/collapse folders to see the files contained within them. Each file should have a link to download it directly.



## Detailed design

### REST API

The API will be extended to support the following operations:

- **List contents**: An endpoint to list the contents of a ZIP file or other container formats. This will return a hierarchical structure of files and directories contained within the archive.

- **Extract files**: An endpoint to extract specific files or directories from the archive. This will allow users to download individual files or groups of files without needing to download the entire archive.

| Endpoint                                              | Description                    |
| --- | --- |
| `<record>/files/<key>/content/entries` | List contents of the archive |
| `<record>/files/<key>/content/entries/<path>` | Extract specific files or directories |

#### LIST operation

The list operation is a GET request to the endpoint `/api/records/<pid_value>/files/<key>/content/entries`.  User must have `can_get_content_files` permission to be able to call the API. The API returns a JSON response with the following structure:

```json5
{
    "entries": [
        {
            "key": "folder1",
            "title": "Folder 1", // optional
            "type": "directory",
            // other metadata fields can be added here and should be ignored by clients
            // if not recognized
            "entries": [ 
                {
                    "key": "file1.txt",
                    "title": "My text file", // optional
                    "type": "file",
                    "size": 1234, // optional
                    "checksum": "md5:abcd1234" // optional
                    "mime_type": "text/plain", // optional
                    // other metadata fields can be added here and should be ignored by clients
                    // if not recognized
                },
                // ...
            ]
        },
        // ...
    ],
    "total": 42 // total number of entries (files and directories)
    "truncated": false // true if the listing was truncated (e.g., too many entries)
}
```

Note: we do not plan to support pagination for the listing operation as pagination over hierarchical structure is difficult to implement. The entire structure will be returned in a single response. The extractor
should return the total number of entries and a flag indicating if the listing was truncated.

#### Extract operation

The extract operation is a GET request to the endpoint `/api/records/<pid_value>/files/<key>/content/entries/<path>`. User must have `can_get_content_files` permission to be able to call the API. It returns the content of the specified file or a container file containing the specified directory and its contents.

### Internal API

Most of the backend logic is implemented inside the `invenio_records_resources` package. 

#### `invenio_records_resources.services.files.extractors`

A new module `invenio_records_resources.services.files.extractors` will be created to handle the extraction. 

#### `invenio_records_resources.services.files.extractors.base`

```python

from invenio_records_resources.files.api import FileRecord

class SendFileProtocol(Protocol):
    def send_file(self) -> None: ...

class FileExtractor:

    def can_process(self, file_record: FileRecord) -> bool:
        """Determine if this extractor can process a given file record."""

    def list(self, file_record: FileRecord) -> list[dict]:
        """Return a listing of the file."""

    def extract(self, file_record: FileRecord, path: str) -> SendFileProtocol:
        """Extract a specific file or directory from the file record."""
```

#### `invenio_records_resources.services.files.extractors.zip`

This module will implement the `FileExtractor` interface for ZIP files using the `zipfile` module from the Python standard library.

There are two sources of the ZIP file and each one needs to have a different processing strategy:

1. Locally stored files - these are files that are stored on the local filesystem where we can easily have a random access to any part of the file.

2. Remotely stored files - these are files that are stored on remote storage backends (e.g., S3, Google Cloud Storage) where we do not have a cheap random access to the file. Seeks are expensive as they require a new connection to the remote storage.

To address both cases, we will add a pre-procesing step. After file is uploaded, a dedicated file processor will be called. This file processor will check the ZIP file and create an index of the contents. This index will be stored as a separate JSON file inside the media_files bucket. The index will contain all the information needed for the `list` operation, so that we do not need to open the ZIP file for listing.

For remote storage backends, the index will also contain a clone of the file table of contents (TOC) of the ZIP file. When a file from the ZIP will be requested, we will use the TOC to feed the `zipfile` module with the information about where the file is located inside the ZIP file. This will allow us to extract the file in a single operation without any seeks.

## Questions

- As support for extractors might require both FileExtractor and FileProcessor, would it make sense to have a contrib module that
would provide both? Similarly to what is in `invenio_vocabularies.contrib` for specific vocabularies. 
  - `invenio_records_resources.contrib.extractors.zip`
  - `invenio_records_resources.contrib.extractors.netcdf`

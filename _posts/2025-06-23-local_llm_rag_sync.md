---
layout: post
title:  "Local LLM RAG Sync"
author: robweber
categories: ai automation coding
tags: python ollama
---

Anyone with a home lab is playing around with local large language models via [Ollama][ollama], and I'm no exception. I have a pretty modest setup using Ollama and [Open WebUI][open-webui] to run some smaller parameter models. Once the novelty of asking it stupid questions, within the privacy of the fully local setup, ran out I started thinking "how can I get this to do something useful?". Naturally [Retrieval Augmented Generation (RAG)](https://en.wikipedia.org/wiki/Retrieval-augmented_generation) came to mind. Have the system put my own data to work for me so to speak.

Luckily, Open WebUI has good built in support for this through their [Knowledge](https://docs.openwebui.com/features/workspace/knowledge) system. The only speed bump I ran in to was how to keep the local knowledge base up to date. In this post I'll detail a Python script I wrote to leverage the Open WebUI API to sync files within the RAG Knowledge base system.

<!--more-->

{% include toc %}

## Background

For the sake of brevity I'm going to skip most of the details behind Ollama, Web UI and how RAG works in general. Setting up a working RAG system utilizing the Open WebUI Knowledge feature is fairly easy and a good primer on getting that working can be found in [this Medium article][rag-setup].

I actually got this all working pretty quickly, but the very next question was "what happens when the data changes?". Loading documents into the Knowledge system is a pretty manual process. There are built-in tools to sync with Google Drive or OneDrive but in my opinion those are pretty limited. I figured there had to be a better way to pipeline the data.

Doing some digging through the documentation I found that Open WebUI has a robust [REST API](https://docs.openwebui.com/getting-started/api-endpoints). It includes the ability to upload files and add/remove files from the Knowledge base.

### Requirements

As with most projects before getting started I jotted down a brief list of requirements to keep myself on track.

1. Create a script I could run on demand or on a schedule (likely via cron)
2. Utilize the Open WebUI [REST API](https://docs.openwebui.com/getting-started/api-endpoints)
3. Synchronize files from a directory with the Knowledge system
4. Synchronize includes adding new files, deleting old files, and updating existing files
5. If possible, the script should minimize data transfer. This means only updating files that have actually changed.

## First Steps

I started with the [examples][open-webui-examples] in the Open WebUI documentation regarding RAG. These were a good start on how to get files uploaded. To meet all my requirements I needed to also look at existing files, and make sure they were part of the right Knowledge area.

### Enable Swagger

The Open WebUI system has [Swagger](https://swagger.io/) built in, however it's disabled by default unless you're in development mode. I run my system in Docker so a quick change to the `docker-compose.yml` enabled the Swagger documentation. If you're running Docker you need to update the ENV variable and set it to __dev__.

```
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    environment:
     - ENV=dev
  volumes:
     - ./data:/app/backend/data
  restart: always
```

Once enabled you can access the Swagger documentation using the URL for your Open WebUI instance `/docs`.

### API Endpoints

Reviewing the API I was able to zero in on a few endpoints that would do what I needed.

* `/knowledge/list` - list all Knowledge base groups
* `/knowledge/{id}` - get information on a specific Knowledge group
* `/knowledge/{id}/files` - various utilities for adding/removing a file from a Knowledge group
* `/knowledge/{id}/reindex` - force Open WebUI to re-index the Knowledge base on the file set
* `/files/` - methods for adding/removing/viewing actual files

### File Operations

Utilizing these endpoints I wrote a Python script to compare a directory of local files with those in a specific Knowledge base. From there it was a simple matter of doing some comparisons to figure out if a file needed to be added, removed, or updated. All of this can be done very quickly with the [filter()](https://docs.python.org/3/library/functions.html#filter) function and [list comprehensions](https://docs.python.org/3/tutorial/datastructures.html#list-comprehensions).

```
result = {}  # final dict containing lists of new, deleted, or existing files

# other functions retrieve file lists for both the local and remote files
local_files = get_local_files()  # list in the format ["file1", "file2"]
remote_files = get_openwebui_files() # list in the format [{"id": "", "meta": {"name": "file1", ...}]

# existing = exists in both remote and local
existing = list(filter(lambda f: f['meta']['name'] in local_files, remote_files))
result['existing'] = list(map(lambda f: {'filename': f['meta']['name'], 'id': f['id']}, existing))

# deleted = exists in remote but NOT in local
deleted = list(filter(lambda f: f['meta']['name'] not in local_files, remote_files))
result['deleted'] = list(map(lambda f: {'filename': f['meta']['name'], 'id': f['id']}, deleted))

# generate list of only remote filenames
remote_filenames = list(map(lambda f: f['meta']['name'], remote_files))
# new files exist in local but NOT in remote
result['new'] = list(filter(lambda f: f not in remote_filenames, local_files))

```

Breaking this code down a bit it starts with some functions to retrieve a list of `local_files` and `remote_files`. From these lists the three lists for new, deleted, or existing can be made. For existing and deleted the first lambda filters the files based on if they do or do not exist in the Open WebUI Knowledge base. The second lambda creates a list of these files in the format `{'filename': name, 'id': openweb_ui_id}`. This format is important since the ID is needed to reference the file via the API.

For the new files a different kind of comparison is done. A list of local filenames is generated where the local file does not exist in the remote file list. For new files I didn't need an ID component since the file doesn't have one yet.

## Comparing Hashes

Using the three lists generated above I just needed to run the API calls to either upload, remove, or re-upload the documents. Almost all my requirements were satisfied, with one exception. I had no way of knowing which of the existing files had actually changed. Without a comparison for this I would have to re-upload them all.

Browsing through the API data I noticed that each file was returning a __hash__ value as part of the file's meta data information. How this hash was being calculated was not explained in the API so I did some digging through the Open WebUI source code and [found this](https://github.com/open-webui/open-webui/blob/main/backend/open_webui/utils/misc.py#L279). The file hash is a SHA256 generated hash value. I quick whipped up my own version:

```
def _calculate_sha256_string(self, string):
      # Create a new SHA-256 hash object
      sha256_hash = hashlib.sha256()
      # Update the hash object with the bytes of the input string
      sha256_hash.update(string.encode("utf-8"))
      # Get the hexadecimal representation of the hash
      hashed_string = sha256_hash.hexdigest()

      return hashed_string
```

During the existing file processing code I just had to calculate the hash for the local file and compare to the one returned from the API. If they match, I can skip uploading the file.

## Final Product

If you're interested in the final product I did upload it [to a Gist][final-script]. While not an Earth shattering example of fantastic code, I think this underscores how the existence of a well defined API can make a big difference when working with any software platform. There are times where leveraging some automation can drastically improve the workflow of a process. Closed systems that don't allow this kind of innovation force you into a more manual pipeline, or you have to come up with workarounds that are prone to failure as well.

### Limitations

It is worth pointing out that while useful, the final script does have some limitations. The first of which is because it utilizes file names for comparisons trying  to synchronize more than one directory would be difficult. If you have any overlap in file names there would be issues. In my case this wasn't a problem but you'd have to lean more on file hashing to figure out the list of new, old, and existing files.

Another limitation is that I'm assuming these files are all text files that the LLM can read. Again, I knew that would be the case with my data but it doesn't make the script generic enough for other use cases. Microsoft has published a [pretty decent tool][markitdown] that transforms documents into Markdown for LLMs that could be inserted for transforming files during the upload process.

## Links

* [Sync WebUI Docs Script][final-script] - final script to sync a local directory with an Open WebUI Knowledge repository
* [Ollama][ollama] - local LLM service
* [Open WebUI][open-webui] - Ollama web front end
* [RAG Setup Example][rag-setup] - Medium article detailing the initial RAG setup in Open WebUI
* [Markitdown][markitdown] - Microsoft tool for converting documents to the Markdown format

[ollama]: https://ollama.com/
[open-webui]: https://www.openwebui.com/
[rag-setup]: https://medium.com/@hautel.alex2000/open-webui-tutorial-supercharging-your-local-ai-with-rag-and-custom-knowledge-bases-334d272c8c40
[open-webui-examples]: https://docs.openwebui.com/getting-started/api-endpoints/#-retrieval-augmented-generation-rag
[markitdown]: https://github.com/microsoft/markitdown
[final-script]: https://gist.github.com/robweber/894508850d7c632ad824b3d02104e868

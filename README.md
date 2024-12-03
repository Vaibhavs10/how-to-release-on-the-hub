# How to release on the Hub

This is a collections of snippets and scripts that come in handy for handling a release on the Hub. There is no order in these snippets, but they have come in clutch in pretty much all the releases.

## Move repos

```python
from huggingface_hub import create_collection, HfApi, move_repo
import os

api = HfApi(token=os.getenv("HF_TOKEN"))

models = api.list_models(model_name = "xx/llama")  # Replace with your query

for model in iter(models):
    move_repo(from_id=model.id, to_id=f"meta/{model.id.split("/")[-1]}") # Replace with the destination org
```

## Update repo visibility

```python
from huggingface_hub import update_repo_settings

models = ["some/repo"]

def do_publish(source):
    update_repo_settings(source, private=False)

for model_id in models:
    do_publish(model_id)
```

## Enable gating programmatically

Use [`update_repo_settings`](https://huggingface.co/docs/huggingface_hub/en/package_reference/hf_api#huggingface_hub.HfApi.update_repo_settings). (Sorry, I can't find my script now, but should be similar to the one about repo visibility above).

## Update metadata

```python
from huggingface_hub import metadata_update

for repo_name in ["lol/yolo"]:
    metadata_update(repo_name, 
                    {"library_name": "transformers"}, 
                    overwrite=True,
                    create_pr=True)
```

Interesting metadata fields we frequently need:
- For gated repos: `extra_gated_heading`, `extra_gated_prompt`, `extra_gated_button_content`
- `license`
- `library_name`, `pipeline_tag`, `tags`

## Upload stuff to a repo

Run this from the folder you want to upload from. You can choose individual files, or use the `.` wildcard to upload everything in the current folder.

```bash
huggingface-cli upload --private --create-pr meta-llama/Llama-Guard-3-11B-Vision .
```

## Download stuff locally

```bash
huggingface-cli download meta-llama/Llama-3.2-3B-Instruct --local-dir Llama-3.2-3B-Instruct
```

P.S. for large repos - make sure to setup `hf_transfer` -> `pip install hf_transfer`

## Obtain IDs for gating

TL;DR: `id=$(curl -s -H "Authorization: Bearer $TOKEN" "https://huggingface.co/api/models/repo/model" | jq ._id)`

Example script for several models:

```bash
#!/bin/bash

TOKEN=$(cat ~/.cache/huggingface/token)

for x in 7b 13b 34b 70b
do
	for m in CodeLlama-${x}-hf CodeLlama-${x}-Python-hf CodeLlama-${x}-Instruct-hf
	do
		id=$(curl -s -H "Authorization: Bearer $TOKEN" "https://huggingface.co/api/models/meta-llama/${m}" | jq ._id)
		echo "	new ObjectId($id), // \"meta-llama/${m}\""
	done
done
```

If you do not have `jq` here is the same code snippet but with `python` and the `json` packages
```bash
#!/bin/bash

TOKEN=$(cat ~/.cache/huggingface/token)

for x in 7b 13b 34b 70b
do
	for m in CodeLlama-${x}-hf CodeLlama-${x}-Python-hf CodeLlama-${x}-Instruct-hf
	do
		id=$(curl -s -H "Authorization: Bearer $TOKEN" "https://huggingface.co/api/models/meta-llama/${m}" | python3 -c "import sys, json; print(json.load(sys.stdin)['_id'])")
		echo "	new ObjectId($id), // \"meta-llama/${m}\""
	done
done
```

## Squash all commits into one

```python
from huggingface_hub import HfApi

api = HfApi()

repo_id = "some/repo"

api.super_squash_history(repo_id=repo_id)
```

## Batch updating model cards

This is useful for releases with many checkpoints. The example below shows how to replace a link, but I've used the same process to programmatically create model cards from a template.

```python
from huggingface_hub import ModelCard

# Get your model list somehow
with open("models.txt") as f:
    models = [m.strip() for m in f]

# Open PRs in each repo for visibility
prs = {}
for model_id in models:
    print(model_id)
    card = ModelCard.load(model_id)
    new_link = "https://replace.me"
    card.text = card.text.replace("https://pre/release/link/to/be/replaced", new_link)
    print(f"-- Pushing to {model_id}")
    prs[model_id] = card.push_to_hub(model_id, create_pr=True, commit_message="Update card")


# If you run this in a notebook or persist the prs to a file,
# you can verify everything looks right and then merge

from huggingface_hub import merge_pull_request
for repo, commit in prs.items():
    print(repo, commit.pr_num)
    merge_pull_request(repo, commit.pr_num)
```

## Get SHA of LFS files

```python
model_id = "some/model"
files = api.model_info(model_id, files_metadata=True).siblings

repo_name = model_id.split("/")[-1]
to_block = []
for f in files:
    if f.rfilename.endswith(".pth") or f.rfilename.endswith(".safetensors"):
        d = {
            "oid": f.lfs.sha256,
            "reasonForBlocking": f"Early release {repo_name}: {f.rfilename}"
        }
        to_block.append(d)
print(json.dumps(to_block, indent=4))
```

## Create a Collection

```python
from huggingface_hub import create_collection, add_collection_item

org = "someorg"
collection = create_collection(namespace=org, title="Some Collection")

for model in model_ids:
    # Need fully qualified ids
    model = f"{org}/{model}"
    add_collection_item(item_id=model, item_type="model", collection_slug=collection.slug)    
```

## Verify Tokenizer

In almost all cases, it is a good idea to verify both the slow and fast tokenizer after a model has been converted via official scripts.

Refer to this notebook to test the same: https://github.com/Vaibhavs10/scratchpad/blob/main/tokenizer_check_minimal_example.ipynb

## GGUF

It is almost inevitible to quantise the model into the infamous GGUF format and share the checkpoints to hub. The easiest way to go about it is:
1. Use the [GGUF My Repo](https://huggingface.co/spaces/ggml-org/gguf-my-repo) to quantise the model checkpoint and host it as a seperate and individual Hugging Face Repository.
2. Once you have the repository ready, test the model with `llama-cli` or `llama-server`
3. After you are happy with the results, move the repository into the respective organisation and make it public!

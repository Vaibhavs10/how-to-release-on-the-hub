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
from huggingface_hub import update_repo_visibility

models = ["some/repo"]

def do_publish(source):
    update_repo_visibility(source, private=False)

for model_id in models:
    do_publish(model_id)
```

## Update metadata

```python
from huggingface_hub import metadata_update

for repo_name in ["lol/yolo"]:
    metadata_update(repo_name, 
                    {"library_name": "transformers"}, 
                    overwrite=True,
                    create_pr=True)
```

## Upload stuff to a repo

Run this from the folder you want to upload from. You can choose individual files, or use the `.` wildcard to upload everything in the current folder.

```bash
huggingface-cli upload --create-pr meta-llama/Llama-Guard-3-11B-Vision .
```

## Download stuff locally

```bash
huggingface-cli download meta-llama/Llama-3.2-3B-Instruct --local-dir Llama-3.2-3B-Instruct --local-dir-use-symlinks False
```

P.S. for large repos - make sure to setup `hf_transfer` -> `pip install hf_transfer`

## Squash all commits into one

```python
from huggingface_hub import HfApi

api = HfApi()

repo_id = "some/repo"

api.super_squash_history(repo_id=repo_id)
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

## Verify Tokenizer

In a almost all cases it is a good idea to verify both the slow as well as fast tokenizer after a model has been converted via official scripts.

Refer to this notebook to test the same: https://github.com/Vaibhavs10/scratchpad/blob/main/tokenizer_check_minimal_example.ipynb

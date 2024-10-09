# How to release on the Hub

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
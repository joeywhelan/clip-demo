# Jina ClipV2 on EIS
## Contents
1.  [Summary](#summary)
2.  [Architecture](#architecture)
3.  [Features](#features)
4.  [Prerequisites](#prerequisites)
5.  [Installation](#installation)
6.  [Usage](#usage)

## Summary <a name="summary"></a>
This is a demonstration of usage of the Jina ClipV2 multi-modal embedder on Elastic Inference Service.

## Architecture <a name="architecture"></a>
![architecture](assets/arch.jpg) 


## Features <a name="features"></a>
- Jupyter notebook
- Builds an Elastic Serverless deployment via Terraform
- Creates a synthetic data set representing a product catalog for a hardware store.  The catalog includes product details such as description, a photo, and location in store.
- Loads this catalog into an Elastic index.  Photos are embedded via Jina ClipV2.
- Performs an image search and a text search against the embeddings, demonstrating the multimodal capabilities of Jina ClipV2.
- Deletes the entire deployment via Terraform

## Prerequisites <a name="prerequisites"></a>
- terraform
- Elastic Cloud account and API key
- Python

## Installation <a name="installation"></a>
- Edit the terraform.tfvars.sample and rename to terraform.tfvars
- Create a Python virtual environment

## Usage <a name="usage"></a>
- Execute notebook
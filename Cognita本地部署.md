# Deploy cognita locally

> [!tip]
> 关于Cognita必须使用`TFY_KEY`的问题已经在该issue中解决：https://github.com/truefoundry/cognita/issues/382
## Description

[Cognita](https://github.com/truefoundry/cognita)是一个API驱动的RAG开源项目，包含用户友好的前端页面和API驱动的后端，用于LLM的能力增强，并且准备了生产就绪环境。本文目标为在本地部署cognita，并配合拥有的多个大模型来源使用。

- LLM provider： Azure OpenAi
- Embedder： [mixedbread](https://www.mixedbread.ai/) mixedbread-ai/mxbai-rerank-large-v1
- Reranker: mixedbread-ai/mxbai-embed-large-v1



## Install & Config

Clone the repo

```shell
git clone git@github.com:truefoundry/cognita.git
```



Config the `model_config.yaml` and add the following in the tail and **comment** other services:

```yaml
############################ Mixedbread ###########################################
#   Uncomment this provider if you want to use Mixedbread as a models provider    #
#   Remember to set `MIX_API_KEY` in container environment (./compose.env)        #
##################################################################################### 
  - provider_name: Mixedbread
    api_format: openai
    base_url: https://api.mixedbread.ai
    api_key_env_var: MIX_KEY
    llm_model_ids: []
    embedding_model_ids:
      - 'mixedbread-ai/mxbai-embed-large-v1'
    reranking_model_ids: 
      - "mixedbread-ai/mxbai-rerank-large-v1"
    default_headers: {}
############################ Azure ###########################################
#   Uncomment this provider if you want to use Azure as a models provider    #
#   Remember to set `AZURE_API_KEY` in container environment                 #
###############################################################################  
  - provider_name: Azure
    api_format: openai
    base_url: # Put Azure Endpoint here
    api_key_env_var: AZURE_KEY
    llm_model_ids: 
      - 'gpt35-16k'
    embedding_model_ids: []
    reranking_model_ids: []
    default_headers: {}
```

Comment all content in `models_config.truefoundary.yaml` cause we will not use `local-infinity` to procvide LLM services

```yaml
# model_providers:
#   - provider_name: truefoundry
#     api_format: openai
#     base_url: https://llm-gateway.truefoundry.com/api/inference/openai
#     api_key_env_var: TFY_API_KEY
#     llm_model_ids:
#       - "openai-main/gpt-4o-mini"
#       - "openai-main/gpt-4-turbo"
#       - "azure-openai/gpt-4"
#       - "together-ai/llama-3-70b-chat-hf"
#     embedding_model_ids:
#       - "openai-main/text-embedding-ada-002"
#     reranking_model_ids: []
#     default_headers:
#       "X-TFY-METADATA": '{"tfy_log_request": "true", "Custom-Metadata": "Cognita-LLM-Request"}'

#   - provider_name: local-infinity
#     api_format: openai
#     base_url: http://cas-infinity.cognita-internal.svc.cluster.local:8000
#     api_key_env_var: INFINITY_API_KEY
#     llm_model_ids: []
#     embedding_model_ids:
#       - "mixedbread-ai/mxbai-embed-large-v1"
#     reranking_model_ids:
#       - "mixedbread-ai/mxbai-rerank-xsmall-v1"
#     default_headers: {}

#   - provider_name: faster-whisper
#     api_format: openai
#     base_url: http://cas-whisper.cognita-internal.svc.cluster.local:8000
#     api_key_env_var: ""
#     llm_model_ids: []
#     embedding_model_ids: []
#     reranking_model_ids: []
#     audio_model_ids:
#       - "Systran/faster-distil-whisper-large-v3"
#     default_headers: {}
```

Get `TFY_HOST` and `TFY_KEY` from truefoundary refering to https://github.com/truefoundry/cognita/issues/382#issuecomment-2421964755

Then input `API_KEY` to `compose.env`

```yaml
#...
## TFY VARS here
TFY_API_KEY=
TFY_HOST=

## BRAVE
BRAVE_API_KEY=

## WHISPER
WHISPER_PORT=10300
WHISPER_MODEL=Systran/faster-distil-whisper-large-v3

## mixedbread key here
MIX_KEY=
## Azure key here
AZURE_KEY=619275d53f8e4d81b01a24acc907bb20
```

Open `docker-compose.yaml` and add the following in `cognita-backend.environment`

```yaml
- MIX_KEY=${MIX_KEY}
- AZURE_KEY=${AZURE_KEY}
```

## Run

You can start the whole service by:

```shell
docker compose --env-file compose.env up -d 
```

If anything goes smoothly, you can visit the frontend by broswer in `IP:5001` and access the API endpoint via `IP:8000`. You can simply call a health check using:

```shell
curl -X 'GET' '127.0.0.1:8000/health-check' -H 'accept: application/json'
# {"status":"OK"}
```

> [!tip]
>
> 注意API文档可以在这里看到：https://cognita.truefoundry.com/api/
>
> 在API文档中的endpoint为`https://cognita.truefoundry.com/api/`，因此在本地使用中用`127.0.0.1`代替

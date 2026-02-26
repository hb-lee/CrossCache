# CrossCache

### About CrossCache

This project references LMCache and implements a flexible KV Cache management system to accelerate TTFT in large model inference. The project decouples tensor semantics in AI from memory semantics in storage, and implements KV Cache as a service on NPU. Based on this project, users can focus only on storage-level management, such as building multi-level shared caches, without needing to pay attention to the handling of tensor semantics in AI.

NOTE: This is a WIP project, and the end-to-end process has already been run on the NPU. Everyone is welcome to contribute.

### Dependency Matrix

- vllm: v0.11.2
- vllm-ascend: 2b82320b
- torch_npu: 2.7.1
- CANN: 8.5.0
- HDK: 25.3.RC1

### How to build

##### Build the CrossCache server

```
git clone ${repo} --recurse-submodules
cd build;
```

- prepare dependency

```
bash build.sh prepare
```

- Prepare env

```
You should edit your env.local to configure your variable in correct way.
```

- build project

```
bash build.sh build
```

##### Build the client library

> Need NPU environment

```
cd python;
python3 setup.py build_ext --inplace
```

##### Run the crosscache

```
Now, we provide one demo call fake_client.py.

- run server
bin/crosscache -c config.cfg

> the config.cfg is the configuration file for crosscache. And the content could be like:

``` cfg
{
	"Basic": {
		"Port": 5555,
		"ChunkSize": 256,
		"PoolSize": 4,
		"CacheDir": "/var/log/crosscache"
	},
	"Request": {
		"LoadWorker": 4,
		"StoreWorker": 4,
		"CommWorker": 2
	},
	"Cache": {
		"IOWorker": 4,
		"CopyWorker": 4,
		"LookupMapScale": 4096
	}
}
```

- run the client to test
python3 fake_client.py
```

### How to run with vllm

Change as following:

```python
# add to vllm/distributed/kv_transfer/kv_connector/factory.py
KVConnectorFactory.register_connector(
	"CrossCacheConnector",
	"vllm.distributed.kv_transfer.kv_connector.v1.crosscache.crosscache_connector",
	"CrossCacheConnector",
)
```

```sh
mkdir ${VLLM_SRC}/vllm/distributed/kv_transfer/kv_connector/v1/crosscache
cp -r python/utils ${VLLM_SRC}/vllm/distributed/kv_transfer/kv_connector/v1/crosscache
cp python/__init__.py ${VLLM_SRC}/vllm/distributed/kv_transfer/kv_connector/v1/crosscache
cp python/ipckey_utils.c* ${VLLM_SRC}/vllm/distributed/kv_transfer/kv_connector/v1/crosscache
cp python/adaptor.py ${VLLM_SRC}/vllm/distributed/kv_transfer/kv_connector/v1/crosscache
cp python/crosscache_connector.py ${VLLM_SRC}/vllm/distributed/kv_transfer/kv_connector/v1/crosscache
```

> ${VLLM_SRC} is the VLLM source directory which you built from.

##### run the vllm

```sh
export VLLM_LOGGING_LEVEL=DEBUG
vllm serve /workspace/models/qwen2.5_7B_Instruct --no-enable-prefix-caching --gpu-memory-utilization 0.8 --kv-transfer-config '{"kv_connector":"CrossCacheConnector", "kv_role":"kv_both", "kv_connector_extra_config": {"cache.server.host": "tcp://127.0.0.1", "cache.server.port": 5555}}'
vllm serve /workspace/models/qwen2.5_7B_Instruct --no-enable-prefix-caching --gpu-memory-utilization 0.8 --kv-transfer-config '{"kv_connector":"CrossCacheConnector", "kv_role":"kv_both", "kv_connector_extra_config": {"cache.server.host": "tcp://127.0.0.1", "cache.server.port": 5555}}' --host 0.0.0.0 --port 9000
```

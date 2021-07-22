mkfile_path := $(PWD)

start:
	docker run \
		-v $(mkfile_path)/vector.toml:/etc/vector/vector.toml:ro \
		-p 18383:8383 --network esvsch \
		registry.cn-shanghai.aliyuncs.com/public-namespace/vector:nightly-alpine

## Docker image history
```shell
docker image history <image>:<tag> --format "{{.Size}}\t{{.CreatedBy}}" --no-trunc
```
do


```
gitlab-runner register \
  --url "http://10.0.1.221:9080" \
  --registration-token "GR1348941RzXMiymP8h7f8qLady8s" \
  --executor "shell" \
  --description "project-runner-shell" \
  --tag-list "shell" \
  --run-untagged="true" \
  --locked="false"
```

```
docker exec -it gitlab-runner gitlab-runner register --non-interactive \
  --url "http://10.0.1.221:9080" \
  --registration-token "GR1348941RzXMiymP8h7f8qLady8s" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "project-runner-docker" \
  --tag-list "docker" \
  --run-untagged="true" \
  --locked="false"
```


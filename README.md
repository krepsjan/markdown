# Markdown for documentation

## Links and sources
- https://hub.docker.com/r/squidfunk/mkdocs-material
- https://docs.docker.com/engine/reference/builder/#run
- https://docs.gitlab.com/ee/ci/docker/using_docker_images.html#overriding-the-entrypoint-of-an-image
- https://phoenixnap.com/kb/docker-run-override-entrypoint
## Using MkDocs

This documentation is written in Markdown, using MkDocs system with A Material Design theme. 
If you want to preview your changes before commiting them, you can use docker containter squidfunk/mkdocs-material 
``` 
# start development server 
docker run --rm -it -p 8000:8000 -v ${PWD}:/docs squidfunk/mkdocs-material 

# build documentation 
docker run --rm -it -v ${PWD}:/docs squidfunk/mkdocs-material build 
``` 

*NOTE* 

Command above outputs the `/site` direcory owned by root. The reason is, that the files are created by the user that runs within the container. If you want your files to be created as another user, run the container as this other user. e.g. 
``` 
docker run --rm -it -v ${PWD}:/docs -u 1000:1000 squidfunk/mkdocs-material build 
```

## Gitlab CI

For example of .gitlab-ci.yml see `src` dir in this repo. 

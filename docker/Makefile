VERSION=1.0-10.2.1
PROJECT_ID=google_containers
PROJECT=gcr.io/${PROJECT_ID}

all: build

build:
	docker build --pull -t ${PROJECT}/kubernetes-kafka:${VERSION} .

push: build
	gcloud docker -- push ${PROJECT}/kubernetes-kafka:${VERSION}

.PHONY: all build push

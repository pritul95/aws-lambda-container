## AWS Lambda Containerized Flask Application

This is example on how to create flask app container to use w/ AWS Lambda.

## Contents
* [File Tree](#file%20tree)
* [Files Review](#files%20review)
    * [Dockerfile](#dockerfile)
    * [api.init.py](#api.init.py)
    * [.serverless-wsgi](#.serverless-wsgi)
    * [entry.sh](#entry.sh)
    * [requirements.txt](#requirements.txt)
    * [wsgi.py](#wsgi.py)
* [Final Step](#final%20step)
* [References](#references)

### File Tree
This example assumes that you have initialized your dir like below.
```
.
├── Dockerfile
├── api
│   └── __init__.py
├── .serverless-wsgi
├── entry.sh
├── requirements.txt
└── wsgi.py
```

### Files Review
In this section we'll go thru each files listed above.

#### Dockerfile
Docker file for building container

```Docker
# Define global args
ARG FUNCTION_DIR="/home/lambda/"
ARG RUNTIME_VERSION="3.9"
ARG DISTRO_VERSION="3.12"
ARG SERVERLESS_TAG="1.7.6"

# Stage 1 - bundle base image + runtime
# Grab a fresh copy of the image and install GCC
FROM python:${RUNTIME_VERSION}-alpine${DISTRO_VERSION} AS python-alpine

# Install GCC (Alpine uses musl but we compile and link dependencies with GCC)
RUN apk add --no-cache \
    libstdc++

# Stage 2 - build function and dependencies
FROM python-alpine AS build-image

# Install aws-lambda-cpp build dependencies
RUN apk add --no-cache \
    build-base \
    libtool \
    autoconf \
    automake \
    libexecinfo-dev \
    make \
    cmake \
    libcurl \
    git

# Include global args in this stage of the build
ARG FUNCTION_DIR
ARG RUNTIME_VERSION
ARG SERVERLESS_TAG

# Create function directory
RUN mkdir -p ${FUNCTION_DIR}

# Install Lambda Runtime Interface Client for Python
RUN python${RUNTIME_VERSION} -m pip install awslambdaric --target ${FUNCTION_DIR}

# Copy required files
COPY * ${FUNCTION_DIR}
COPY api ${FUNCTION_DIR}/api

# Install the function's dependencies
RUN python${RUNTIME_VERSION} -m pip install -r ${FUNCTION_DIR}/requirements.txt --target ${FUNCTION_DIR}

# Download serverless-wsgi and copy wsgi_handler.py & serverless_wsgi.py
RUN git config --global advice.detachedHead false
RUN git clone https://github.com/logandk/serverless-wsgi --branch ${SERVERLESS_TAG} ${FUNCTION_DIR}serverless-wsgi
RUN cp ${FUNCTION_DIR}serverless-wsgi/wsgi_handler.py ${FUNCTION_DIR}wsgi_handler.py && cp ${FUNCTION_DIR}serverless-wsgi/serverless_wsgi.py ${FUNCTION_DIR}serverless_wsgi.py

# Stage 3 - final runtime image
# Grab a fresh copy of the Python image
FROM python-alpine

# Include global arg in this stage of the build
ARG FUNCTION_DIR

# Set working directory to function root directory
WORKDIR ${FUNCTION_DIR}

# Copy in the built dependencies
COPY --from=build-image ${FUNCTION_DIR} ${FUNCTION_DIR}

# (Optional) Add Lambda Runtime Interface Emulator and use a script in the ENTRYPOINT for simpler local runs
ADD https://github.com/aws/aws-lambda-runtime-interface-emulator/releases/latest/download/aws-lambda-rie /usr/bin/aws-lambda-rie
RUN chmod 755 /usr/bin/aws-lambda-rie
COPY entry.sh /
RUN chmod +x entry.sh

ENTRYPOINT [ "./entry.sh" ]
CMD [ "wsgi_handler.handler" ]

```

#### api.init.py
 This file is initializing basic flask app with one route to `/` which return hello world.

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, World!"
```

#### .serverless-wsgi
This is required json config file for serverless-wsgi framework. Here I have value for `app` set to `wsgi.app`, which represents the `app` variable inside [wsgi.py](#wsgi.py) file.
```
{
    "app": "wsgi.app"
}
```

#### entry.sh
"It executes the Lambda Runtime Interface Client for Python. If the execution is local, the Runtime Interface Client is wrapped by the Lambda Runtime Interface Emulator."

```script
#!/bin/sh

if [ -z "${AWS_LAMBDA_RUNTIME_API}" ]; then
    exec /usr/bin/aws-lambda-rie /usr/local/bin/python -m awslambdaric $1
else
    exec /usr/local/bin/python -m awslambdaric $1
fi
```

#### requirements.txt
Requirements file for your python app.

```
werkzeug==1.0.1
flask==1.1.2
serverless-wsgi==1.7.6
```

#### wsgi.py
Friendly wrapper around flask app for serverless-wsgi.

```python
# -*- coding: utf-8 -*-
"""
wsgi.py: Wsgi init
"""

import api

app = api.app

```

#### Final Step
Once image is built, push to ECR and create new lambda with that image. Create new API Gw and integrate with newly containerized lambda. And that's all :)

```
>>> curl -X GET "https://pnjj776bx0.execute-api.us-east-1.amazonaws.com/dev"
>>> Hello, World!
```

#### References
* [AWS Blog](https://aws.amazon.com/blogs/aws/new-for-aws-lambda-container-image-support)
* [Serverless WSGI](https://github.com/logandk/serverless-wsgi)
* [AWS Lambda Emulator](https://github.com/aws/aws-lambda-runtime-interface-emulator/)
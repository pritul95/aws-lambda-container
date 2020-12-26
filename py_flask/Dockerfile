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

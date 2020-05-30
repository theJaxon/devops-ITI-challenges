FROM python:alpine

WORKDIR /usr/src/app

ENTRYPOINT [ "python3" ]

RUN apk add git && \
    git clone https://github.com/MohamedMSaeed/DevOps-Challenge-Demo-Code.git . && \
    pip install -r requirements.txt




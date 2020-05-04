FROM python:alpine

WORKDIR /usr/src/app

RUN mkdir //usr/src/app/templates /usr/src/app/tests /usr/src/app/static

COPY templates ./templates 
COPY tests ./tests
COPY static ./static

COPY hello.py requirements.txt ./

RUN pip install -r requirements.txt

ENTRYPOINT [ "python3" ]
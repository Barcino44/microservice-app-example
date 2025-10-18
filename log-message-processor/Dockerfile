FROM python:3.6-alpine

WORKDIR /app

RUN apk add --no-cache build-base

COPY . .

RUN pip3 install -r requirements.txt

CMD ["python3", "-u", "main.py"]
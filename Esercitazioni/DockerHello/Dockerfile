# syntax=docker/dockerfile:1.4
FROM python:3.10-slim

USER root

WORKDIR /app

RUN pip install flask

COPY . /app

EXPOSE 8000

ENTRYPOINT ["python3"]

CMD ["app.py"]
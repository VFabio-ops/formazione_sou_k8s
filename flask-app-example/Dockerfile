# syntax=docker/dockerfile:1.4
FROM python:3.10-slim

USER root

WORKDIR /app

COPY . /app

RUN --mount=type=cache,target=/root/.cache/pip pip install flask

EXPOSE 8000

ENTRYPOINT ["python3"]

CMD ["app.py"]
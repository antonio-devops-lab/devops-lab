FROM python:3.12-alpine

ARG APP_VERSION=dev
ENV APP_VERSION=$APP_VERSION

RUN adduser -D -u 10001 appuser
USER appuser

WORKDIR /app
COPY app.py .

EXPOSE 8080
CMD ["python", "app.py"]

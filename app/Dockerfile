FROM python:3.12-alpine

RUN adduser -D -u 10001 appuser
USER appuser

WORKDIR /app
COPY app.py .

EXPOSE 8080
CMD ["python", "app.py"]

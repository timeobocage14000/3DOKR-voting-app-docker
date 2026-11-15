FROM python:3.14.10-alpine3.24 AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.14.10-alpine3.24

RUN addgroup -S -g 10001 app && adduser -S -u 10001 -G app app

WORKDIR /app
USER 10001:10001

COPY --from=builder /install /usr/local
COPY --chown=10001:10001 app.py .
COPY --chown=10001:10001 templates ./templates
COPY --chown=10001:10001 static ./static

EXPOSE 8080

CMD ["python", "app.py"]

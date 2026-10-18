FROM python:3.13-slim
WORKDIR /app
COPY ingest/service.py /app/service.py
COPY schemas/events.json /app/schemas/events.json
ENV SCHEMA_PATH=/app/schemas/events.json
USER 65534:65534
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/health',timeout=2)"
CMD ["python", "-u", "service.py"]

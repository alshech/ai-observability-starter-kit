FROM python:3.12-slim

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1

RUN useradd --create-home --uid 10001 appuser
WORKDIR /app/user_agent

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY --chown=appuser:appuser . .
USER appuser

EXPOSE 8088

CMD ["python", "main.py"]

FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

RUN groupadd --gid 1000 app && \
    useradd --uid 1000 --gid 1000 --create-home --shell /usr/sbin/nologin app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY --chown=1000:1000 app.py .

USER 1000:1000

CMD ["python", "app.py"]


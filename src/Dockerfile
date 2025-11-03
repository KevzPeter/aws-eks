FROM python:3.11-slim

# Prevent Python from writing .pyc files and buffer stdout/stderr
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# minimal build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends build-essential gcc && rm -rf /var/lib/apt/lists/*

# python deps
COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt gunicorn

COPY . /app

ENV FLASK_APP=app.py
ENV FLASK_ENV=production
ENV MONGO_HOST=localhost
ENV MONGO_PORT=27017

EXPOSE 5000

# running the app using gunicorn server
# app module is `app` and the Flask instance is `app` (app:app)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]

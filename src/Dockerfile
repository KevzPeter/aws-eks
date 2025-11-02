FROM python:3.11-slim

# Prevent Python from writing .pyc files and buffer stdout/stderr
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Install small set of system dependencies (kept minimal)
RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential gcc \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies (also install gunicorn to serve the app)
COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt gunicorn

# Copy application files into the container
COPY . /app

# Optional environment defaults (can be overridden at runtime)
ENV FLASK_APP=app.py
ENV FLASK_ENV=production
ENV MONGO_HOST=localhost
ENV MONGO_PORT=27017

EXPOSE 5000

# Run the Flask app with gunicorn for production-ish serving
# The app module is `app` and the Flask instance is `app` (app:app)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]

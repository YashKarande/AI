# Use the official Python image
FROM python:3.13-slim

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Install the project into /app
COPY . /app
WORKDIR /app

# Allow statements and log messages to immediately appear in the logs
ENV PYTHONUNBUFFERED=1

# Install dependencies
RUN uv sync

EXPOSE $PORT

# Run the FastMCP server
CMD ["uv", "run", "server.py"]

FROM ollama/ollama:latest

# Listen on all interfaces, port 8080
ENV OLLAMA_HOST 0.0.0.0:8080

# Store model weight files in /models
ENV OLLAMA_MODELS /models

# Reduce logging verbosity
ENV OLLAMA_DEBUG false

# Never unload model weights from the GPU
ENV OLLAMA_KEEP_ALIVE -1

# Store the model weights in the container image
ENV MODEL gemma3:270m
RUN ollama serve & sleep 5 && ollama pull $MODEL

# Start Ollama
ENTRYPOINT ["ollama", "serve"]

FROM python:3.13-slim

# Copy uv from the official image
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Install system dependencies
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy all files
COPY . .

# Install Python dependencies
RUN uv sync

# Expose port
EXPOSE 8080

# Run the application
CMD ["uv", "run", "uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8080"]

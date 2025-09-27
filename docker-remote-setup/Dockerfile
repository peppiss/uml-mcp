FROM python:3.11-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    default-jdk \
    graphviz \
    curl \
    wget \
    ca-certificates \
    gnupg \
    dos2unix \
    bash \
    file \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js and npm using NodeSource repository
RUN curl -fsSL https://deb.nodesource.com/setup_18.x | bash - \
    && apt-get install -y nodejs \
    && rm -rf /var/lib/apt/lists/*

# Verify Node.js and npm installation
RUN node --version && npm --version

# Install PlantUML
RUN curl -L -o /opt/plantuml.jar \
    https://github.com/plantuml/plantuml/releases/latest/download/plantuml.jar

# Install SuperGateway globally
RUN npm install -g supergateway

# Set working directory
WORKDIR /app

# Copy requirements first for better Docker layer caching
COPY ../requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Install additional dependencies for remote access
RUN pip install fastapi uvicorn

# Copy application code
COPY .. .

# Create necessary directories
RUN mkdir -p /app/output /app/logs

# Set environment variables
ENV UML_MCP_OUTPUT_DIR=/app/output
ENV PLANTUML_JAR_PATH=/opt/plantuml.jar
ENV PYTHONPATH=/app
ENV NODE_ENV=production

# Create a startup script
RUN cat > /app/start-remote-server.sh << 'EOF'
#!/bin/bash
set -e

echo "Starting UML-MCP Server with SuperGateway..."
echo "Output directory: $UML_MCP_OUTPUT_DIR"
echo "PlantUML JAR: $PLANTUML_JAR_PATH"

# Start SuperGateway to bridge stdio to StreamableHTTP
exec supergateway \
  --stdio "python /app/mcp_server.py" \
  --streamablehttp \
  --port 8000 \
  --host 0.0.0.0 \
  --cors \
  --verbose
EOF

# Health check script
RUN cat > /app/healthcheck.sh << 'EOF'
#!/bin/bash
curl -f http://localhost:8000/health || exit 1
EOF

# Ensure proper line endings and permissions for all shell scripts
RUN dos2unix /app/*.sh 2>/dev/null || true
RUN chmod +x /app/*.sh
RUN ls -la /app/*.sh
RUN file /app/*.sh
RUN which bash
RUN head -1 /app/start-remote-server.sh | od -c

# Expose ports
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD ["/app/healthcheck.sh"]

# Start the remote server
CMD ["/app/start-remote-server.sh"]
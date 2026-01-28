FROM ubuntu:latest
LABEL authors="i"

# Create non-root user
RUN useradd -m -u 1000 -s /bin/bash appuser
USER appuser

ENTRYPOINT ["top", "-b"]
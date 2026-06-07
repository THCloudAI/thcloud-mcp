FROM ghcr.io/astral-sh/uv:python3.12-alpine
RUN uv tool install meta-ads-mcp
ENV PATH="/root/.local/bin:$PATH"
ENTRYPOINT ["meta-ads-mcp"]

FROM caddy:2.10.2-alpine

# Create non-root user
RUN adduser -D -g 'caddy' caddy

# Switch to non-root user
USER caddy

# Copy Caddyfile into the container
COPY Caddyfile /etc/caddy/Caddyfile

# Expose port 8080
EXPOSE 8080

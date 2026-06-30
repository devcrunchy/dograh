# Bake the proxy config into the image so a redeploy always ships the current
# config. A bind-mounted file does NOT force `docker compose up` to recreate the
# container, so nginx kept stale config in memory across redeploys.
FROM nginx:1.27-alpine
COPY dograh.proxy.conf /etc/nginx/conf.d/default.conf

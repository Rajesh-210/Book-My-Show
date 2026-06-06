# ─────────────────────────────────────────────────────
# Stage 1 — Build  (uses your existing node:18 base)
# ─────────────────────────────────────────────────────
FROM node:18 AS builder

WORKDIR /app

# Copy dependency files
COPY package.json package-lock.json ./

# Fix stale lock file: force compatible PostCSS versions first
RUN npm install postcss@8.4.21 postcss-safe-parser@6.0.0 --legacy-peer-deps

# Install all remaining dependencies
RUN npm install --legacy-peer-deps

# Copy source code
COPY . .

# Prevent OpenSSL errors with Node 18 + old CRA
ENV NODE_OPTIONS=--openssl-legacy-provider

# Build static production files into /app/build
RUN npm run build

# ─────────────────────────────────────────────────────
# Stage 2 — Serve  (tiny nginx image, no Node needed)
# ─────────────────────────────────────────────────────
FROM nginx:stable-alpine

# Remove default nginx placeholder page
RUN rm -rf /usr/share/nginx/html/*

# Copy built React files from Stage 1
COPY --from=builder /app/build /usr/share/nginx/html

# nginx config — handles React Router (client-side routing)
RUN echo 'server { \
    listen 80; \
    root /usr/share/nginx/html; \
    index index.html; \
    location / { try_files $uri $uri/ /index.html; } \
}' > /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]

# ---------- Build stage ----------
FROM node:18 AS builder

WORKDIR /app

# Copy dependency manifests first for Docker layer caching
COPY package.json package-lock.json ./

# Install exact dependencies from package-lock.json
RUN npm ci --legacy-peer-deps

# Copy application source
COPY . .

# Build the React production application
RUN NODE_OPTIONS=--openssl-legacy-provider npm run build


# ---------- Production stage ----------
FROM nginx:alpine

# Copy React production build into NGINX
COPY --from=builder /app/build /usr/share/nginx/html

# NGINX listens on port 80
EXPOSE 80

# Keep NGINX running in the foreground
CMD ["nginx", "-g", "daemon off;"]
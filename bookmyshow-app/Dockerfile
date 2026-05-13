# Base image
FROM node:18

# Set working directory
WORKDIR /app

# Copy dependency files first (for caching)
COPY package.json package-lock.json ./

# Install dependencies cleanly
RUN npm install --legacy-peer-deps

# Copy full application code
COPY . .

# Fix OpenSSL issue (Webpack / React builds)
ENV NODE_OPTIONS=--openssl-legacy-provider

# App port
ENV PORT=3000

# Expose application port
EXPOSE 3000

# Start application
CMD ["npm", "start"]

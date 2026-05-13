# Use Node.js LTS image
FROM node:18

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --legacy-peer-deps

# Copy application source code
COPY . .

# React/Webpack OpenSSL fix
ENV NODE_OPTIONS=--openssl-legacy-provider

# Make app accessible outside container
ENV HOST=0.0.0.0
ENV PORT=3000

# Expose application port
EXPOSE 3000

# Start React application
CMD ["npm", "start"]

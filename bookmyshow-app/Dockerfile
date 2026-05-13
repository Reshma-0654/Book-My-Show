# Use Node.js LTS image
FROM node:18

# Create app directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --legacy-peer-deps

# Copy project files
COPY . .

# React/OpenSSL fix
ENV NODE_OPTIONS=--openssl-legacy-provider

# Make React accessible outside container
ENV HOST=0.0.0.0
ENV PORT=3000

# Expose port
EXPOSE 3000

# Start app
CMD ["npm", "start"]

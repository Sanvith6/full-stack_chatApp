# Use lightweight Node image
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy dependency files first (layer caching)
COPY package.json package-lock.json ./

# Install ONLY production dependencies
RUN npm ci --only=production

# Copy application source
COPY . .

# Set environment
ENV NODE_ENV=production

# Expose backend port
EXPOSE 5001

# Start application
CMD ["node", "src/index.js"]


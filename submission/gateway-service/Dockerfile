# Use official Node.js LTS base image
FROM node:18-alpine

# Set working directory inside the container
WORKDIR /app

# Copy package files first (for better layer caching)
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy the rest of the application source code
COPY . .

# Expose the port this service runs on
EXPOSE 3003

# Start the service
CMD ["node", "index.js"]

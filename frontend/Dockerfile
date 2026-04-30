# -------------------------
# Stage 1: Build React App
# -------------------------
FROM node:16-alpine AS build

# Set working directory
WORKDIR /app

# Copy package files first (better caching)
COPY package.json package-lock.json ./

# Install dependencies
RUN npm install

# Copy rest of the source code
COPY . .

# Build production bundle
RUN npm run build


# -------------------------
# Stage 2: Serve with Nginx
# -------------------------
FROM nginx:stable-alpine

# Remove default nginx static files
RUN rm -rf /usr/share/nginx/html/*

# Copy build output from previous stage
COPY --from=build /app/build /usr/share/nginx/html

# Expose port 80
EXPOSE 80

# Start nginx
CMD ["nginx", "-g", "daemon off;"]

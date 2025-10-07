# Stage 1: Build stage
FROM squidfunk/mkdocs-material as builder

# Install build dependencies
RUN apk add --no-cache --update nodejs npm

WORKDIR /build

# Copy dependency files first for better caching
COPY requirements.txt .
COPY package*.json ./

# Install dependencies
RUN pip --no-cache-dir install -r requirements.txt
RUN npm install

# Copy the rest of the project files
COPY . .

# Build the static site
RUN mkdocs build

# Stage 2: Runtime stage
FROM nginx:alpine as runtime

# Install runtime dependencies
RUN apk add --no-cache bash curl

# Copy the built static site from the builder stage
COPY --from=builder /build/site /usr/share/nginx/html

# Expose port for the web server
EXPOSE 80

# Use a simple startup script to enable customization at runtime
COPY dev/nginx.conf /etc/nginx/conf.d/default.conf

# Command to run the web server
CMD ["nginx", "-g", "daemon off;"]

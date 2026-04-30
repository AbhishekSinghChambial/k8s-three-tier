# -------------------------
# Stage 1: Install deps
# -------------------------
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./

# clean + reproducible install
RUN npm ci --only=production

# -------------------------
# Stage 2: Final image
# -------------------------
FROM node:18-alpine

WORKDIR /app

# copy only required things
COPY --from=builder /app/node_modules ./node_modules
COPY . .

EXPOSE 5000

CMD ["node", "index.js"]

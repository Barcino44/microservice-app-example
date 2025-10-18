FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

#Create a small image

FROM node:18-alpine

WORKDIR /app

COPY --from=builder /app .

EXPOSE 8082

CMD ["npm", "start"]
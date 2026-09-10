FROM node:22-alpine AS front-node

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build

#reverse proxy

FROM nginx:alpine

COPY --from=front-node /app/dist/frontend/browser/ /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]

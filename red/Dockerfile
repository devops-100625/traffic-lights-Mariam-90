FROM node:20-alpine

WORKDIR /app

COPY package.json ./
RUN npm install
COPY app.js ./
COPY index.pug favicon.ico ./views/
EXPOSE 80

CMD ["node","app.js"]
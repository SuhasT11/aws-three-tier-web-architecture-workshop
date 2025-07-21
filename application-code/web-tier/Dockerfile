FROM node:slim

WORKDIR /app
COPY package*.json ./
RUN npm install

COPY ./public ./public
COPY ./src ./src

EXPOSE 3000

ENTRYPOINT ["npm"]
CMD ["start"]
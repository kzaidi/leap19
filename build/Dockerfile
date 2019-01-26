FROM node:9-alpine

COPY . /src
RUN cd /src && npm install
EXPOSE 8080
USER node
CMD ["node", "/src/server.js"]

FROM node:26.13.0-alpine3.24 AS deps

WORKDIR /usr/src/app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:26.13.0-alpine3.24

ENV PORT=8080

WORKDIR /usr/src/app
USER 1000:1000

COPY --from=deps --chown=1000:1000 /usr/src/app/node_modules ./node_modules
COPY --chown=1000:1000 package.json server.js ./
COPY --chown=1000:1000 views ./views

EXPOSE 8080

CMD ["node", "server.js"]

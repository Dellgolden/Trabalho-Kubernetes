FROM node:20 as build

ARG REACT_APP_BACKEND_URL
ENV REACT_APP_BACKEND_URL=${REACT_APP_BACKEND_URL:-/api}

WORKDIR /app

# Copiar arquivos essenciais do projeto
COPY package.json package-lock.json ./

RUN npm install

# Instalar dependências e corrigir problemas com Babel e React
RUN yarn config set strict-ssl false && CYPRESS_INSTALL_BINARY=0 yarn install
# RUN yarn install
RUN yarn add @babel/plugin-proposal-private-property-in-object --dev
RUN yarn add @types/react --dev

# Copiar todo o projeto para o container
COPY . .

RUN npm run build

# Construir a aplicação
RUN yarn build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]


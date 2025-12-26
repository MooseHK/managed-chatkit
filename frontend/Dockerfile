FROM node:18-alpine AS build
WORKDIR /app

# Add build arguments for Vite
ARG VITE_CHATKIT_WORKFLOW_ID
ENV VITE_CHATKIT_WORKFLOW_ID=$VITE_CHATKIT_WORKFLOW_ID

# Install dependencies and build
COPY frontend/package.json frontend/package-lock.json* ./
RUN npm ci --legacy-peer-deps
COPY frontend/ ./
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY frontend/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

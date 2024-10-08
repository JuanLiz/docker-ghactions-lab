# Imagen base. Se pone el alias base para usarla en tareas aparte
FROM node:lts-alpine AS base

# Se toma la imagen base para compilar la aplicación
FROM base AS builder
RUN apk add --no-cache libc6-compat
WORKDIR /app
# Copiar los archivos de dependencias de la aplicación a la imagen
COPY package.json package-lock.json* ./
# Instalar las dependencias
RUN npm ci
COPY . .
# Compilar la aplicación con npm
RUN npm run build

# Se vuelva a tomar la imagen base, ahora para la ejecución de la aplicación
FROM base AS runner
WORKDIR /app
# Definir variables de entorno
ENV NODE_ENV production
# Añadir grupo y usuario al sistema
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Ajustar permisos de los archivos
RUN mkdir .next
RUN chown nextjs:nodejs .next
# Copiar los archivos de la imagen de compilación a la imagen final
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

# Definiendo el usuario que ejecutará la aplicación
USER nextjs
# Puerto que se va a exponer para acceder a la aplicación
EXPOSE 3000
ENV PORT 3000

# Comando usado para iniciar la aplicación cuando se ejecute el contenedor
CMD HOSTNAME="0.0.0.0" node server.js

FROM node:20-alpine AS base  

FROM base AS deps  
RUN apk add --no-cache libc6-compat  
WORKDIR /app  
COPY package*.json ./  
RUN npm ci  
RUN npm install sharp  

# Install PM2 here  
RUN npm install -g pm2  

FROM node:20-alpine AS builder  
WORKDIR /app  
COPY --from=deps /app/node_modules ./node_modules  
COPY . .  
RUN npm run build  

FROM base AS runner  
WORKDIR /app  
ENV NODE_ENV=production  
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs && \
    mkdir .next && \
    chown nextjs:nodejs .next  

COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./  
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static  
COPY --from=builder --chown=nextjs:nodejs /app/public ./public  

# Switch to nextjs user  
USER nextjs  
EXPOSE 3001  
ENV PORT 3001  

# Run the application with PM2, specifying server.js and instances  
CMD ["pm2-runtime", "start", "server.js", "-i", "0"]

FROM public.ecr.aws/docker/library/node:18-alpine
    
RUN apk update && apk add --update --no-cache \
	python3 && \
  rm /usr/lib/python3.12/EXTERNALLY-MANAGED && \
  python3 -m ensurepip

WORKDIR /app

COPY ./ ./

RUN pip3 install -r requirements.txt

RUN npm install \
    && npm run build

EXPOSE 3001

CMD [ "npm", "run", "startOnlyServer"]
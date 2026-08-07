FROM alpine:latest
RUN apk add --update --no-cache python3 && ln -sf python3 /usr/bin/python
WORKDIR /app
COPY . /app
EXPOSE 5000
ENTRYPOINT ["python3"]
CMD ["-m","http.server","5000" ]

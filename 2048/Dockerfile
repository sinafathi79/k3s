FROM alpine:3.24
RUN apk add --update --no-cache python3
WORKDIR /app
COPY . .
RUN adduser -D -u 1000 appuser && chown -R appuser /app
USER appuser
EXPOSE 5000
ENTRYPOINT ["python3"]
CMD ["-m","http.server","5000"]

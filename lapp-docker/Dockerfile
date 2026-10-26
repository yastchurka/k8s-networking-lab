FROM golang:alpine AS build
WORKDIR /src
COPY . .
RUN go build -o /go_server main.go

FROM alpine:latest
COPY --from=build /go_server /go_server
EXPOSE 8080
CMD ["/go_server"]
RUN echo "hello"

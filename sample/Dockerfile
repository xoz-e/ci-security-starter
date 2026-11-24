# The runtime image is scratch, so the image scan is the static binary
# and not a distro package set. The build stage does not ship.
FROM golang:1.27.4-alpine3.24 AS build

WORKDIR /src
COPY go.mod main.go ./
RUN CGO_ENABLED=0 go build -buildvcs=false -trimpath -ldflags='-s -w' -o /out/app .

FROM scratch
COPY --from=build /out/app /app
USER 65532:65532
EXPOSE 8080
ENTRYPOINT ["/app"]

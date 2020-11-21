FROM golang:1.15 as build

WORKDIR /go/src/github.com/othalla/plex_exporter

RUN git clone https://github.com/othalla/plex_exporter.git . \
	&& make build-linux

FROM alpine:3.12
COPY --from=build /go/src/github.com/othalla/plex_exporter/plex_exporter_unix /usr/bin/plex_exporter

ENTRYPOINT /usr/bin/plex_exporter

CMD [ "--help" ]

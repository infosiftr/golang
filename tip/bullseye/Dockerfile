#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#

FROM --platform=$BUILDPLATFORM golang:bullseye AS bootstrap
FROM buildpack-deps:bullseye-scm AS target
RUN dpkg --print-architecture > /arch
FROM --platform=$BUILDPLATFORM buildpack-deps:bullseye-scm AS build

ENV PATH /usr/local/go/bin:$PATH

COPY --from=bootstrap /usr/local/go /usr/local/goroot-bootstrap
COPY --from=target /arch /target-arch

# tip-20240810: https://github.com/golang/go/tree/d36353499f673c89a267a489beb80133a14a75f9
ARG GOLANG_COMMIT='d36353499f673c89a267a489beb80133a14a75f9'
ENV GOLANG_COMMIT $GOLANG_COMMIT

RUN set -eux; \
	now="$(date '+%s')"; \
	arch="$(dpkg --print-architecture)"; arch="${arch##*-}"; \
# TODO something cleaner here/above 😂😭
# TODO we need to *also* set GOHOST* based on the userspace of *this* container!! (ie, if our userspace is i386 but our actual underlying host is amd64 but we're building for arm64, then we need to set GOHOSTARCH=386 but GOARCH=arm64)
	arch="$(cat /target-arch)"; \
	case "$arch" in \
		'amd64') \
			export GOAMD64='v1' GOARCH='amd64' GOOS='linux'; \
			;; \
		'armel') \
			export GOARCH='arm' GOARM='5' GOOS='linux'; \
			;; \
		'armhf') \
			export GOARCH='arm' GOARM='7' GOOS='linux'; \
			;; \
		'arm64') \
			export GOARCH='arm64' GOARM64='v8.0' GOOS='linux'; \
			;; \
		'i386') \
			export GO386='softfloat' GOARCH='386' GOOS='linux'; \
			;; \
		'mips64el') \
			export GOARCH='mips64le' GOOS='linux'; \
			;; \
		'ppc64el') \
			export GOARCH='ppc64le' GOOS='linux'; \
			;; \
		'riscv64') \
			export GOARCH='riscv64' GOOS='linux' GORISCV64='rva20u64'; \
			;; \
		's390x') \
			export GOARCH='s390x' GOOS='linux'; \
			;; \
		*) echo >&2 "error: unsupported architecture '$arch' (likely packaging update needed)"; exit 1 ;; \
	esac; \
	\
# before we get too far, let's validate that our "bootstrap" Go works
	export GOROOT_BOOTSTRAP=/usr/local/goroot-bootstrap; \
# TODO figure out why QEMU's user-mode emulation with Go 1.23.0 causes "can't start telemetry child process: fork/exec /usr/local/go/bin/go: invalid argument" (it works if we run it a second time, but then later "go build" fails to fork/exec the compile process 🙃)
	"$GOROOT_BOOTSTRAP/bin/go" version; \
	\
	git init --quiet /usr/local/go; \
	git -C /usr/local/go fetch --depth 1 https://github.com/golang/go.git "$GOLANG_COMMIT:"; \
	git -C /usr/local/go checkout --quiet FETCH_HEAD; \
	\
# save the Git timestamp so we can use it for reproducibility
	SOURCE_DATE_EPOCH="$(git -C /usr/local/go log -1 --format='format:%ct' HEAD)"; \
	export SOURCE_DATE_EPOCH; \
	touchy="$(date -d "@$SOURCE_DATE_EPOCH" '+%Y%m%d%H%M.%S')"; \
# for logging validation/edification
	date --date "@$SOURCE_DATE_EPOCH" --rfc-2822; \
# sanity check (detected value should be older than our wall clock)
	[ "$SOURCE_DATE_EPOCH" -lt "$now" ]; \
	\
	( \
		export GOCACHE='/tmp/gocache'; \
		\
		cd /usr/local/go/src; \
		./make.bash; \
		\
# remove a few intermediate / bootstrapping files the official binary release tarballs do not contain (and ".git" that is hard to make reproducible)
		rm -rf \
			/usr/local/go/.git* \
			/usr/local/go/pkg/*/cmd \
			/usr/local/go/pkg/bootstrap \
			/usr/local/go/pkg/obj \
			/usr/local/go/pkg/tool/*/api \
			/usr/local/go/pkg/tool/*/go_bootstrap \
			/usr/local/go/src/cmd/dist/dist \
			"$GOCACHE" \
		; \
		\
# by default, our "host" binaries end up in /usr/local/go/bin and "target" binaries in /usr/local/go/bin/GOOS_GOARCH, so we have to move those up a level to match a native build (relevant for 32bit builds on 64bit, for example)
		if [ -d "/usr/local/go/bin/${GOOS}_${GOARCH}" ]; then \
			mv "/usr/local/go/bin/${GOOS}_${GOARCH}" /usr/local/go/bin.real; \
			rm -rf /usr/local/go/bin; \
			mv /usr/local/go/bin.real /usr/local/go/bin; \
		fi; \
		\
# clamp timestamps for reproducibility (allows "COPY --link" to be more clever/useful)
		touch -t "$touchy" /usr/local/.go-date-stamp; \
		find /usr/local/go -depth -newer /usr/local/.go-date-stamp -exec touch -ht "$touchy" '{}' +; \
		rm /usr/local/.go-date-stamp; \
	); \
	\
# ideally at this point, we would just "COPY --link ... /usr/local/go/ /usr/local/go/" but BuildKit insists on creating the parent directories (perhaps related to https://github.com/opencontainers/image-spec/pull/970), and does so with unreproducible timestamps, so we instead create a whole new "directory tree" that we can "COPY --link" to accomplish what we want
	mkdir /target /target/usr /target/usr/local; \
	mv -vT /usr/local/go /target/usr/local/go; \
	ln -svfT /target/usr/local/go /usr/local/go; \
	touch -t "$touchy" /target/usr/local /target/usr /target; \
	\
# smoke test
	go version; \
	go version | grep -qF "$GOOS/$GOARCH"; \
# make sure our reproducibile timestamp is probably still correct (best-effort inline reproducibility test)
	epoch="$(stat -c '%Y' /target/usr/local/go)"; \
	[ "$SOURCE_DATE_EPOCH" = "$epoch" ]; \
	find /target -newer /target/usr/local/go -exec sh -c 'ls -ld "$@" && exit "$#"' -- '{}' +

FROM buildpack-deps:bullseye-scm

# install cgo-related dependencies
RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		g++ \
		gcc \
		libc6-dev \
		make \
		pkg-config \
	; \
	rm -rf /var/lib/apt/lists/*

# don't auto-upgrade the gotoolchain
# https://github.com/docker-library/golang/issues/472
ENV GOTOOLCHAIN=local

ENV GOPATH /go
ENV PATH $GOPATH/bin:/usr/local/go/bin:$PATH
# (see notes above about "COPY --link")
COPY --from=build --link /target/ /
RUN mkdir -p "$GOPATH/src" "$GOPATH/bin" && chmod -R 1777 "$GOPATH"
WORKDIR $GOPATH

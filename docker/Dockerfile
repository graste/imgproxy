FROM alpine:edge AS BUILDIMG
LABEL maintainer="Sergey Alexandrovich <darthsim@gmail.com>"

SHELL ["/bin/ash", "-eo", "pipefail", "-c"]

ENV GOPATH /go
ENV PATH /usr/local/go/bin:$PATH

COPY . /go/src/github.com/DarthSim/imgproxy
WORKDIR /go/src/github.com/DarthSim/imgproxy

# Install dependencies
RUN echo "http://dl-cdn.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories \
  && apk --no-cache upgrade \
  && apk --no-cache add \
    curl \
    ca-certificates \
    go \
    gcc \
    g++ \
    glib-dev \
    make \
    musl-dev \
    expat-dev \
    fftw-dev \
    giflib-dev \
    lcms2-dev \
    libexif-dev \
    libheif-dev \
    libimagequant-dev \
    libjpeg-turbo-dev \
    libpng-dev \
    librsvg-dev \
    libwebp-dev \
  && rm -rf /var/cache/apk*

# Build ImageMagick
WORKDIR /root/ImageMagick
RUN curl -LfsS https://imagemagick.org/download/ImageMagick.tar.gz | tar -xz --strip-components 1 \
  && ./configure \
    --disable-deprecated \
    --disable-docs \
    --disable-openmp \
    --disable-static \
    --enable-silent-rules \
    --with-threads \
    --without-bzlib \
    --without-dps \
    --without-freetype \
    --without-heic \
    --without-jbig \
    --without-jpeg \
    --without-lcms \
    --without-lzma \
    --without-magick-plus-plus \
    --without-perl \
    --without-png \
    --without-tiff \
    --without-utilities \
    --without-webp \
    --without-wmf \
    --without-x \
    --without-xml \
  && make install-strip

# Build libvips (latest or specified version)
ARG VIPS_VERSION=latest
WORKDIR /root/vips-${VIPS_VERSION}
RUN if [ "${VIPS_VERSION}" = "latest" ]; then \
      VIPS_VERSION=$(curl -fsS 'https://api.github.com/repos/libvips/libvips/releases/latest' | sed -n 's/.*tag_name":\s"v\(.*\)".*/\1/p' | head -1); \
    fi \
    && echo "Vips version: $VIPS_VERSION" \
    && curl -LfsS https://github.com/libvips/libvips/releases/download/v$VIPS_VERSION/vips-$VIPS_VERSION.tar.gz | tar -xz --strip 1 \
    && ./configure \
      --disable-debug \
      --disable-dependency-tracking \
      --disable-magickload \
      --disable-static \
      --enable-debug=no \
      --enable-gtk-doc-html=no \
      --enable-gtk-doc=no \
      --enable-pyvips8=no \
      --enable-silent-rules \
      --without-OpenEXR \
      --without-gsf \
      --without-orc \
      --without-tiff \
    && make check \
    && make install-strip

# Build imgproxy
WORKDIR /go/src/github.com/DarthSim/imgproxy
RUN CGO_LDFLAGS_ALLOW="-s|-w" go build -v -o /usr/local/bin/imgproxy

# Copy compiled libs here to copy them to the final image
WORKDIR /root/libs
WORKDIR /root
RUN ldd /usr/local/bin/imgproxy | grep /usr/local/lib/ | awk '{print $3}' | xargs -I '{}' cp '{}' libs/

# ==================================================================================================
# Final image

FROM alpine:edge
LABEL maintainer="Sergey Alexandrovich <darthsim@gmail.com>"

RUN echo "http://dl-cdn.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories \
  && apk --no-cache upgrade \
  && apk --no-cache add \
    bash \
    ca-certificates \
    expat \
    fftw \
    giflib \
    glib \
    lcms2 \
    libexif \
    libgsf \
    libheif \
    libimagequant \
    libjpeg-turbo \
    libpng \
    librsvg \
    libwebp \
  && rm -rf /var/cache/apk*

COPY --from=BUILDIMG /usr/local/bin/imgproxy /usr/local/bin/
COPY --from=BUILDIMG /root/libs/* /usr/local/lib/

ENV VIPS_WARNING=0
ENV MALLOC_ARENA_MAX=4

CMD ["imgproxy"]

EXPOSE 8080

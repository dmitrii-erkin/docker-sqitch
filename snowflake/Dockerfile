FROM debian:stable-slim AS snow-build

WORKDIR /work

# Download the ODBC driver and SnowSQL.
# https://docs.snowflake.net/manuals/user-guide/snowsql-install-config.html#downloading-the-snowsql-installer
# https://docs.snowflake.net/manuals/release-notes/client-change-log-snowsql.html
# https://sfc-repo.snowflakecomputing.com/index.html
ADD https://sfc-repo.snowflakecomputing.com/snowsql/bootstrap/1.2/linux_x86_64/snowsql-1.2.25-linux_x86_64.bash snowsql.bash
ADD https://sfc-repo.snowflakecomputing.com/odbc/linux/2.25.10/snowflake_linux_x8664_odbc-2.25.10.tgz snowflake_linux_x8664_odbc.tgz
COPY conf ./

# Tell SnowSQL where to store its versions and config. Need to keep it inside
# the image so it doesn't try to load the version from $HOME, which will
# typically be mounted to point to the originating host.
ENV WORKSPACE /var/snowsql

# Set locale for Python triggers.
ENV LC_ALL=C.UTF-8 LANG=C.UTF-8

# Install prereqs.
ARG sf_account
RUN apt-get -qq update \
    && apt-get -qq --no-install-recommends install odbcinst \
    # Configure ODBC. https://docs.snowflake.net/manuals/user-guide/odbc-linux.html
    && gunzip -f *.tgz && tar xf *.tar  \
    && mkdir odbc \
    && mv snowflake_odbc/lib snowflake_odbc/ErrorMessages odbc/ \
    && mv simba.snowflake.ini odbc/lib/ \
    && perl -i -pe "s/SF_ACCOUNT/$sf_account/g" odbc.ini \
    && cat odbc.ini >> /etc/odbc.ini \
    && cat odbcinst.ini >> /etc/odbcinst.ini \
    # Unpack and upgrade snowsql, then overwrite its config file.
    # && SNOWSQL_DEST=. SNOWSQL_LOGIN_SHELL=/dev/null bash snowsql.bash \
    && sed -e '1,/^exit$/d' snowsql.bash | tar zxf - \
    && ./snowsql -Uv \
    && echo "[connections]\naccountname = $sf_account\n\n[options]\nnoup = true" > /var/snowsql/.snowsql/config

FROM sqitch/sqitch:latest

# Install runtime dependencies, remove unnecesary files, and create log dir.
USER root
RUN apt-get -qq update \
    && apt-get -qq --no-install-recommends install unixodbc \
    && apt-get clean \
    && rm -rf /var/cache/apt/* /var/lib/apt/lists/* \
    && rm -rf /man /usr/share/man /usr/share/doc \
    && mkdir -p /usr/lib/snowflake/odbc/log

# Install SnowSQL plus the ODBC driver and config.
COPY --from=snow-build /work/snowsql /bin/
COPY --from=snow-build --chown=sqitch:sqitch /var/snowsql /var/
COPY --from=snow-build /work/odbc /usr/lib/snowflake/odbc/
COPY --from=snow-build /etc/odbc* /etc/
RUN chmod --recursive a+rx /var/.snowsql

# The .snowsql directory is copied to /var.
USER sqitch
ENV SNOWSQL_DOWNLOAD_DIR /var/.snowsql
